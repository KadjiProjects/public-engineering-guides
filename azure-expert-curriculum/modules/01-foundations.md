# Module 01 — Foundations & the Azure Resource Model

> **Week 1.** Everything in Azure is written in one grammar: resources, managed by ARM, arranged in a hierarchy, governed by identity + policy. Learn the grammar once and every service becomes a vocabulary word, not a new language.

---

## 1. Why this matters to you as a .NET developer

You already think in solutions, projects, NuGet packages, and MSBuild. Azure has exact analogues, and mapping them early makes everything downstream feel familiar:

| You know | Azure equivalent | The mapping |
|---|---|---|
| `.sln` grouping projects shipped together | **Resource Group** | Lifecycle unit: deploy together, delete together |
| `csproj` (declarative desired state → build engine) | **Bicep/ARM template** | Declarative desired state → ARM reconciles |
| NuGet package identity + version | **Resource ID + API version** | Every resource has a stable, unique, versioned identity |
| Assembly attributes / metadata | **Tags** | Queryable metadata (cost, ownership, environment) |
| Roslyn analyzers blocking bad code at build | **Azure Policy** | Rules blocking/auditing bad resources at deploy |

## 2. The four mental models (internalize before anything else)

1. **Shared responsibility.** Azure secures the platform; you secure what you put in it. The line moves with the service model: IaaS (you own OS upward) → PaaS (app + data) → serverless (code) → SaaS (configuration). *Expert habit: for any service you adopt, say out loud what you still own — on App Service you still own your dependencies' CVEs, your TLS-to-database settings, your data classification.*
2. **The control plane vs the data plane.** The control plane is Azure Resource Manager (ARM): `management.azure.com`, where resources are created/configured, guarded by Azure RBAC. The data plane is the resource doing its work: your query to `myserver.database.windows.net`, your blob upload to `myaccount.blob.core.windows.net`. **They have separate endpoints, separate permissions, and often separate network paths.** A Contributor (control plane) cannot necessarily read blobs (data plane) — that needs a data-plane role like *Storage Blob Data Reader*. This single distinction resolves half of all early confusion.
3. **Everything is a resource with an ID.** Format: `/subscriptions/{sub}/resourceGroups/{rg}/providers/{namespace}/{type}/{name}` — e.g. `.../providers/Microsoft.Web/sites/my-api`. Resource IDs are how RBAC scopes, diagnostic settings, private endpoints, and role assignments all reference things. Learn to read them like you read namespaces.
4. **Declarative + idempotent beats imperative.** ARM deployments describe *desired state*; running the same template twice is safe (create-or-update semantics). This is why the portal is for exploration and IaC is for anything real (module 11).

## 3. The hierarchy and what lives at each level

```
Entra ID Tenant (identity boundary — one per organization)
└── Root Management Group
    └── Management Groups (org structure: platform / landing zones / sandboxes)
        └── Subscriptions (billing + quota + isolation boundary)
            └── Resource Groups (lifecycle units)
                └── Resources
```

- **RBAC and Policy assignments inherit downward.** Assign *Reader* at a management group → it applies to every subscription, RG, and resource beneath. Design the tree so guardrails can be set once, high up.
- **Subscription = blast radius.** Quotas (e.g. vCPU limits), some service limits, and billing separation live here. Typical enterprise pattern: separate subscriptions per environment class (prod / non-prod) or per landing zone, not per app.
- **Resource group rules of thumb:** resources that share a lifecycle live together; a RG has a *location* (stores metadata) but its resources can live in other regions; you can't nest RGs; moving resources between RGs/subscriptions is possible but restricted per type — plan names/placement up front.

### Cross-cutting governance primitives (know all five)

