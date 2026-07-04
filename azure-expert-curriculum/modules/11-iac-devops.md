# Module 11 — Infrastructure as Code & Delivery

> **Week 11.** Everything you clicked in labs 1–10 becomes code, and every deployment becomes a pipeline with zero stored secrets. Bicep for infrastructure, GitHub Actions with OIDC federation for delivery, environments and progressive rollout for safety. This module is where the whole curriculum compounds.

---

## 1. Bicep — from zero to modules

Bicep is Azure-native declarative IaC that compiles to ARM JSON. It is the first language to master (Terraform second, for multi-cloud shops — concepts transfer 1:1).

```bicep
// main.bicep — the shape of real files
@description('Environment name used in resource names')
@allowed(['dev', 'test', 'prod'])
param env string
param location string = resourceGroup().location
@secure()
param sqlAdminObjectId string          // no secret *values* in code — IDs and references only

var appName = 'cloudboard-${env}'

resource plan 'Microsoft.Web/serverfarms@2024-04-01' = {
  name: 'asp-${appName}'
  location: location
  sku: { name: env == 'prod' ? 'P1v3' : 'B1' }
  properties: { reserved: true }                 // Linux
}

resource app 'Microsoft.Web/sites@2024-04-01' = {
  name: 'app-${appName}'
  location: location
  identity: { type: 'UserAssigned', userAssignedIdentities: { '${uami.id}': {} } }
  properties: {
    serverFarmId: plan.id                        // implicit dependency — no dependsOn needed
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|10.0'
      healthCheckPath: '/healthz'
      appSettings: [
        { name: 'AZURE_CLIENT_ID', value: uami.properties.clientId }
        { name: 'KeyVault__Uri',   value: kv.outputs.uri }
      ]
    }
  }
}

module kv 'modules/keyvault.bicep' = {           // modules = your reusable pattern library
  name: 'kv'
  params: { env: env, readerPrincipalId: uami.properties.principalId }
}

output appHostName string = app.properties.defaultHostName
```

**Language essentials:** `param` (+decorators `@secure`, `@allowed`, `@minLength`), `var`, `resource` (symbolic names create implicit `dependsOn`), `module`, `output`, `existing` (reference without deploying: `resource law 'Microsoft.OperationalInsights/workspaces@...' existing = { name: ... }`), loops (`[for x in list: {...}]`), conditions (`= if (env == 'prod')`), string interpolation, `uniqueString(resourceGroup().id)` for globally-unique names.

**Idioms of production Bicep:**
- **Modules per pattern**, not per resource: `web-app.bicep`, `sql-private.bicep`, `private-endpoint.bicep` (the module 9 stamp: PE + DNS zone + zone group + link). Publish shared ones to a **Bicep registry** (ACR: `br:myacr.azurecr.io/bicep/modules/...`), or start from **Azure Verified Modules (AVM)** — Microsoft's maintained module library.
- **RBAC in code:** role assignments are resources (`Microsoft.Authorization/roleAssignments` with `guid()` names) — the module 2 "MI + role" spine belongs in Bicep, not in a wiki.
- **Scopes:** RG deployments mostly; subscription scope (`targetScope='subscription'`) to create RGs + policy; management-group scope for governance.
- **What-if before every apply:** `az deployment group what-if -g rg -f main.bicep -p env=prod` — the plan/preview. **Deployment stacks** (`az stack group create`) manage a *set* as a unit: deny out-of-band changes, clean up removed resources — the modern answer to drift and to complete-mode.
- **Parameter files:** `.bicepparam` per environment; secrets never in files — they're `@secure()` params sourced from Key Vault or federated pipeline context.

## 2. Pipelines — GitHub Actions with OIDC (zero deploy secrets)

Workload identity federation (module 2) applied: the pipeline exchanges its GitHub-issued OIDC token for an Entra token. **No publish profiles, no service-principal passwords stored anywhere.** Setup once: app registration (or UAMI) + federated credential matching `repo:org/repo:environment:prod` + RBAC on the target scope.

```yaml
# .github/workflows/deploy.yml
permissions: { id-token: write, contents: read }        # id-token = the OIDC magic

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }
      - run: dotnet test --configuration Release
      - run: dotnet publish src/Api -c Release -o publish
      - uses: actions/upload-artifact@v4
        with: { name: api, path: publish }

  deploy-prod:
    needs: build
    environment: prod                                    # gates: approvals, wait timers, branch rules
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}         # IDs, not secrets
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Infra (what-if is a separate PR job)
        run: az deployment group create -g rg-cloudboard-prod -f infra/main.bicep -p infra/prod.bicepparam
      - uses: actions/download-artifact@v4
        with: { name: api, path: publish }
      - name: Deploy to staging slot, then swap
        run: |
          az webapp deploy -g rg-cloudboard-prod -n app-cloudboard-prod --slot staging --src-path publish --type zip
          # smoke test the slot here
          az webapp deployment slot swap -g rg-cloudboard-prod -n app-cloudboard-prod --slot staging
```

