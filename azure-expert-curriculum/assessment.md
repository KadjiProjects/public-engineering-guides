# Assessment — Milestone Gates & Design Exercises

> Three gates (after modules 4, 8, 12). Each has **recall checks** (closed book, write answers, then verify against the modules) and a **design exercise** with a model answer + rubric. Do the exercise *before* reading the model answer — the struggle is the learning. Passing bar: you'd be comfortable defending your answer to a senior architect.

---

## Gate M1 — "Builder" (after Module 04)

### A1. Recall (closed book, 20 min)
1. Draw the resource hierarchy and mark where RBAC/Policy inherit.
2. Control plane vs data plane: definition + the storage-account example + which roles belong to which.
3. The passwordless pattern end-to-end: MI → role → token → SQL/KV/Blob (name the .NET types involved).
4. App Service: what a slot swap actually swaps; sticky vs traveling settings.
5. Functions 2026 realities: isolated vs in-process status + dates; why Flex Consumption; identity-based connections syntax.
6. Idempotency: why at-least-once forces it; one implementation.

### A2. Design exercise (45 min)
**Scenario:** Internal expense-approval tool. 500 employees, EU data residency, spiky usage (month-end), one dev team of 3, budget-sensitive. Needs: web API + UI, background receipt-OCR step (3rd-party API), SQL data, file uploads, Entra SSO, no public database exposure, deployments without downtime.

**Produce:** service choices + why, identity/secrets plan, deployment approach, one thing you'd explicitly *not* build.

<details><summary><b>Model answer (open after attempting)</b></summary>

- **Region:** an EU region with the needed SKUs (e.g. Sweden Central/West Europe); data residency satisfied by region choice; note paired region for backups.
- **Compute:** App Service Linux B/P-tier (small team, single web workload, slots) hosting API+UI; **Functions Flex Consumption** for OCR step (spiky, event-driven, pay-per-use) triggered via a queue — decouples the 3rd-party API's latency/failures from user requests.
- **Data:** Azure SQL **serverless** (month-end spikes, auto-pause off-hours saves real money at this scale); Blob for receipts (lifecycle to Cool@90d); no Cosmos (no scale/latency case — saying *no* is the point).
- **Messaging:** one Service Bus queue (Standard) `receipts-to-ocr`; MaxDeliveryCount+DLQ = the OCR retry story; consumer idempotent (receipt ID gate).
- **Identity:** Entra SSO via Microsoft.Identity.Web; user-assigned MI for app+function; roles on SQL (contained user), Blob, SB; third-party OCR API key in **Key Vault** (the one legitimate secret), KV reference in app settings.
- **Network (right-sized):** private endpoint for SQL + disable public access; App Service VNet integration for egress. Full hub-spoke is overkill at this size — name that trade-off. Front Door optional; at 500 internal users, App Service access restrictions + Entra auth may suffice — WAF if exposed beyond corp.
- **Deploy:** GitHub Actions OIDC, Bicep, staging slot + swap; EF migrations as pipeline step.
- **Not building:** AKS/microservices (3 devs, one domain), multi-region DR (agree RTO/RPO first — likely zone redundancy or even plain PITR restore meets it), Redis (no scale-out pressure yet; ARR affinity off + stateless anyway).

**Rubric (1 pt each, pass ≥8/10):** justified compute per workload shape · async decoupling of OCR · serverless SQL tied to usage pattern · zero secrets except vaulted 3rd-party key · SQL private + reasoning about *how much* network is enough · SSO named concretely · slots/zero-downtime · DLQ mentioned · at least one explicit "won't build" with reason · cost posture coherent (small team, spiky).
</details>

---

## Gate M2 — "Engineer" (after Module 08)

### B1. Recall (closed book, 20 min)
1. ACA revisions vs App Service slots — map the concepts; one thing each does the other can't.
2. Cosmos: why RU/s ÷ physical partitions matters; three properties of a good partition key; what `TransactionalBatch` requires.
3. Consistency levels: default, what Session guarantees, when you'd pay for Strong.
4. Service Bus peek-lock lifecycle with all four settle outcomes.
5. The outbox pattern: draw it, mark the crash windows, explain why each is safe.
6. Event vs message; SB vs EG vs EH sorting of: telemetry stream / blob-created / send-invoice command.
7. EF Core in cloud: retry mechanism, migration deployment discipline, context lifetime.

