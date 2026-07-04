# Module 12 — Architecture, Reliability & Cost (Capstone)

> **Week 12.** Everything converges. This module trains the expert's actual job: take a business problem cold, produce a defensible design across the Well-Architected pillars, do the SLA and cost math out loud, and know the reference architectures well enough to adapt rather than invent. Finish with the assessment's part C — the mock design review.

---

## 1. Well-Architected Framework — as a practice, not a poster

Five pillars: **Reliability, Security, Cost Optimization, Operational Excellence, Performance Efficiency.** The expert moves:

- Use it as a **review protocol**: for each pillar, ask "what did we decide, what did we trade away, how do we know?" Every module in this curriculum answered one slice (2=Security, 10=OpsEx, 6–8=Perf/Reliability of data paths, 11=OpsEx, this one=the math).
- **Trade-offs are the content.** Pillars conflict by design (BC SQL tier: +reliability/perf, −cost). A design note that names the losing pillar and why is worth ten diagrams.
- The **Azure Advisor** + WAF assessment tooling exists — run it against real workloads for a prioritized punch list.
- Sibling for org-scale: **CAF** (Strategy→Plan→Ready→Adopt→Govern→Manage) and **landing zones** = the governed foundation (management-group tree, policy baseline, connectivity hub, identity) workloads deploy into. WAF = one workload; CAF = the estate. Name the right altitude.

## 2. Reliability engineering — the math and the patterns

**The two numbers first:** RTO (max downtime) and RPO (max data loss) — *the business sets them*; every DR design is derived from them. Asking for RTO/RPO before proposing anything is the tell of a professional.

**Composite SLA math (do it live):** serial dependencies multiply: FD 99.99% × App Service 99.95% × SQL 99.99% ≈ **99.93%** (~6 h/year). Parallel redundancy: two independent 99.9% paths ≈ 1−(0.001²) = 99.9999% *for that hop*. Rules: chain multiplies, redundancy compensates, and your **SLO must be ≤ the composite SLA of the chain** — promising 99.95% on a 99.9% chain is fiction. Also: SLA = financial credit, not a physics guarantee; zone-redundant SKUs typically carry the higher SLAs.

**The resilience ladder (cost ↑):**
1. **Single region, zone-redundant everything** (App Service ZR ≥3 instances, SQL ZR, ZRS storage, SB Premium ZR) — survives datacenter loss; the right default for most.
2. **Multi-region active-passive:** paired region, SQL failover group (one listener endpoint), GZRS/RA-GZRS storage, FD with priority routing + health probes; *rehearsed* failover runbook. RTO minutes–hour, honest and affordable.
3. **Multi-region active-active:** both live behind FD; needs data strategy (Cosmos multi-region, or partitioned/tenant-homed writes), idempotency everywhere, conflict thinking. Expensive in engineering, not just compute — justify with real RTO/RPO.

**Code-level patterns (name + place them):** retry w/ backoff + jitter (only idempotent ops), circuit breaker, timeout budgets end-to-end, bulkhead isolation, load leveling via queues (module 8), graceful degradation (feature-flagged fallbacks, module 11), health checks (liveness vs readiness semantics). In .NET: `Microsoft.Extensions.Resilience` standard pipelines — Aspire ServiceDefaults wires them (module 5).