| Primitive | Question it answers | Notes |
|---|---|---|
| **RBAC** | *Who* can do *what*, *where*? | `(principal, role, scope)` triple. Built-in roles first; custom roles when justified. Data-plane roles are separate (see model #2). |
| **Azure Policy** | *What is allowed to exist / how must it be configured?* | Effects: `deny`, `audit`, `append`, `modify`, `deployIfNotExists`. Initiatives = policy bundles. |
| **Tags** | *Whose is it, what's it for, who pays?* | Enforce with policy (`modify` to inherit tags from RG). Minimum viable set: `env`, `owner`, `costCenter`, `app`. |
| **Locks** | *Can it be deleted/changed accidentally?* | `CanNotDelete` / `ReadOnly`. Beware `ReadOnly`: it also blocks some data-plane-ish operations (e.g. listing storage keys). |
| **Resource Graph** | *What exists across the whole estate?* | KQL over all subscriptions in seconds: `az graph query -q "Resources | where type == 'microsoft.web/sites' | project name, location"` |

## 4. ARM mechanics an expert actually uses

- **Resource providers & API versions.** Each type belongs to a provider namespace (`Microsoft.Web`, `Microsoft.Sql`, ...) which must be *registered* on the subscription (usually automatic; a classic gotcha in fresh subscriptions: `az provider register --namespace Microsoft.App`). Every resource operation is versioned (`?api-version=2024-04-01`) — this is why templates pin API versions and why portal/CLI/SDK can show different property names.
- **Deployment scopes.** Templates can target a resource group (most common), subscription (create RGs, assign policy), management group, or tenant. Module 11 exploits this.
- **Idempotency & incremental mode.** Default deployment mode is *incremental*: things in the template are created/updated; things not mentioned are left alone. *Complete* mode deletes unmentioned resources — powerful and dangerous; prefer **deployment stacks** (the modern mechanism for managing a set of resources as a unit with deny-settings and cleanup).
- **Throttling & async operations.** ARM operations are async (202 + polling). CLIs/SDKs hide this, but understanding it explains "provisioningState: Running/Succeeded/Failed" and why `--no-wait` exists.
- **Activity Log** = the control-plane audit trail (who deployed/deleted what, when). First stop for "who changed this?"

## 5. Your tooling stack (set up now — the labs assume it)

| Tool | Role | Install check |
|---|---|---|
| **Azure CLI** (`az`) | Primary scripting/automation surface | `az version`, then `az login`, `az account set -s <sub>` |
| **Azure PowerShell** | Alternative; strong for Windows-ops estates | optional |
| **Bicep CLI** | IaC (bundled with recent az) | `az bicep version` |
| **Azure Developer CLI** (`azd`) | App-centric: provision + deploy templates/Aspire in one (`azd up`) | `azd version` |
| **VS / VS Code + Azure extensions** | Everyday IDE integration | Azure Tools extension pack |
| **Cloud Shell** | Browser terminal with everything preinstalled + a persisted file share | portal `>_` icon |

*Judgment note:* the portal is a **read/explore/diagnose** surface. Anything you did in the portal that survives to next week should exist in code (module 11). Get that habit in week 1, not week 11.

## 6. Regions, zones, and placement (the physical layer)

- **Region** = a set of datacenters. **Availability Zone (AZ)** = one or more physically isolated datacenters in a region (own power/cooling/network). Zone-redundant deployments (≥3 AZs) survive a datacenter failure with no multi-region complexity — the default resilience posture for production.
- **Region pairs** exist for geo-replication and staggered platform updates; some services expose paired-region behavior (GRS storage fails over to the pair).
- **Choosing a region = three constraints together:** user latency, data-residency law, and service/SKU availability (not everything ships everywhere — check before designing).
- Non-obvious extras worth knowing exist: sovereign clouds (Government/China), Edge Zones, and that a few services are *non-regional* (Entra ID, Front Door, DNS).

## 7. Decision frameworks to rehearse

**"Where do I put this new app's resources?"** → One RG per app per environment (`rg-cloudboard-dev`, `rg-cloudboard-prod`), tagged, in a subscription appropriate to its environment class, region chosen by the three constraints, zone-redundant SKUs for prod.

**"Who gets access?"** → Groups, not individuals → role at the narrowest workable scope → data-plane roles for data access → PIM for anything privileged (module 2). Never grant Owner when Contributor + explicit role assignments would do.

**"How do I keep 50 teams from creating chaos?"** → Management-group tree with policy initiatives at the top (allowed regions, required tags, denied SKUs, security baselines) + landing zones (module 12) + Resource Graph for continuous inventory.

## 8. Lab 01 (do it — ~90 min)

> Full instructions with cleanup: [labs.md → Lab 01](../labs.md). Summary:

1. Create your subscription's skeleton with CLI only: RG `rg-learn-01`, tags `env=lab owner=<you>`.
2. Deploy a storage account **three ways** — portal, CLI, and a 10-line Bicep file — and diff what each produced (`az resource show`). Feel why IaC wins.
3. Assign yourself *Storage Blob Data Contributor* on the account; upload a blob with `az storage blob upload --auth-mode login`. Then remove the role and watch the data-plane call fail while the portal (control plane) still shows the account. **This is the control/data-plane lesson in your hands.**
4. Write one Resource Graph query listing every resource you own with its tags.
5. Put a `CanNotDelete` lock on the RG; try to delete; remove lock; delete RG (cleanup).

## 9. Self-check (answer from memory, then verify)

1. A colleague with *Contributor* on a storage account says "access denied" reading a blob with Entra auth. Why, and what's the fix?
2. Where do quotas live — resource group, subscription, or region? What does that imply for prod/non-prod separation?
3. What are the two things a resource ID encodes that RBAC and private endpoints both rely on?
4. Incremental vs complete deployment mode — which is default, and what replaced complete-mode's use case?
5. Name the five governance primitives and one sentence each.
6. Why can a resource group in *West Europe* contain a resource in *Sweden Central*? What is the RG's location actually for?

## 10. Explain out loud (Feynman prompts)

- Explain control plane vs data plane to a junior dev using the storage-account lab as the example.
- Explain why "the portal doesn't scale" without using the word "clicking."

## 11. Verify before you build (drift-prone facts)

- Deployment **stacks** capabilities/regions (they were evolving through 2024–25).
- Current `az` login experience (device code vs broker) and any subscription-selection changes.
- Availability-zone support for the specific SKUs you pick (per region, per service).

*Next: [Module 02 — Identity, access & secrets](02-identity-security.md) — where the "no secrets anywhere" pattern becomes real .NET code.*
