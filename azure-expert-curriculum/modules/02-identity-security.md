# Module 02 — Identity, Access & Secrets

> **Week 2.** In the cloud, identity is the perimeter. This module turns the slogan into working .NET code: an app that authenticates to SQL, Storage, Service Bus and Key Vault **without a single secret in code, config, or pipeline**. Master this once and it repeats in every later module.

---

## 1. The cast of characters (get the nouns exactly right)

| Term | What it actually is |
|---|---|
| **Entra ID tenant** | Your org's identity directory. Issues tokens. One tenant ↔ many subscriptions. |
| **Security principal** | Anything that can be authenticated: user, group, **service principal**, **managed identity**. |
| **App registration** | The *definition* of an application in Entra (client ID, redirect URIs, exposed scopes/roles). Think: the class. |
| **Service principal** | The *instance* of an app registration in a tenant — the identity that actually gets role assignments. Think: the object. |
| **Managed identity (MI)** | A service principal whose credentials Azure itself owns and rotates. You never see a secret. **The default choice for service-to-service auth.** |
| **Workload identity federation** | Lets an *external* system (GitHub Actions, another cloud, Kubernetes) exchange its own token for an Entra token — no exported secrets. This is how pipelines authenticate now (module 11). |

**System-assigned vs user-assigned MI:** system-assigned is created/deleted with the resource (1:1, simplest); user-assigned is a standalone resource shared by many apps (pre-provisionable, survives app recreation, lets you grant roles *before* the app exists — better for IaC). Rule of thumb: user-assigned for anything deployed via pipelines; system-assigned for one-offs.

## 2. Tokens in 90 seconds (what's actually flying around)

Everything is **OAuth 2.0 / OpenID Connect**. Your app requests a token from Entra for a *resource* (audience), e.g. `https://database.windows.net/` or `https://vault.azure.net`. The token is a signed JWT carrying the principal's object ID and roles/scopes; the resource validates signature + audience + expiry. Two flows you must know cold:

- **Client credentials** (service → service, no user): what MI does under the hood.
- **Authorization code + PKCE** (user sign-in in web apps): what `Microsoft.Identity.Web` does for ASP.NET Core.

Delegated permissions (acting *as a user*) vs application permissions (acting *as itself*) — Graph API work makes this distinction matter; get it wrong and admins refuse consent.

## 3. The .NET code — `Azure.Identity` and `DefaultAzureCredential`

The `Azure.Identity` package gives one abstraction that works identically on your laptop and in production:

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Azure.Storage.Blobs;

// ONE credential object for the whole app — register as singleton.
// Locally: uses your az login / Visual Studio sign-in.
// In Azure: uses the app's managed identity. Same code. No secrets.
var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    // In production, pin the user-assigned MI explicitly (faster + unambiguous):
    ManagedIdentityClientId = builder.Configuration["AZURE_CLIENT_ID"]
});

builder.Services.AddSingleton<TokenCredential>(credential);
builder.Services.AddSingleton(new BlobServiceClient(
    new Uri("https://mystorage.blob.core.windows.net"), credential));
builder.Services.AddSingleton(new SecretClient(
    new Uri("https://my-kv.vault.azure.net"), credential));
```

`DefaultAzureCredential` tries a chain (environment → workload identity → managed identity → VS/az CLI…). **Expert notes:** (a) it's for *convenience across environments* — in hot paths in production consider `ManagedIdentityCredential` directly to skip probing; (b) credential objects cache and refresh tokens internally — create once, share everywhere; (c) failures surface as `CredentialUnavailableException` with a per-link explanation — read it, it names which chain link failed and why.

**SQL with no password** (works with `Microsoft.Data.SqlClient`):

```csharp
// Connection string — note: no user, no password.
"Server=tcp:my-srv.database.windows.net;Database=appdb;Authentication=Active Directory Default;"
```

Then in the database, create the contained user for the identity and grant least privilege:

```sql
CREATE USER [my-api-identity] FROM EXTERNAL PROVIDER;      -- name of the MI
ALTER ROLE db_datareader ADD MEMBER [my-api-identity];
ALTER ROLE db_datawriter ADD MEMBER [my-api-identity];
```

**User sign-in for ASP.NET Core** — use `Microsoft.Identity.Web`:

```csharp
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));
builder.Services.AddAuthorization(o =>
    o.AddPolicy("Admins", p => p.RequireRole("App.Admin")));   // app roles from the registration
