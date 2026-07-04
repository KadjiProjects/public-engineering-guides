# The Lab Track — Building "CloudBoard"

> One capstone application, built progressively across all 12 modules. By module 12 you will have designed, secured, deployed, wired, observed, and cost-optimized a real multi-service system — and rebuilt it from code in one pipeline run. **This is where knowledge becomes expertise.**

## What CloudBoard is

A deliberately simple multi-tenant kanban/task SaaS — simple *domain* so all your attention goes to the *platform*:

- **API** (ASP.NET Core): boards, cards, members — lands on App Service (lab 3), containerized (lab 5).
- **Worker** (Functions, isolated): processes card events, sends notifications — Flex Consumption (lab 4).
- **Data:** Azure SQL (boards/members, lab 6), Cosmos DB (high-volume card activity, lab 7), Blob (attachments, lab 7), Redis (cache/session, lab 6).
- **Spine:** Service Bus (commands + events, lab 8) with outbox + idempotent consumers.
- **Platform:** managed identities everywhere (lab 2), private networking (lab 9), OTel observability (lab 10), Bicep + GitHub OIDC pipeline (lab 11), multi-region resilience (lab 12).

## Ground rules (read once, apply always)

1. **Subscription & budget:** use a personal/dev subscription. Create a **budget alert at €25 and €50** on day one (`Cost Management → Budgets`). Every lab ends with cleanup; between weekly sessions, delete or deallocate anything billable you're not using. Serverless/smallest SKUs are specified deliberately — the concepts scale, the bill shouldn't.
2. **Naming:** `rg-cloudboard-<lab|env>`, `app-cloudboard-…`, tags `env`, `owner`, `module` on everything (you'll query them in lab 1 and thank yourself in cost views).
3. **Two files always open:** a `journal.md` (commands you ran, errors you hit, fixes — this becomes your personal runbook) and the module you're in.
4. **When stuck 30+ minutes:** that *is* the lab. Diagnose with the module's tools (log stream, Kudu, KQL, `nslookup`) before searching. Then write the fix in the journal.
5. **AI assistants are allowed and encouraged** — as the executor, with *you* deciding what to do and why (that's the expertise split from the Concepts Map). Never paste a command you can't explain.

## Prerequisites (one-time, ~1 h)

- .NET 10 SDK, Azure CLI (`az login` works), Bicep (`az bicep version`), azd, Docker Desktop (module 5+), Functions Core Tools v4, VS Code/VS + Azure extensions, a GitHub account + empty `cloudboard` repo, `hey` or `bombardier` for load, Azurite for local storage emulation.

---

## Lab 01 — Foundations (Module 01, ~90 min)
**Goal:** feel the resource model, control vs data plane, and why IaC.
1. `az group create -n rg-cloudboard-lab01 -l swedencentral --tags env=lab owner=<you> module=01`
2. Storage account three ways — portal, `az storage account create`, and a minimal `storage.bicep` (param name/location, one resource, one output). `az resource show` each; diff the JSON; note what the portal set silently.
3. Data-plane lesson: grant yourself `Storage Blob Data Contributor` **scoped to the account**; `az storage blob upload --auth-mode login …` succeeds; remove role; retry → 403 while `az storage account show` (control plane) still works. Journal the two token audiences involved.
4. Resource Graph: `az graph query -q "Resources | where tags.owner == '<you>' | project name, type, location, tags.module"`.
5. Lock `CanNotDelete` on the RG → try delete → remove → delete RG. **Cleanup: RG gone.**

## Lab 02 — Identity spine (Module 02, ~2.5 h)
**Goal:** the passwordless pattern working locally *and* deployed, unchanged.
1. `rg-cloudboard-dev`. Bicep: user-assigned MI `id-cloudboard-dev`, Key Vault (RBAC mode, purge protection), storage account. Role assignments *in Bicep*: your user + the MI get `Key Vault Secrets User` and `Storage Blob Data Contributor`.
2. Scaffold the CloudBoard API (`dotnet new webapi`): `/health` endpoint; `GET /config-demo` reads secret `Demo--Message` via KV config provider; `POST /attachments` writes a blob. One `DefaultAzureCredential` singleton.
3. Run locally (your `az login` identity). Seed the secret. Both endpoints work with zero secrets in any file.
4. Add Azure SQL: logical server (Entra-only admin = you), **serverless** DB (auto-pause 60 min, min 0.5 vCore). Connect with `Authentication=Active Directory Default`; `CREATE USER [id-cloudboard-dev] FROM EXTERNAL PROVIDER` + roles.
5. Break/observe: remove the MI's KV role → exact exception text into the journal → restore (note RBAC propagation minutes).
6. Token anatomy: `az account get-access-token --resource https://vault.azure.net` → jwt.ms → journal `aud`, `oid`, `exp`. **Cleanup: keep RG (it's the dev spine); DB auto-pauses.**

## Lab 03 — App Service (Module 03, ~3 h)
**Goal:** production-shaped hosting with slots you've actually swapped.
1. Extend Bicep: Linux plan P0v3 (or B1 if budget-tight; note the slot limitation you accept), app + `staging` slot, MI attached, health check `/health`, ARR affinity off, `WEBSITE_RUN_FROM_PACKAGE=1`, app settings incl. KV reference for `Demo--Message`.
2. Deploy v1 (`dotnet publish` → zip → `az webapp deploy`). Verify KV reference resolved (portal shows green check).
3. Slot drill: v2 changes the demo endpoint's response. One sticky setting (`SlotName`), one traveling setting (`ReleaseLabel`). **Write your post-swap prediction in the journal first**, swap, verify. Then 20% traffic-split to staging; `curl -s https://…/config-demo -H` in a loop; watch `x-ms-routing-name` distribute.
4. Autoscale rule CPU>60% (min 1 max 3 for lab); `hey -z 3m -c 50 <url>`; watch instance count metric; find the scale event in Activity Log.
5. Diagnostics tour: cause a 500 on `/boom`; find it via log stream → Kudu → note the gap App Insights will fill. **Cleanup: scale plan to B1/stop app between sessions.**

## Lab 04 — Functions worker (Module 04, ~3 h)
**Goal:** event-driven worker on Flex, DLQ story rehearsed, Durable felt.
1. `func init CloudBoard.Worker --worker-runtime dotnet-isolated`; local run against Azurite + a dev Service Bus namespace (Basic tier, queue `card-events`).
2. API gains `POST /cards` → publishes `CardCreated` via `ServiceBusClient` (DI, MI auth). Worker's `ServiceBusTrigger` uses **identity-based connection** (`SbConn__fullyQualifiedNamespace`).
3. Bicep: Flex Consumption app (Linux, .NET isolated), one always-ready instance off first. Deploy; measure cold start (first hit after 20 min idle) vs warm; then always-ready=1 and re-measure. Journal the numbers.
4. Poison drill: messages with `"poison": true` throw. Watch DeliveryCount → DLQ at 5. Peek DLQ (`az servicebus … receive --peek` or Service Bus Explorer in portal); write the runbook entry: detect → inspect → resubmit/drop → prevent.
5. Durable mini-saga `FulfillCardImport`: two activities (validate, import) + compensation (mark-failed) on error; kill `func` mid-orchestration locally; restart; watch replay finish it. **Cleanup: Flex scales to zero; SB Basic is pennies.**

## Lab 05 — Containers & Aspire (Module 05, ~3 h)
**Goal:** the system runs as containers on ACA; Aspire owns the dev loop.
1. `dotnet publish /t:PublishContainer` the API; retag onto a chiseled base; `docker images` size comparison in the journal.
2. ACR (Basic for lab): `az acr build -t cloudboard/api:{{.Run.ID}} .`; disable admin user.
3. Aspire: add AppHost + ServiceDefaults; AppHost declares sql (connection to your Azure dev DB or local container — your choice), redis container, servicebus, api, worker. `aspire run` / F5 → dashboard → find one trace crossing API→SB→worker.
4. `azd init` (use the Aspire template flow) → `azd up` → Container Apps environment. Verify: ACR pull via MI, env vars injected, both apps healthy.
5. KEDA: worker `minReplicas 0`, scale rule on `card-events` queue length (target 5). Send 50 messages; watch replicas rise then fall to zero.
6. Canary: v2 of API image → new revision at 10% → promote to 100% → instant rollback by re-weighting. Journal: revisions vs slots comparison table. **Cleanup: `azd down` if not continuing same week; keep ACR.**

## Lab 06 — SQL & Redis (Module 06, ~3 h)
**Goal:** resilient EF Core against real cloud behavior; cache that survives Redis dying.
1. EF Core model (Boards/Cards/Members); migrations applied **via CLI** in a pipeline-shaped step (`dotnet ef migrations bundle` or script), not at startup. `EnableRetryOnFailure` on.
2. Feel auto-pause: wait for pause (or `az sql db pause`-equivalent via serverless idle), cold-hit the API; in App Insights dependencies (lab 10 will formalize) or logs, find the retried attempt.
3. Query Store: `hey` load on a list endpoint with a deliberate missing index; portal → Query Performance Insight → top query; add index migration; re-measure P95 before/after (journal numbers).
4. Redis (Basic C0 for lab): `AddStackExchangeRedisCache` + HybridCache on `GET /boards/{id}`; measure P95 cached vs not; then **stop Redis** and prove graceful fallback (cache-aside catches, hits SQL, logs warning). **Cleanup: delete Redis between sessions (it bills hourly); SQL pauses itself.**

## Lab 07 — Cosmos & Blob (Module 07, ~3 h)
**Goal:** partition-key scars, earned safely; storage done the SaaS way.
1. Cosmos serverless account; container `activity` pk `/tenantId`. Seeder console app: 10k events, 80% tenant-A.
2. RU table: point read vs single-partition query vs cross-partition query — journal `RequestCharge` for each (SDK response property).
3. Hot-partition fix: switch to synthetic pk `tenantId_bucket` (bucket = hash(cardId)%8) in a v2 container; re-run the 80% load; compare throttling/latency.
4. Change feed → Functions Cosmos trigger maintaining `cardCountByBoard` in a `projections` container; stop the worker, add events, start — watch catch-up.
5. Blob: `POST /attachments` v2 returns a **user-delegation SAS** (15 min, single blob, create-only); upload via `curl` directly to storage; lifecycle rule Cool@30d; Event Grid blob-created → worker logs it. **Cleanup: serverless Cosmos idles free-ish; delete seeded data.**

## Lab 08 — Messaging spine (Module 08, ~3 h)
**Goal:** outbox + idempotency + sessions, provably correct.
1. Upgrade SB to Standard; topic `board-events`, subscriptions `notifications` (filter: `type='CardCreated'`) and `audit` (all); queue `orders` with dedup + MaxDeliveryCount 5.
2. **Outbox:** API writes card + outbox row in one EF transaction; `OutboxRelay` BackgroundService publishes → topic, marks sent. Chaos-test: `kill` the relay mid-batch, restart, prove no loss/no dupes (audit subscription count = SQL count).
3. Idempotent consumer: `ProcessedMessages` table gate in the worker; replay the same message 3× (temporarily disable dedup) → one effect.
4. Sessions: queue `board-ops` session-enabled, `SessionId=boardId`; two worker instances; interleave ops for 3 boards; prove per-board order in logs, cross-board parallelism.
5. DLQ tooling: tiny `dlq-tool` console (peek/resubmit/purge) — you'll reuse it your whole career. **Cleanup: SB Standard ~cheap; downgrade if idle weeks.**

## Lab 09 — Private networking (Module 09, ~3.5 h)
**Goal:** private-by-default, and the DNS debugging reflex.
1. Bicep hub (`10.0.0.0/16`: AzureBastionSubnet, `dns`, `shared`) + spoke (`10.1.0.0/16`: `app-int` delegated Microsoft.Web, `endpoints`), peered both ways.
2. Private endpoints for SQL + Key Vault into `endpoints`; public access **disabled**; private DNS zones (`privatelink.database.windows.net`, `privatelink.vaultcore.azure.net`) with zone groups, linked to hub+spoke.
3. App Service VNet integration into `app-int`; app works; Cloud Shell `nslookup` shows public CNAME chain, `sqlcmd` from outside fails — journal both proofs.
4. **DNS forensics:** cheapest B-series VM in `shared` (Bastion access, no public IP). `nslookup` both services (expect 10.x). Now unlink the SQL zone from the spoke → app breaks; capture the *exact* error and the nslookup evidence; relink; document the debug sequence as a runbook.
5. Front Door (Standard) in front: probe `/health`, WAF Detection mode; App Service access restriction = `AzureFrontDoor.Backend` service tag + `X-Azure-FDID` check; prove direct URL now 403s. **Cleanup: delete VM + Bastion (₵₵!) same day; FD Standard is modest; keep the Bicep.**

## Lab 10 — Observability (Module 10, ~3 h)
**Goal:** one trace across the system; alerts that actually fired.
1. OTel distro (`UseAzureMonitor`) in API + worker — or confirm Aspire ServiceDefaults already did; one workspace-based App Insights per env.
2. Generate a `POST /cards` → find the **single trace**: HTTP → SB publish → worker → SQL + Cosmos, in the transaction view. Screenshot for your portfolio.
3. Custom signals: `ActivitySource` span `ImportBoard`, counter `cards_created` (tenant tag). Query both in KQL.
4. Save the module's five KQL queries against real load (include `/boom` failures).
5. Alerts: metric (5xx>5 in 5m), log (failure-rate query), availability test (3 regions → `/health`), all → action group (your email). Trip each once, deliberately.
6. Golden-signals workbook; then incident drill: stop Redis (lab 06 fallback) + watch latency alert → dashboard → trace → "root cause" in ≤10 min. Journal the timeline. **Cleanup: availability tests off between sessions if noisy.**

## Lab 11 — IaC & pipeline (Module 11, ~4 h)
**Goal:** clone → pipeline → environment. Zero portal, zero secrets.
1. Backfill `infra/`: modules `web-app.bicep`, `sql-private.bicep`, `servicebus.bicep`, `cosmos.bicep`, `keyvault.bicep`, `observability.bicep` (diagnostic-settings loop), `private-endpoint.bicep` (the lab-09 stamp). Root `main.bicep` + `dev.bicepparam`/`prod.bicepparam`.
2. Prove parity: deploy to `rg-cloudboard-dev2`; `what-if` against the original RG; reconcile drift; delete dev2.
3. OIDC: UAMI `id-github-cloudboard` + federated credentials for `environment:dev` and `environment:prod`; Contributor on the RGs + needed data roles. Delete every repo secret.
4. Workflows: `pr.yml` (build, test, `what-if` → PR comment) and `deploy.yml` (build → dev auto → prod behind approval; staging-slot deploy + smoke + swap; EF migration bundle step first, expand/contract discipline noted).
5. App Configuration + feature flag `NewCardView` wrapping a visible change; deploy dark; flip live; kill-switch.
6. **The finale:** delete the entire dev RG. Re-run the pipeline. Time to working system = your headline number. **Cleanup: it IS the cleanup.**

## Lab 12 — Capstone hardening & the exam (Module 12, full day)
1. Scenario C1 ([assessment.md](assessment.md)) cold: 60 min, one page. Self-score with the rubric; gap list.
2. Harden CloudBoard: SQL failover group → paired region; FD priority routing to a passive secondary app; storage → RA-GZRS. Update Bicep, not portal.
3. **Failover drill:** stop the primary app (simulate region loss); measure real RTO on your dashboard; fail back. Journal every rough edge — that's your runbook.
4. Chaos Studio: one experiment (App Service shutdown or SQL failover) with golden signals watching.
5. WAF assessment on CloudBoard → fix top 3 → export report. Write 5 ADRs. Record the 10-minute whiteboard pitch.
6. **Final cleanup decision:** keep CloudBoard alive as your living portfolio at minimum SKUs (~€/day — budget-check), or `az group delete` everything and keep the repo — which can resurrect it in one run. You proved that in lab 11.
