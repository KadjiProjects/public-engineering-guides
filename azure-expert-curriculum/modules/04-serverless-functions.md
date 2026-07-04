# Module 04 — Azure Functions & Serverless

> **Week 4.** Functions replace the Windows services, cron jobs, and queue workers of your on-prem life — with per-execution billing and event-driven scale. The .NET story changed decisively: **isolated worker model only** (in-process support ends 2026-11-10) and **Flex Consumption** as the recommended serverless plan. Learn the current world, not the tutorials of 2021.

---

## 1. The programming model (isolated worker — the only .NET path forward)

Your functions run in a **separate .NET worker process** (a console app you own) that the Functions host talks to over gRPC. Consequences that matter:

- **It's a normal .NET app.** Full DI, middleware, your target framework (.NET 8/9/10, even .NET Framework 4.8 for legacy), no host-imposed package versions.
- `Program.cs` looks like any modern .NET app:

```csharp
var builder = FunctionsApplication.CreateBuilder(args);
builder.ConfigureFunctionsWebApplication();
builder.Services.AddSingleton<TokenCredential>(new DefaultAzureCredential());
builder.Services.AddHttpClient<CatalogClient>();
builder.Services.AddApplicationInsightsTelemetryWorkerService();
builder.ConfigureFunctionsApplicationInsights();
builder.Build().Run();
```

- Functions are methods with attributes:

```csharp
public class OrderFunctions(ILogger<OrderFunctions> log, OrderService orders)
{
    [Function(nameof(ProcessOrder))]
    public async Task ProcessOrder(
        [ServiceBusTrigger("orders", Connection = "SbConn")] OrderMessage msg,
        FunctionContext ctx)
    {
        await orders.HandleAsync(msg);   // your normal, testable service layer
    }

    [Function(nameof(GetOrder))]
    public async Task<IActionResult> GetOrder(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "orders/{id}")] HttpRequest req,
        string id) => new OkObjectResult(await orders.GetAsync(id));
}
```

- **Middleware** (isolated-only feature): cross-cutting concerns (correlation, exception shaping) via `IFunctionsWorkerMiddleware`.
- **Connection settings** like `SbConn` above are *app-setting prefixes*, and support **identity-based connections**: instead of a connection string, set `SbConn__fullyQualifiedNamespace = mybus.servicebus.windows.net` and grant the MI a role. No secrets — consistent with module 2.

## 2. Triggers & bindings — and when to skip bindings