### B2. Design exercise (60 min)
**Scenario:** Consumer food-delivery platform, one metro area today → 10 cities in a year. Peaks 6–9 pm (20× baseline). Flows: order placement (must not lose orders), restaurant dashboard (live order feed), courier app (location pings every 5 s), customer notifications. Team: 8 devs, container-friendly. Investor pressure: demo multi-city scale story.

**Produce:** architecture (compute/data/messaging), the order-placement path in detail (reliability!), how location pings differ from orders, scale story, top-3 cost controls.

<details><summary><b>Model answer</b></summary>

- **Compute:** **Container Apps** environment (8 devs, several services, event-driven scale, revisions for canary): `orders-api`, `restaurant-api`, `courier-ingest`, `notifier`, `dispatch-worker`. KEDA on queue depth for workers; HTTP concurrency for APIs; scale-to-zero for off-peak workers.
- **Order path (the heart):** `orders-api` → SQL (order record) **+ outbox row in same transaction** → relay publishes `OrderPlaced` to Service Bus **topic** → subscriptions: `dispatch`, `notifications`, `restaurant-feed`. Consumers idempotent (orderId gate); DLQ monitored with runbook; **sessions** (`SessionId=orderId`) where per-order ordering matters (status transitions). Payment authorization as a saga step with compensation (void auth) — orchestrated (Durable or dispatch-worker owns it).
- **Location pings ≠ orders:** high-volume, loss-tolerant, time-series → **Event Hubs** (courier-ingest produces; consumers: live-map processor with checkpointing, cold path to Data Lake via capture). Explicitly *not* Service Bus — cost/semantics mismatch. Latest-position cache in **Redis** (key `courier:{id}`), TTL'd.
- **Data:** SQL (orders/financial — relational integrity, reporting); **Cosmos** for restaurant live feed + order-status projections (pk `/restaurantId`, change feed → SignalR/notifications), or Redis pub/sub + SignalR Service for the live dashboard — defend either; menus in Cosmos (`/restaurantId`) or SQL+cache. Multi-city = partition-friendly keys now (`cityId` in pk candidates), single region until latency data says otherwise — **multi-region is a data decision, not a compute decision**; say it.
- **Notifications:** `notifier` consumes topic; provider (ACS/push) keys in KV; retries + DLQ.
- **Scale story for investors:** load-test artifacts (KEDA scaling graph 1×→20×), per-city partition keys, stateless services, and the honest line: "region per continent later; city-sharded within region now."
- **Cost top-3:** scale-to-zero workers + off-peak floor for APIs; Event Hubs instead of per-message-priced brokers for pings (do the per-million math); reservations only after 3 months of telemetry (right-size first).

**Rubric (pass ≥9/12):** outbox present and placed correctly · idempotent consumers + DLQ story · topic/subscription fan-out (not point-to-point spaghetti) · EH vs SB justified for pings · Redis for hot position state · SQL-for-money vs NoSQL-for-feeds separation · partition keys designed for city growth · saga/compensation for payment · ACA scaling mechanics named (KEDA) · canary/revisions mentioned · honest multi-region posture · cost controls tied to traffic shape.
</details>

---

## Gate M3 — "Expert" (after Module 12)

### C1. The cold design exam (60 min, then the review)
**Scenario:** B2B ticketing SaaS ("Ticketo") selling to EU + US enterprise customers. Requirements: 99.95% contractual availability with credits; customer data must stay in customer's home geography (EU customers in EU, US in US); SSO with *customers'* identity providers; seasonal on-sale spikes 50×; 40-dev org (6 teams); auditors require: change history on tickets, quarterly DR proof, SOC2-friendly access control. Current pain: their v1 is a monolith on VMs with SQL auth passwords in config.

