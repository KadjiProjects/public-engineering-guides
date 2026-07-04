# Module 03 — App Service Deep Dive

> **Week 3.** App Service is the .NET workhorse: the most common landing place for ASP.NET Core APIs and sites. "Expert" here means knowing the plan/app split, the deployment machinery, configuration precedence, slots, scaling behavior, and its networking story — well enough to debug production at 2 a.m.

---

## 1. The mental model: Plan vs App

- An **App Service *Plan*** is the compute: a set of VM instances of a given SKU in a region (Windows or Linux). You pay for the plan, not the app.
- An **App (site)** is a deployable unit that *runs on* a plan. Many apps can share one plan (they share its instances' CPU/RAM — noisy-neighbor risk within your own plan).
- Every instance of the plan runs **every app** on that plan. Scaling out the plan scales all its apps together.

**SKU ladder (judgment, not memorization):** Free/Shared (dev toys) → **Basic** (dev/test, no slots on B1... check current) → **Standard** (slots, autoscale, backups) → **Premium v3/v4** (faster cores, more slots, VNet integration performance, zone redundancy) → **Isolated (App Service Environment v3)** (dedicated, fully VNet-injected — for hard compliance/network requirements only; big cost step). Production default: **Premium v3+ with zone redundancy** where the SLA demands it.

**Linux vs Windows:** modern .NET runs great on Linux (cheaper, containers native). Choose Windows only for .NET Framework, or Windows-specific dependencies (GDI, COM, some legacy auth).

## 2. Configuration — where settings actually come from

Precedence as your ASP.NET Core app sees it: **App settings (portal/IaC) become environment variables and override `appsettings.json`.** Nested keys use `__` (double underscore): app setting `ConnectionStrings__Default` ↔ `Configuration.GetConnectionString("Default")`.

- **Slot settings ("deployment slot setting" checkbox):** a setting marked sticky stays with the *slot* during a swap (e.g. `Environment=Staging` stays on staging). Unmarked settings travel with the *app content*. Getting this wrong is the classic swap incident — rehearse it in the lab.
- **Key Vault references** (module 2) for secrets; **App Configuration** for shared/feature-flag config.
- **Connection strings section** vs app settings: on Windows they map to specific env-var prefixes (`SQLAZURECONNSTR_` etc.); with ASP.NET Core, plain app settings via `ConnectionStrings__` are simpler and portable.
- Changing app settings **restarts the app** (workers recycle). Plan config changes around traffic.

## 3. Deployment — the machinery under `dotnet publish`

| Method | What happens | When to use |
|---|---|---|
| **Zip deploy / OneDeploy** | Push a zip; Kudu extracts it | The default for pipelines (`az webapp deploy`) |
| **Run-from-package** (`WEBSITE_RUN_FROM_PACKAGE=1`) | App runs *directly from* the mounted zip — read-only wwwroot, atomic switch, faster cold start | **Production best practice** for most .NET apps |
| **Container** | You ship an image (from ACR); App Service runs it | When you need OS-level dependencies or parity with Container Apps |
| CI integrations (GitHub Actions / DevOps) | Wrap the above | Always — humans shouldn't hand-deploy (module 11) |

**Kudu / SCM site** (`https://<app>.scm.azurewebsites.net`): the management sidecar — file browser, deployment logs, process explorer, console. Learn it; it's your black-box flight recorder. Note it authenticates separately and can be locked to the VNet.

**Zero-downtime deploys = slots.** Deploy to `staging` slot → warm it (health endpoint, smoke tests) → **swap** (App Service swaps *routing*, after applying sticky settings and warming the target). Auto-swap exists; canary exists via **traffic splitting** (route e.g. 10% to a slot — "testing in production" with real users). Rollback = swap back (fast, but remember: anything the new code wrote to the database stays — schema changes need their own strategy: expand/contract, module 12).

## 4. Scaling & reliability behavior

- **Scale up** = bigger SKU (restart). **Scale out** = more instances. Autoscale rules (Standard+) on metrics (CPU, memory, HTTP queue length) or schedule; **always set min ≥ 2 for prod** (and ≥ 3 zone-redundant instances when ZR is on).
- **Automatic scaling** (newer, per-app HTTP-based scaling on Premium) vs classic autoscale rules — know both exist; check current capabilities.
- **Statelessness is on you:** instances are interchangeable; ARR affinity (sticky cookies) is a crutch — turn it **off** and externalize state (Redis, module 6). In-memory session + scale-out = intermittent bugs.
- **Health check** (`/healthz` path setting): instances failing it get pulled from rotation and replaced. Wire it to real dependencies (DB reachable?) but keep it cheap.
- **Always On**: keeps the app warm (off = idle unload = first-hit latency). On for anything real; irrelevant on Flex/Functions consumption (different model).
- **Zone redundancy (Premium v3+):** instances spread across AZs. Combined with min-3 instances, this is the single biggest availability upgrade per euro.

## 5. Networking (the part app devs skip — don't)

Two distinct directions, two features:

- **Inbound — Private Endpoint:** gives the *app* a private IP in a VNet; combine with "disable public access". For internal APIs, this + private DNS zone (`privatelink.azurewebsites.net`) is the pattern (mechanics in module 9).
- **Outbound — VNet Integration (regional):** lets the app's *outbound* calls originate inside a subnet you own → reach private endpoints of SQL/Storage, on-prem via VPN/ER, and get a stable egress via NAT Gateway. Requires a delegated subnet; Premium recommended.
- **Access restrictions:** IP/service-tag/VNet rules on the app and (separately!) on the SCM site.
- The default `*.azurewebsites.net` is public with a Microsoft TLS cert; custom domains + managed certificates are built in; put **Front Door or App Gateway (WAF)** in front for anything internet-facing (module 9), and lock the app to accept traffic only from it (`X-Azure-FDID` / service tag restriction).

## 6. Diagnostics — the 2 a.m. toolkit

1. **Log stream** + App Service logs (stdout/stderr; on Linux, `docker logs` equivalents).
2. **"Diagnose and solve problems"** blade — genuinely good automated detectors (CPU, memory, restarts, SNAT).
3. **Kudu process explorer + memory dumps**, profiler traces.
4. **App Insights** (module 10) — the real answer; enable from day one.
5. **Metrics that matter:** HTTP 5xx, response time P95, HTTP queue length, CPU/memory per instance, SNAT port usage (the classic "mysterious outbound failures under load" — mitigations: connection reuse/`HttpClientFactory`, NAT Gateway, private endpoints).
6. Know the **file system model**: `%HOME%` (`/home`) is shared, persisted across instances (unless run-from-package read-only wwwroot); everything else is per-instance and ephemeral. Writing uploads to local disk is the classic migration bug (→ Blob, module 7).

## 7. Decision framework

**App Service vs the alternatives:** it wins when you want PaaS web hosting with slots, easy custom domains/TLS, and no container pipeline. Container Apps wins for many small services/event-driven scale/sidecars (module 5). Functions wins for event-driven units (module 4). Step down to VMs only under real constraints. Within App Service: Linux P-v3, run-from-package, min 2–3 instances, ZR for prod, slots for every deploy, health check on, ARR affinity off, MI for all dependencies, VNet-integrated egress, WAF-fronted ingress.

## 8. Lab 03 (~3 h)

> [labs.md → Lab 03](../labs.md): the capstone API lands on App Service.

1. Bicep: Linux plan (P0v3/P1v3), app with system+user MI, staging slot, health check, app settings incl. a Key Vault reference. ARR affinity off.
2. Deploy via zip from CLI; flip to run-from-package; observe wwwroot become read-only (Kudu).
3. Slot exercise: deploy v2 to staging with one sticky setting + one traveling setting; predict post-swap state *in writing*; swap; verify prediction. Then split 20% traffic to staging and watch requests distribute (`x-ms-routing-name`).
4. Autoscale: rule on CPU > 60%; generate load (`hey`/`bombardier`); watch instances rise in metrics; find the same event in the Activity Log.
5. Break-glass: cause an exception on one endpoint, find it via log stream, then via Kudu, then note how much faster App Insights would have been.

## 9. Self-check

1. Two apps on one plan: one gets a traffic spike — what happens to the other, and what are your two structural fixes?
2. Explain exactly what "swap" swaps, and what sticky settings do during it.
3. Why is run-from-package the production default? Two concrete benefits.
4. Inbound private endpoint vs outbound VNet integration — a sentence each; can an app need both?
5. Where can your app write files that survive restarts and are shared across instances — and why shouldn't it (usually)?
6. What is SNAT exhaustion, its symptom, and two mitigations?
7. Your health check endpoint calls the database. Defend or attack this design.

## 10. Explain out loud

- Narrate a zero-downtime release of a breaking-schema change (slots + expand/contract) to a release manager.
- Explain to a cost reviewer why prod runs 3 zone-redundant P1v3 instances instead of one P3v3.

## 11. Verify before you build

- Current SKU lineup (Premium v4 rollout, per-region ZR support) and which tiers include slots/how many.
- Automatic scaling (per-app) capabilities vs classic autoscale.
- .NET version support matrix on App Service (LTS/STS timelines) and Linux vs Windows parity gaps.

*Next: [Module 04 — Functions & serverless](04-serverless-functions.md).*