- **Trigger** = what starts the function (HTTP, timer, Service Bus, Event Grid, Event Hubs, Blob, Cosmos change feed, Durable orchestration...). Exactly one per function.
- **Input/output bindings** = declarative data plumbing. Convenient, but the expert guidance (Microsoft's own): for anything non-trivial **use the service SDKs directly via DI** — better error handling, batching, options, and testability. Bindings shine for simple glue.
- **Retry & poison handling is trigger-specific and you must know it per trigger:** Service Bus → delivery count → **dead-letter queue** (module 8); Event Hubs → checkpointing semantics, your code must be idempotent; HTTP → no retries (caller's problem); timer/blob → host retry options. *Universal rule: every function must be **idempotent** — at-least-once delivery is the baseline assumption everywhere.*

## 3. Hosting plans — the 2026 decision table

| Plan | Model | Scale | Cold start | VNet | Pick when |
|---|---|---|---|---|---|
| **Flex Consumption** ⭐ | Serverless, per-execution + optional always-ready instances | To ~1000 instances, per-function scaling, concurrency-based | Mitigated via always-ready | **Yes** | **Default for new serverless .NET.** Linux, isolated-only, .NET 8/9/10; memory sizes 512/2048/4096 MB |
| Consumption (classic) | Serverless | To 200 | Real | No | Legacy/simple apps; Windows-only needs; being superseded by Flex |
| Premium (Elastic) | Pre-warmed workers, always ≥1 | Fast elastic | None (pre-warmed) | Yes | Constant traffic + serverless bursts; long executions; custom images |
| Dedicated (App Service plan) | Runs on your existing plan | Plan autoscale | None (Always On) | Yes | You already own plan capacity; predictable steady load |
| Container Apps hosting | Functions runtime in ACA | KEDA | Depends | Yes | You're consolidating everything on Container Apps (module 5) |

**Clock facts (verified Jul 2026):** in-process model support **ends 2026-11-10** — migrate anything `dotnet` (in-process) to `dotnet-isolated`; Functions runtime **1.x ends 2026-09-14**; .NET 10 on Linux *classic* Consumption isn't supported — use Flex.

**Cold start engineering:** always-ready instances (Flex) / pre-warmed (Premium); trim dependencies; ReadyToRun compilation; avoid heavy static initialization; keep the credential/service clients singleton (they're expensive).

## 4. Durable Functions — stateful workflows on serverless

For multi-step, long-running, or fan-out work, Durable adds **orchestrations** (deterministic workflow code), **activities** (the actual work), **entities** (tiny actors), and **timers/external events** — state persisted via event sourcing in storage.

```csharp
[Function(nameof(FulfillOrder))]
public async Task FulfillOrder([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var order = ctx.GetInput<Order>();
    await ctx.CallActivityAsync(nameof(ReserveStock), order);
    // Fan-out/fan-in:
    var tasks = order.Lines.Select(l => ctx.CallActivityAsync<Shipment>(nameof(ShipLine), l));
    var shipments = await Task.WhenAll(tasks);
    // Human interaction with timeout:
    using var cts = new CancellationTokenSource();
    var approval = ctx.WaitForExternalEvent<bool>("ManagerApproval");
    var timeout  = ctx.CreateTimer(ctx.CurrentUtcDateTime.AddHours(24), cts.Token);
    if (await Task.WhenAny(approval, timeout) == approval && approval.Result) { /* ... */ }
}
```

**The iron law:** orchestrator code must be **deterministic** (no `DateTime.Now`, no `Guid.NewGuid()`, no direct I/O — use `ctx.CurrentUtcDateTime`, activities for I/O) because it *replays* from history. Patterns to name in design reviews: function chaining, fan-out/fan-in, async HTTP status polling, monitor, human interaction, saga-style compensation (module 8/12).

## 5. Operations & configuration

- `host.json` = host behavior (per-trigger concurrency, retry, logging sampling). `local.settings.json` = local-only settings (never deployed, never committed with secrets).
- **Deploy exactly like App Service** (module 3): zip / run-from-package; slots exist on some plans (not Flex — use versioned deployments/traffic patterns instead; check current story).
- **Scale controller** decisions are visible in logs; per-function scaling on Flex means one hot function doesn't force-scale the whole app.
- **Monitoring:** App Insights integration is first-class; watch *invocation count, failure rate, duration percentiles, instance count*; distributed traces connect HTTP → queue → function (module 10).
- **Security:** HTTP triggers behind Front Door/APIM in prod; function keys are weak auth (use Entra via Easy Auth or APIM); Flex + VNet integration covers private egress.

## 6. Decision framework

**Functions vs App Service vs Container Apps:** unit of work is an *event/short task* → Functions. It's a *web app/API with steady traffic* → App Service (or ACA). It's *many services + custom runtimes/sidecars* → ACA. **Within Functions:** new app → Flex Consumption; steady heavy load or special images → Premium; workflow state → Durable; ultra-simple queue glue where you already run ACA → Functions-on-ACA.

**Anti-patterns to name:** long-synchronous HTTP work in a function (use async status / Durable); functions calling functions over HTTP in a chain (use queues or Durable); shared mutable static state; assuming exactly-once delivery; giant "god function" apps (group by scaling + deployment lifecycle).

## 7. Lab 04 (~3 h)

> [labs.md → Lab 04](../labs.md): the capstone gains an async worker path.

1. Scaffold isolated .NET Functions app (`func init --worker-runtime dotnet-isolated`); run locally with Azurite + local Service Bus emulator or a dev namespace.
2. HTTP trigger enqueues to Service Bus (SDK via DI, not output binding — feel the difference); Service Bus trigger processes with **identity-based connection** (no connection string anywhere).
3. Deploy to **Flex Consumption** via Bicep + `az functionapp deployment`; set one always-ready instance; measure cold vs warm latency with and without it.
4. Poison-message drill: throw on a specific message; watch delivery count climb; find it in the DLQ; write the one-page runbook entry for "message in DLQ".
5. Durable mini-saga: orchestration with two activities + a compensation on failure; kill the host mid-run; restart; watch replay complete it.

## 8. Self-check

1. Why did the isolated model win, and name two capabilities it has that in-process never had.
2. What happens on 2026-11-10, and what's the migration path for a `dotnet` (in-process) app?
3. Flex vs Premium: three deciding factors.
4. What guarantees does a Service Bus trigger give (delivery count, DLQ), and what does that force on your code?
5. Why can't an orchestrator call `HttpClient` directly? What breaks, mechanically?
6. Identity-based connections: what replaces the connection string app setting for Service Bus, exactly?
7. When is an output binding the *wrong* choice, per Microsoft's own guidance?

## 9. Explain out loud

- Explain replay-based orchestration to someone who knows async/await but not event sourcing.
- Defend "every handler must be idempotent" with a concrete duplicate-delivery scenario.

## 10. Verify before you build

- Flex Consumption regional availability, memory sizes, and current limits (evolving fast).
- Deployment/slots story on Flex (changed vs classic plans).
- Durable Functions: current backend options (Azure Storage vs Netherite vs MSSQL vs durable task scheduler service).

*Next: [Module 05 — Containers: ACR → Container Apps → AKS](05-containers.md).*
