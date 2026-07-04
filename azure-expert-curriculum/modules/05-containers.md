# Module 05 — Containers: ACR → Container Apps → AKS (+ Aspire)

> **Week 5.** Containers are how multi-service .NET systems ship. The expert path: build proper images, store them in ACR, default to **Container Apps** for running them, know precisely when AKS is justified — and use **Aspire** to make the local dev loop and deployment sane.

---

## 1. .NET containers done right (the image itself)

- **`dotnet publish /t:PublishContainer`** builds an image with no Dockerfile (SDK container builds) — great default. Write a Dockerfile when you need custom layers.
- **Base image choices:** `mcr.microsoft.com/dotnet/aspnet` (runtime) vs `runtime-deps` + self-contained; **chiseled/distroless variants** = smaller + fewer CVEs — production default. Alpine if you know musl implications.
- **Multi-stage Dockerfile pattern** (build → publish → tiny runtime layer); copy csproj first and `dotnet restore` separately for layer caching.
- **Run as non-root** (chiseled images already do; port 8080 by default in modern .NET images), pin image digests in prod, set `ASPNETCORE_URLS`/`DOTNET_` envs explicitly.
- Health: expose `/healthz` (ASP.NET Core health checks) — every orchestrator below consumes it.

## 2. Azure Container Registry (ACR)