**Prove it:** backup/restore *tested* (a backup you've never restored is a hope), failover drills scheduled, **Azure Chaos Studio** experiments (kill zone, inject SQL latency) against pre-prod with golden signals watching (module 10). "We test failure on purpose" is expert-tier maturity.

## 3. Cost engineering (FinOps) — the five levers

1. **Right-size from telemetry** — Advisor + your P95s, not gut feeling. Downsize is a deploy, not a project (IaC, module 11).
2. **Commit on the baseline:** Reservations (specific SKU/region) and **Savings Plans** (flexible compute $/hr commit) for steady load — the single biggest %-win on compute.
3. **Consumption where spiky:** serverless SQL, Flex Consumption, ACA scale-to-zero — pay for demand, not capacity (modules 4–6).
4. **Azure Hybrid Benefit** — carry Windows/SQL licenses; routinely halves lift-and-shift compute cost estimates.
5. **Waste patrol:** orphaned disks/IPs, over-retained logs (module 10's `Usage` query), dev running nights/weekends (auto-shutdown), storage without lifecycle rules, cross-**AZ/region egress** surprises.

**Practice:** tags → cost allocation (module 1); budgets + alerts per subscription; anomaly alerts on; cost review as a monthly engineering ritual with a named owner. Estimate *before* building (Pricing Calculator) and put the number in the design doc — architecture without a monthly € figure is a sketch.

## 4. Reference architectures — carry three in your head

**A. The zone-redundant web workload (your default answer):**
Front Door Premium (WAF, caching) → App Service P1v3 ZR ≥3 (staging slot) → SQL GP ZR via private endpoint ← Key Vault + user-assigned MI; Service Bus Premium → Functions Flex worker; Redis for session/cache; everything private (hub-spoke, module 9), OTel→App Insights + golden alerts, Bicep + OIDC pipeline, reservations on plan+SQL. *You built exactly this across the labs — defend it pillar by pillar.*

**B. Event-driven microservices:** ACA environment (internal) + FD/AppGW ingress; services with KEDA scaling; Service Bus topics as the spine; outbox pattern at every write→publish seam; Cosmos for high-scale entities with change-feed projections; saga orchestrations (Durable or a coordinator service) for long flows.

**C. The migration landing (Migration Knowledge Map's world):** landing zone first; 6-R'd portfolio; MI (Managed Instance) for legacy SQL, App Service for web tiers, Functions for jobs; hybrid ER/VPN during transition; modernize incrementally toward A/B.

For anything else: **Azure Architecture Center** patterns catalog + baseline architectures — experts *adapt* documented baselines and cite them, they don't freehand.

## 5. The design-review method (rehearse until automatic)

1. **Requirements first:** users/scale, latency, data classification/residency, RTO/RPO, budget, team skills. Write them down — half the value is exposing the unknowns.
2. **Constraints → candidate shape** (usually variant of A/B/C above).
3. **Walk the five pillars**, naming trade-offs and mitigations.
4. **Close the loop on the five checklist items** (from the Cram Sheet): identity, network, reliability, observability, cost — each with a concrete mechanism, not an adjective.
5. **State the SLA math and the monthly cost.**
6. **Name what you'd verify on Learn before committing** (limits, SKUs, previews) — calibrated uncertainty is senior behavior, bluffing is not.

## 6. Lab 12 — the capstone exam (give it a full day)

> [labs.md → Lab 12](../labs.md) has the full scenario pack.

1. Take scenario C1 from [assessment.md](../assessment.md) (global ticketing SaaS) **cold**: 60 minutes, produce the one-page design (diagram + pillar notes + SLA math + cost estimate).
2. Compare against the model answer; score yourself with its rubric; write down every gap.
3. **Harden your real capstone:** add the missing reliability pieces to CloudBoard — SQL failover group to the paired region, FD priority routing, chaos experiment (kill the primary region's app) — and measure actual RTO with your module-10 dashboard.
4. Run the WAF assessment tool against CloudBoard; fix the top three findings; keep the report.
5. Write the **architecture decision record (ADR)** set for CloudBoard: five ADRs, one per pillar-defining choice. This artifact is interview gold.

## 7. Self-check

1. Compute the composite SLA: FD 99.99 × ACA 99.95 × Cosmos 99.999 × SB 99.9 — and state the max SLO you'd sign.
2. Zone redundancy vs multi-region: which failure classes, which cost classes, which data implications?
3. Your RTO is 4 h, RPO 15 min, budget modest: pick the resilience-ladder rung and the exact SQL/storage/routing features.
4. Five cost levers — one sentence and one Azure feature each.
5. Why is a restore-never-tested backup a risk register item, not a control?
6. What makes active-active *engineering*-expensive beyond the second region's bill?
7. Recite reference architecture A with every module's contribution labeled.

## 8. Explain out loud

- Deliver the scenario-C1 design as a 10-minute whiteboard talk to a skeptical CTO (record yourself; cringe; repeat).
- Explain error budgets and SLA math to a sales team that wants "100% uptime" in the contract.

## 9. Verify before you build

- Current SLA figures for your exact SKUs (they change; ZR variants differ).
- Regional pairing status (some newer regions are non-paired — affects GRS strategy).
- Savings Plan vs Reservation coverage rules for the services in your design.

---

**You're done with the path — now the loop:** monthly Concepts-Map re-read, flashcards weekly, one Architecture Center baseline studied per month, Azure Updates skimmed, and the capstone kept alive as your living portfolio. Book AZ-204 now; AZ-305 after four more design exercises. Expertise from here is reps.