**Pipeline architecture that scales:** PR workflow = build + tests + `what-if` posted as a PR comment; main → dev auto-deploy; prod behind an **environment approval**. Database migrations as an explicit job (module 6: pipeline-applied, expand→contract). Azure DevOps Pipelines: same concepts (service connections with workload identity federation, environments, approvals) — translate freely.

## 3. Progressive delivery — deploy ≠ release

- **Blue-green:** App Service slots swap (module 3) / ACA revision at 100→0 flip (module 5). Instant rollback = swap back.
- **Canary:** slot traffic-split or ACA revision weights (10% → observe golden signals, module 10 → promote). Automate promotion on SLO health if you're mature.
- **Feature flags decouple release from deploy:** **Azure App Configuration** feature manager + `Microsoft.FeatureManagement.AspNetCore` — dark-launch code paths, target by percentage/user, kill-switch instantly without redeploying. Config changes without restarts (App Config sentinel-key refresh pattern).
- **Rollback discipline:** artifacts immutable + versioned; infra rollback = redeploy previous template (stacks help); data rollback = *doesn't exist* — hence expand/contract migrations (never destructive in the same release that deploys the code using them).
- **azd** (module 5) spans this space for app-centric repos: `azd provision`/`azd deploy` with Bicep in `infra/` — a great starting skeleton that you then own and harden as above.

## 4. Environments & the estate

Dev/test/prod as **separate resource groups minimum, separate subscriptions preferably** (quota + blast radius + cost clarity, module 1). Same Bicep, different `.bicepparam`. Policy enforces the guardrails (allowed SKUs in dev, required ZR in prod). Ephemeral PR environments (deploy per-PR to a disposable RG, destroy on merge) are the power move when cost allows — stacks make teardown clean.

## 5. Decision framework

Bicep unless multi-cloud (then Terraform, same discipline). AVM modules before writing your own; registry-published internal modules for house patterns. OIDC federation always — treat any stored deploy credential as a finding. `what-if` gating; stacks for lifecycle; approvals on prod; slots/revisions for zero-downtime; flags for risk decoupling; migrations as explicit expand/contract jobs. The test of success: **`git clone` → one pipeline run → a working, secured, observable environment.** If any step needs portal clicks, it isn't done.

## 6. Lab 11 (~4 h — the consolidation lab)

> [labs.md → Lab 11](../labs.md)

1. Backfill: turn your labs-01→10 estate into `infra/` Bicep — modules for web app, SQL+PE, Service Bus, Cosmos, KV, observability (diagnostic settings loop). Deploy to a fresh `rg-cloudboard-dev2` and diff against the hand-built one (`what-if` against the old RG is illuminating).
2. Set up OIDC: UAMI + federated credentials for `dev` and `prod` GitHub environments; delete every stored secret from the repo settings.
3. Build the PR workflow (test + what-if comment) and the deploy workflow above; make prod require your manual approval.
4. Progressive: deploy a visible v2 via slot + 20% traffic; add a feature flag in App Configuration wrapping the new endpoint; kill-switch it live.
5. Deployment stack: recreate dev as a stack with deny-settings; try a portal edit and watch it blocked; delete the stack and watch clean teardown.
6. Finale: delete `rg-cloudboard-dev2` entirely and restore it with one pipeline run. Time it. That number is your new disaster-recovery brag.

## 7. Self-check

1. What does `what-if` compare, and name two changes it's known to be noisy about?
2. Deployment stacks vs complete mode — what problems do stacks solve that mode never did?
3. Walk the OIDC token exchange: who issues what, what's matched in the federated credential, what's never stored.
4. Where do secret *values* live in a healthy pipeline, and how does Bicep receive them?
5. Slot swap vs ACA revision weights vs feature flag — which risk does each mitigate?
6. Why must schema migrations be backward-compatible for one release (tie to slots)?
7. What belongs in a module vs the root template? Your rule.

## 8. Explain out loud

- Explain to an auditor how production is deployed with zero stored credentials and full approval trail.
- Defend "the portal is read-only for prod" to a team that hotfixes by clicking.

## 9. Verify before you build

- Deployment stacks GA surface & deny-settings behavior.
- AVM module coverage for your services (rapidly growing).
- `azure/login` + federated credential subject formats (environments vs branches vs tags).

*Next: [Module 12 — Architecture, reliability & cost](12-architecture.md) — the capstone.*
