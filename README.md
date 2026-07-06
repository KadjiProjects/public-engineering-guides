# KadjiProjects — public engineering guides

Three independent, self-contained, fully generic resource sets. Nothing in this repository
references or depends on any private codebase.

## 📁 [catalog-api-net10-upgrade/](catalog-api-net10-upgrade/)

**A two-part, LLM-executable upgrade playbook for a plugin-based .NET 6 API.** Part 1
migrates the platform to .NET 10: verified package matrix, the four known breakages
(including the runtime-only SQL `Encrypt=true` trap), and batch strategies for upgrading
hundreds of runtime-loaded filter/subscriber plugin modules. Part 2 optionally replaces
AutoMapper with [Mapwright](https://github.com/lodestar-labs/Mapwright), the compile-time
source-generated mapper — including the delegate-injection pattern for generic
repository/processor pipelines.

- `part1/` and `part2/` — phase files with hard GATEs, written for step-by-step execution
  by a careful developer or a less-capable LLM; discovery-driven, so copies with more
  entities or modules follow the identical process.
- `challenges-explained.html` — the full story for humans, junior-developer friendly.
- `troubleshooting.md` — every known error mapped to cause and fix.

Start at [catalog-api-net10-upgrade/README.md](catalog-api-net10-upgrade/README.md).

## 📁 [explicit-mapping/](explicit-mapping/)

**A dependency-free AutoMapper replacement (the `ExplicitMapping` class library) plus a
complete, LLM-executable migration guide** for removing AutoMapper from any .NET project,
on any target framework.

- `src/ExplicitMapping/` — the library: 7 files, zero package dependencies, compile-time-safe
  mapping contracts + a test-time coverage verifier (`MappingVerifier`, the
  `AssertConfigurationIsValid()` replacement).
- `tests/ExplicitMapping.Tests/` — 8 NUnit tests; `dotnet test ExplicitMapping.sln` → 8/8.
- `guide/` — the migration guide in three editions: multi-file markdown (GATE-per-phase,
  for step-by-step execution), single-file markdown (for one-shot LLM consumption), and a
  styled single-page HTML (for humans). Covers the entire AutoMapper feature surface with
  a 16-pattern catalog, call-site rewriting, special cases, verification, and troubleshooting.

Start at [explicit-mapping/README.md](explicit-mapping/README.md).

## 📁 [azure-expert-curriculum/](azure-expert-curriculum/)

**A complete 12-week curriculum taking an experienced .NET developer to expert-level
Azure skills** — developing, deploying, and configuring Azure services. Verified against
Microsoft Learn (July 2026).

- `index.html` — start here: the learning method (active recall, spaced repetition,
  deliberate practice), the 12-module path, milestone gates, certification mapping.
- `modules/01–12` — deep modules: foundations/ARM, identity & passwordless auth
  (`DefaultAzureCredential`), App Service, Functions (isolated worker, Flex Consumption),
  containers (ACR/Container Apps/AKS/Aspire), Azure SQL + EF Core + Redis, Cosmos DB &
  storage, messaging (Service Bus/Event Grid/Event Hubs, outbox, sagas), networking
  (private endpoints & DNS, hub-spoke), observability (OpenTelemetry + KQL), IaC & delivery
  (Bicep, GitHub OIDC), architecture/reliability/cost. Each: concepts → .NET code →
  decision framework → lab → self-check.
- `labs.md` — one capstone application built progressively across all modules, with cost guards.
- `assessment.md` — three milestone gates with design exercises, model answers, and rubrics.
- Four companion study documents: an interactive flashcard app (84 cards), a concepts map,
  an interview cram sheet, and a migration knowledge map.

Start at [azure-expert-curriculum/index.html](azure-expert-curriculum/index.html).

## License

MIT — see [LICENSE](LICENSE).