```

## 4. RBAC done properly

- An assignment is **(security principal, role definition, scope)**. Scope = management group / subscription / RG / resource. Narrowest workable scope wins.
- **Control-plane vs data-plane roles** (module 1's lesson): *Contributor* manages the storage account but cannot read blobs; *Storage Blob Data Reader* reads blobs but can't manage the account. Services with this split include Storage, Key Vault (in RBAC mode), Service Bus, Event Hubs, Cosmos (data-plane RBAC), SQL (via Entra database users).
- **Deny assignments** exist (created by the platform, e.g. deployment stacks) and override allows.
- **PIM (Privileged Identity Management)**: standing privileged access is the anti-pattern; PIM makes elevation just-in-time, time-boxed, approved and audited. The expert stance: *nobody* holds standing Owner on prod.
- **Conditional Access**: tenant-level policy engine (require MFA/compliant device, block legacy auth, restrict by network/risk). You don't configure it as a dev, but your designs assume it exists — e.g. that's why long-lived tokens/cookies get revoked mid-session.

## 5. Key Vault — for the secrets that must exist

MI removes most secrets; third-party API keys, certificates, and encryption keys still need a home.

- **Three object types:** secrets (arbitrary strings, versioned), keys (crypto ops happen *inside* the vault/HSM), certificates (lifecycle + auto-renewal integration).
- **Use RBAC authorization mode** (not legacy access policies) for uniform governance.
- **Two consumption patterns in .NET:**
  1. Config provider — secrets appear in `IConfiguration`: `builder.Configuration.AddAzureKeyVault(new Uri(kvUri), credential);`
  2. **Key Vault references** in App Service/Functions app settings — `@Microsoft.KeyVault(SecretUri=https://my-kv.vault.azure.net/secrets/ApiKey/)` — platform resolves it; app just reads an env var. Zero code change from on-prem.
- **Operational discipline:** soft-delete + purge protection on (default/enforced), one vault per app per environment (blast radius + access clarity), alert on `SecretNearExpiry` events (Event Grid), never log secret values (obvious; still happens).
- **What Key Vault is not:** a database, a config store (that's **App Configuration**, which pairs with KV: config + feature flags in App Config, secrets referenced from KV), or high-throughput per-request storage — cache secrets in memory; the SDK's `SecretClient` doesn't cache for you (App Config provider + KV reference caching does).

## 6. Decision frameworks

**"How does X authenticate to Y?"** — the expert decision tree:
1. Both in Azure? → **Managed identity + RBAC.** Always the first answer.
2. Caller is a pipeline (GitHub/DevOps)? → **Workload identity federation (OIDC)** to a service principal or MI. No PATs, no secrets.
3. Caller is on-prem/third-party? → App registration + client certificate (preferred over client secret) in their vault; or Entra External ID if it's a partner org's users.
4. End users? → `Microsoft.Identity.Web` (employees) / **Entra External ID** (customers).
5. A legacy component that only speaks connection strings? → the string lives in **Key Vault**, surfaced by reference; plan its retirement.

**"Which storage auth: keys, SAS, or Entra?"** → Entra + RBAC by default. SAS only for delegated, time-boxed, third-party access — and then **user-delegation SAS** (signed by your Entra token, revocable, no account key exposure). Account keys: disable them by policy where possible.

## 7. Lab 02 (~2.5 h) — "no secrets anywhere"

> Full steps + cleanup: [labs.md → Lab 02](../labs.md). You build the identity spine of the capstone:

1. Create user-assigned MI `id-cloudboard-dev` + Key Vault (RBAC mode) + storage account, via Bicep.
2. Minimal ASP.NET Core API: reads a secret via KV config provider, writes a blob, both through `DefaultAzureCredential`. Run locally (your `az login` identity — grant yourself the data roles) then deployed (MI). Same code both places — verify with zero config edits.
3. Add Azure SQL (serverless, auto-pause) + Entra-only auth; create the MI's contained user; EF Core connects with `Authentication=Active Directory Default`.
4. Break it on purpose: remove a role assignment, observe the exact exception; re-add and note propagation delay (RBAC changes can take a few minutes — a real operational fact).
5. Inspect the token: `az account get-access-token --resource https://vault.azure.net` → paste into jwt.ms → find `oid`, `aud`, expiry.

## 8. Self-check

1. App registration vs service principal vs managed identity — one sentence each, and which gets the role assignment.
2. Why prefer user-assigned MI in IaC-driven estates? Two reasons.
3. What does `DefaultAzureCredential` resolve to locally vs in App Service, and what's the production-hardening advice?
4. Your API needs to read blobs. Contributor role: yes/no, and what instead?
5. Key Vault reference vs config provider — when each?
6. A GitHub Actions pipeline deploys to Azure. Describe auth with zero stored credentials.
7. What are delegated vs application permissions, and which needs admin consent more often?

## 9. Explain out loud

- Walk a security reviewer through "how our app talks to SQL with no password" — every hop, every artifact.
- Explain to a junior why a leaked *connection string* is an incident but a leaked *client ID* usually isn't.

## 10. Verify before you build

- Entra product names/SKUs (renames are frequent: Entra ID P1/P2, External ID packaging).
- SqlClient `Authentication=Active Directory Default` guidance for the current Microsoft.Data.SqlClient version.
- Whether the target service supports data-plane RBAC generally (a few still key/SAS-first).

*Next: [Module 03 — App Service deep dive](03-compute-app-service.md).*