**Produce (one page):** target architecture + geography model, identity story (workforce + customer SSO + workloads), the 50× spike plan, the 99.95% math, DR + audit answers, migration approach from v1, monthly cost order-of-magnitude, five ADR titles.

<details><summary><b>Model answer sketch (compare, don't copy)</b></summary>

- **Geography = tenancy model first:** two *independent deployments* (EU stamp, US stamp — "deployment stamps" pattern), each: FD (global entry, geo-routing) → regional stamp. Data residency by *stamp assignment at tenant onboarding*, not by request routing. Global control plane (tenant directory) minimal, replicated.
- **Per stamp:** zone-redundant ACA (6 teams → service boundaries) or App Service per service; SQL BC/GP **ZR** + failover group to in-geo paired region (residency preserved — name that check); Cosmos (ticket events, `/tenantId_eventDate` hierarchical pk) with change feed → audit projections; SB Premium (ZR) spine, outbox everywhere; Redis; all private endpoints, hub-spoke per stamp.
- **Identity:** workforce = Entra + PIM + CA; **customer SSO = Entra External ID / per-tenant OIDC federation** to their IdPs; workloads = MIs everywhere — v1's passwords die in migration step 1. RBAC via groups; access reviews → the SOC2 answer, plus Policy + Defender for Cloud evidence.
- **50× spikes:** the on-sale path is special: queue-based **load leveling** (buy requests → SB → ordered processors with sessions per event), waiting-room pattern at FD if extreme; autoscale rehearsed with load tests; capacity: pre-scale rules on schedule (on-sales are known!) beats reactive.
- **99.95% math:** chain FD 99.99 × ACA(ZR) 99.95 × SQL(BC-ZR) 99.995 × SB 99.95 ≈ 99.89 — **doesn't meet 99.95 serially** → mitigations: reduce synchronous chain (queue between web and processing), regional failover for the stamp, and/or contractual scope (availability defined at ticket-purchase API). Showing the math *and* the gap is the expert move.
- **Audit:** ticket change history = Cosmos change feed → immutable store (Blob with immutability policy / Ledger tables option); quarterly DR = scripted failover-group drill + evidence from monitor dashboards (lab 12 pattern); access = PIM reports + activity logs to workspace with retention.
- **Migration:** strangler off the monolith — land VMs as-is (rehost) into the landing zone if timeline demands, then peel services (auth first: passwords→MI/KV; then the spikey on-sale path to queues). 6 R's language.
- **Cost order:** two stamps ≈ 2× infra + FD + Premium SKUs — thousands/month scale; reservations on stamp baselines; the spike plan is consumption-priced. Precision isn't expected; *structure* is.
- **ADRs:** stamp-per-geography; outbox as standard; ACA over AKS (or reverse, argued); SQL for transactions + Cosmos for events split; External ID federation for customer SSO.

**Rubric (pass ≥10/13):** stamps/residency by tenancy · composite SLA computed *and* gap addressed · queue-leveling for spikes · scheduled pre-scale insight · customer-IdP federation distinct from workforce · MI/KV kills passwords in step 1 · failover groups in-geo · audit mechanisms concrete · strangler migration with first slice named · DR as rehearsed drill with evidence · private-by-default assumed · cost structured by stamp · ADRs are real decisions, not headings.
</details>

### C2. The mock design review (do this with a peer or an AI playing hostile reviewer)
Defend your C1 answer against: "Why not one global deployment with geo-replication?" · "Why not AKS — we might need it?" · "The SLA math shows 99.89 — you're breaching contract?" · "Cosmos is expensive — why not just SQL?" · "Can't we skip the queue and scale the API?" Hold your ground where right; concede with grace where they are. Being *changeable by evidence but not by pressure* is the final expert behavior this curriculum can teach.

---

## Scoring yourself honestly
- **Gate not passed** → redo the relevant module's lab variant, not just re-read. Reading feels like progress; rebuilding is progress.
- **Passed M3** → you're ready for AZ-305 prep as consolidation, real design ownership at work, and the monthly maintenance loop in the [index](index.html).
