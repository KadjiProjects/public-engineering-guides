# Module 10 — Observability

> **Week 10.** "You can't promise an SLA you can't measure." This module wires modern .NET telemetry (OpenTelemetry) into Azure Monitor, teaches you enough KQL to be dangerous, and turns raw signals into alerts, dashboards, and SLOs a team can operate on.

---

## 1. The map: what collects what

- **Azure Monitor** = the umbrella: **Metrics** (numeric time series, near-real-time, cheap) + **Logs** (rich records in **Log Analytics workspaces**, queried with KQL) + alerts, workbooks, dashboards.
- **Application Insights** = the APM personality of Monitor for *your code*: requests, dependencies, exceptions, custom events/metrics, **distributed traces**, Live Metrics, Application Map. Workspace-based (its data lives in Log Analytics — one query surface).
- **Platform telemetry:** every resource emits metrics for free; **diagnostic settings** route resource *logs* (SQL audit, Key Vault access, Front Door logs...) to the workspace — turning these on is step one of any real deployment (and a Bicep loop, module 11).
- **Infra flavors:** VM Insights, Container Insights + **managed Prometheus/Grafana** for AKS/ACA estates.
- Cost reality: **log ingestion is the bill driver.** Levers: sampling (App Insights), table plans (Analytics vs Basic logs), retention settings, and *not logging garbage*. An expert designs telemetry volume like any other capacity.

## 2. OpenTelemetry in .NET — the current story

OTel is the vendor-neutral standard (traces/metrics/logs) built on .NET's native `Activity`/`Meter` APIs. The Azure story: **the Azure Monitor OpenTelemetry Distro** (`Azure.Monitor.OpenTelemetry.AspNetCore`) — OTel with an Application Insights exporter, preconfigured:

```csharp
builder.Services.AddOpenTelemetry().UseAzureMonitor();   // conn string from config/env
// Enrich with your own signals:
builder.Services.ConfigureOpenTelemetryTracerProvider((sp, tp) =>
    tp.AddSource("CloudBoard.*"));
builder.Services.ConfigureOpenTelemetryMeterProvider((sp, mp) =>
    mp.AddMeter("CloudBoard.Orders"));
```

```csharp
// Custom spans + metrics with pure BCL types (no vendor lock):
private static readonly ActivitySource Activity = new("CloudBoard.Orders");
private static readonly Meter Meter = new("CloudBoard.Orders");
private static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders_placed");

using var span = Activity.StartActivity("PlaceOrder");
span?.SetTag("order.tenant", tenantId);
OrdersPlaced.Add(1, new KeyValuePair<string, object?>("tenant", tenantId));
```

- **Correlation is automatic** via W3C `traceparent`: HTTP calls, SqlClient, Azure SDKs, and Service Bus message hops all propagate context → the **end-to-end transaction view** shows browser→API→queue→worker→SQL as one trace. This is the payoff; verify it in the lab.
- **Aspire's ServiceDefaults** (module 5) wires exactly this for every service — reuse it.
- Classic App Insights SDK vs OTel distro: new work → **OTel distro**; know the classic SDK exists in older codebases (TelemetryClient, ITelemetryInitializer) for maintenance. Structured logging via `ILogger` flows into `traces`/`AppTraces` either way — log *structured properties*, not interpolated strings, so KQL can filter on them.
- **Sampling:** keep percentage sampling on in high-volume prod; custom metrics stay accurate, traces become representative — understand that trade before debugging "missing" requests.

## 3. KQL — the 80/20 working set

Tables you'll live in (workspace names): `AppRequests`, `AppDependencies`, `AppExceptions`, `AppTraces`, plus resource tables (`AzureDiagnostics` or resource-specific) and `Usage` (your ingestion bill).

```kusto
// 1. Failure rate by operation, last hour
AppRequests
| where TimeGenerated > ago(1h)
| summarize total = count(), failed = countif(Success == false) by OperationName
| extend failurePct = round(100.0 * failed / total, 2)
| order by failurePct desc

// 2. P50/P95/P99 latency trend
AppRequests
| where TimeGenerated > ago(24h)
| summarize p50=percentile(DurationMs,50), p95=percentile(DurationMs,95), p99=percentile(DurationMs,99)
          by bin(TimeGenerated, 15m)
| render timechart

// 3. What is my slow request actually waiting on? (join to dependencies)
AppDependencies
| where TimeGenerated > ago(1h) and OperationId in (
    (AppRequests | where DurationMs > 2000 | project OperationId))
| summarize avg(DurationMs), count() by DependencyType, Target
| order by avg_DurationMs desc

// 4. Exceptions clustered by problem
AppExceptions
| where TimeGenerated > ago(24h)
| summarize count(), any(OuterMessage) by ProblemId
| order by count_ desc

// 5. Who's eating the ingestion budget?
Usage
| where TimeGenerated > ago(30d)
| summarize GB = sum(Quantity) / 1000 by DataType
| order by GB desc
```