- Private OCI registry. SKUs: Basic/Standard/**Premium** (Premium adds private link, geo-replication, higher throughput).
- **Auth the right way:** pulls via **managed identity** (`AcrPull` role) from ACA/AKS/App Service — no admin user (disable it), no passwords in deploy YAML.
- **ACR Tasks:** cloud-side image builds (`az acr build -t app:{{.Run.ID}} .`) — no local Docker needed; also base-image-update triggers (auto-rebuild when the .NET base patches — a quiet security win).
- Content trust/signing (notation), vulnerability scanning via Defender for Cloud; retention policies to stop tag sprawl.
- Tagging discipline: immutable release tags (`api:1.4.2`, or the git SHA) + moving env tags if you must; **never deploy `:latest`** to prod.

## 3. Azure Container Apps (ACA) — the default runtime

Serverless containers on managed Kubernetes+KEDA+Envoy you don't see. Core model:

- **Environment** = the isolation/network boundary (its own VNet-integratable space, shared Log Analytics). Apps in one environment share it. Workload profiles (consumption + dedicated GPU/CPU profiles) mix serverless and reserved compute.
- **Container App** = your service; **Revisions** = immutable versions of it. Single-revision mode (default) or **multiple-revision mode** → traffic splitting between revisions = **built-in blue-green/canary** (`az containerapp ingress traffic set --revision-weight rev1=90 rev2=10`).
- **Ingress:** internal or external HTTPS out of the box (Envoy), custom domains, session affinity if you must; TCP ingress exists.
- **Scaling = KEDA:** HTTP concurrency, CPU/memory, or **event-driven scalers** (Service Bus queue length, Event Hubs lag, Kafka, cron...). `minReplicas: 0` → scale-to-zero. Example: scale rule on Service Bus queue depth turns your worker into a pay-for-backlog system — module 8's consumer pattern.
- **Dapr integration (optional):** sidecar for service invocation (mTLS), pub/sub, state, secrets — adopt deliberately, not by default.
- **Jobs:** run-to-completion workloads (manual, scheduled, event-triggered) — the container answer to "batch/cron".
- **Sidecars/init containers, volume mounts (Azure Files), secrets refs to Key Vault, MI per app** — all supported; use MI + Key Vault, same as everywhere.

**ACA vs App Service for an API:** ACA wins on microservice count, event-driven scale-to-zero, sidecars, gRPC/h2c, revisions model; App Service wins on the classic single web app with slots and simplest ops. Both are right answers in different shops — argue it by requirements.

## 4. AKS — when Kubernetes is actually justified

Say *no* by default (module's expert signal), *yes* when you need: custom controllers/operators/CRDs, service mesh with fine control, GPU scheduling patterns, strict pod-level network policy regimes, multi-tenant platform engineering, or portability mandates. You take on: upgrades (control plane + node images), node pool capacity management, CNI choice and IP planning, cluster RBAC↔Entra integration, admission policies, observability plumbing.

If you do run AKS, the modern defaults: **managed identity + workload identity federation** for pods (no kubelet identity abuse), **Azure CNI Overlay** networking, **Entra-integrated RBAC**, **auto-upgrade channels + maintenance windows**, **cluster autoscaler (or Node Auto-Provisioning/Karpenter)**, KEDA add-on for event scale, Azure Monitor managed Prometheus + Container Insights, and **AKS Automatic** (opinionated, hands-off mode) for teams that want AKS-with-guardrails. Deploy workloads via GitOps (Flux add-on) or pipelines with `kubectl`/Helm — but that's platform work; this curriculum's center of gravity stays ACA.

## 5. Aspire — the .NET distributed-app dev loop

Aspire (aspire.dev) is the **orchestration + service-wiring + observability layer for local dev**, and a path to deployment:

- **AppHost project** declares your system in C#:

```csharp
var builder = DistributedApplication.CreateBuilder(args);
var sql   = builder.AddSqlServer("sql").AddDatabase("appdb");
var cache = builder.AddRedis("cache");
var bus   = builder.AddAzureServiceBus("bus");            // provisions/emulates per context
var api   = builder.AddProject<Projects.CloudBoard_Api>("api")
                   .WithReference(sql).WithReference(cache).WithReference(bus);
builder.AddProject<Projects.CloudBoard_Worker>("worker").WithReference(bus);
builder.Build().Run();
```

- **ServiceDefaults** project wires OpenTelemetry, health checks, resilience (`AddStandardResilienceHandler`) into every service — one line per service, consistent observability (pays off in module 10).
- **Dashboard:** live resources, console logs, structured logs, traces, metrics across all services — locally, and deployable alongside the app.
- **Deployment:** `azd up` translates the app model to **Azure Container Apps** (or App Service) — provisions ACR, environment, apps, wiring (connection strings/service discovery injected). `azd deploy <service>` for incremental code pushes. This is the fastest route from "solution on laptop" to "running in Azure" that still produces sane infrastructure.
- Honest scope: Aspire ≠ production orchestrator; it *models* the app and hands off to ACA/K8s. Its value = dev-loop speed + consistent defaults + deployment scaffolding.

## 6. Decision framework

Image: SDK-built, chiseled, non-root, digest-pinned. Registry: ACR Premium for prod (private link + geo-rep as needed), MI-only auth, Tasks for builds. Runtime ladder: **single web app → App Service; services/event-driven/sidecars → Container Apps; platform-grade needs → AKS (Automatic first)**. Local dev of anything multi-service → Aspire; deployment scaffold → azd; long-term IaC → export/author Bicep (module 11).

## 7. Lab 05 (~3 h)

> [labs.md → Lab 05](../labs.md): capstone goes multi-service.

1. Containerize the module-03 API with `dotnet publish /t:PublishContainer`; retarget to a chiseled base; compare image sizes.
2. `az acr build` it into ACR; disable admin user; pull-test.
3. Aspire-ify the solution (AppHost + ServiceDefaults + the worker from module 04 as a project); run locally; explore the dashboard's traces across API→bus→worker.
4. `azd up` to Container Apps; verify MI-based ACR pull; then flip the worker to `minReplicas: 0` with a Service Bus KEDA scale rule and watch it wake on a message.
5. Multi-revision canary: deploy v2 of the API, split 90/10, promote, then instant-rollback by weight — write down how this compares to App Service slots.

## 8. Self-check

1. Three concrete reasons chiseled images are the production default.
2. How does an ACA app pull from ACR with no credentials? Name the exact role.
3. Revisions vs slots — map the concepts and name one thing revisions can do that slots can't.
4. Which KEDA scaler turns a queue worker into pay-per-backlog, and what settings matter (min/max, queue length target)?
5. Give three requirements that genuinely justify AKS over ACA — and two that don't (but people claim).
6. What do AppHost and ServiceDefaults each contribute in Aspire?
7. Where do connection strings "come from" when azd deploys an Aspire app?

## 9. Explain out loud

- Argue ACA vs AKS for a 6-service .NET system to a CTO who "wants Kubernetes" — steelman both sides.
- Explain image layer caching to a teammate whose CI rebuilds take 10 minutes.

## 10. Verify before you build

- ACA workload profiles, GPU support, and current limits; Dapr/KEDA versions bundled.
- Aspire release cadence & Functions-on-Aspire status (was preview); azd → App Service path maturity.
- AKS Automatic capabilities/regions; current default CNI guidance; Karpenter/NAP status.

*Next: [Module 06 — Relational data & caching](06-data-sql.md).*