Learn these operators cold: `where`, `summarize` (+`count/countif/percentile/avg/dcount`), `bin`, `extend/project`, `order by`, `top`, `join kind=inner`, `render`, `parse_json`/`todynamic` for custom dimensions. That set answers 90% of production questions; everything else you look up.

## 4. From signals to operations

- **Alert anatomy:** signal (metric / log query / activity log) → condition → **action group** (email/Teams/SMS/webhook/Function/Logic App) → optional **alert processing rules** (suppression windows, routing).
- **Metric alerts** = fast/cheap (availability, 5xx rate, CPU, DB DTU, DLQ depth via SB metrics); **log alerts** = expressive (that KQL failure-rate query, scheduled) but minutes of latency and per-evaluation cost. Choose accordingly.
- **Golden-signal starter pack per service:** availability (synthetic **availability tests** hitting `/healthz` from multiple regions), request failure rate, P95 latency, dependency failure rate, DLQ depth, DB saturation, and *absence-of-telemetry* (heartbeat) alerts.
- **SLO thinking:** SLI = measured indicator (e.g. % requests < 500 ms and 2xx); SLO = target (99.5% monthly); **error budget** = 1 − SLO — alert on *budget burn rate* (fast-burn page, slow-burn ticket), not on every blip. This vocabulary instantly elevates design reviews.
- **Dashboards & workbooks:** workbook per workload = golden signals + drill-down queries; pin the exec view to a shared dashboard. **Availability tests + Application Map + Live Metrics** are the demo trio during incidents.
- **Smart detection / anomaly features** exist — treat as hints, not substitutes for explicit SLO alerts.

## 5. Decision framework

One workspace per environment (or per landing zone) unless data sovereignty says otherwise; App Insights workspace-based, one per app per env; diagnostic settings on *everything* to that workspace; OTel distro in code; sampling tuned; 5–8 golden alerts per workload wired to a real on-call channel; ingestion reviewed monthly (`Usage` query); SLOs written down with the business, error-budget alerts implemented. "We have App Insights" ≠ observability — the artifact is *alerts + dashboard + runbook links*.

## 6. Lab 10 (~3 h)

> [labs.md → Lab 10](../labs.md)

1. Add the OTel distro to API + worker (or lean on Aspire ServiceDefaults); deploy; confirm **one trace** spans HTTP→Service Bus→worker→SQL in the transaction view.
2. Add a custom `ActivitySource` span + a business `Counter` (orders_placed); find both in KQL (`AppDependencies`/`AppMetrics` or `customMetrics`).
3. Write and save the five KQL queries above against your own traffic (generate load incl. some failures).
4. Alerts: metric alert on 5xx-rate, log alert on the failure-rate query, availability test on `/healthz` from 3 regions → action group to your email; trigger each deliberately (break the app briefly).
5. Build the golden-signals **workbook**; simulate an incident (kill Redis from lab 06) and walk your own dashboard→trace→root cause path; time it.
6. Check `Usage`, turn sampling to 50%, observe the effect on volume and on your queries' counts (learn to read sampled data).

## 7. Self-check

1. Metrics vs logs: storage model, latency, cost — and which alert type each feeds.
2. What exactly propagates in `traceparent`, and name three .NET libraries that do it automatically.
3. Why prefer the OTel distro over the classic SDK for new services? Two reasons.
4. Your P95 is fine but one customer complains — which two views/queries find their specific experience?
5. Sampling is on at 25%: what do request *counts* in KQL mean now, and how do you correct (`_ItemCount`/itemCount)?
6. Define SLI/SLO/error budget for a checkout API, and the two burn-rate alerts you'd set.
7. Top three drivers of a surprise Log Analytics bill and the lever for each.

## 8. Explain out loud

- Walk an incident: alert fires → dashboard → transaction trace → dependency at fault → mitigation → the follow-up alert you add.
- Explain to a PM why you page on error-budget burn instead of every 5xx.

## 9. Verify before you build

- OTel distro current package/features (logs signal maturity, sampling options).
- Table plans (Basic/Auxiliary logs) pricing tiers and retention defaults.
- Availability tests current flavor (standard tests) and regional coverage.

*Next: [Module 11 — IaC & delivery](11-iac-devops.md).*
