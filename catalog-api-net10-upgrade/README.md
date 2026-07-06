# Catalog API — .NET 6 → .NET 10 upgrade + optional Mapwright migration

An LLM-executable upgrade playbook for a **plugin-based .NET 6 REST API** (layered
architecture: Domain / Application / Persistence / Processing / Service, with
filter and subscriber **plugin modules loaded at runtime from disk** — potentially
hundreds of them). Every step in this guide was verified by performing the upgrade
on a real production system of this exact shape.

The work is split in **two independent parts**:

| Part | What | Risk | Optional? |
|---|---|---|---|
| **Part 1** | Migrate everything from .NET 6 to .NET 10, keeping AutoMapper in place | Low–medium (mechanical, but two runtime-only failure modes) | No — .NET 6 is out of support |
| **Part 2** | Replace AutoMapper with [Mapwright](https://github.com/lodestar-labs/Mapwright), a compile-time source-generated mapper | Low (the compiler verifies the result) | Yes — do it after Part 1 is green, or never |

**Humans start here:** [challenges-explained.html](challenges-explained.html) — a
full plain-language explanation of every challenge in this migration, written for a
junior developer.

**LLMs start here:** [part1/00-ground-rules.md](part1/00-ground-rules.md) and follow
the files in order. Do not skip a GATE.

## The key areas this migration must address

1. **Framework retarget across every project** — `net6.0` → `net10.0` (a 4-version
   jump: 6 → 7 → 8 → 9 → 10 in one hop). Mechanical, batchable with one script.
2. **Package upgrades with breaking API changes** — EF Core 6 → 10,
   Microsoft.Extensions 6 → 10, Serilog 2.x → 4.x and all its sinks, Swashbuckle
   6 → 10. Two of these break code, not just versions (areas 4 and 5).
3. **SQL connection strings — runtime-only failure.** EF Core 10's
   `Microsoft.Data.SqlClient` defaults to `Encrypt=true`. Every connection string
   without an explicit `Encrypt=` or `TrustServerCertificate=` **compiles clean and
   fails at runtime** with `SqlException`. This is the most dangerous single item in
   the whole migration.
4. **The Serilog email sink rewrite.** `Serilog.Sinks.Email` 3.0+ deleted the
   `EmailConnectionInfo` API. Any wrapper built on it must be rewritten against
   `EmailSinkOptions` (full replacement source provided).
5. **The Swashbuckle/OpenAPI namespace move.** Swashbuckle 10 depends on
   Microsoft.OpenApi 2.x, where `OpenApiInfo` moved from
   `Microsoft.OpenApi.Models` to `Microsoft.OpenApi`.
6. **Plugin modules at scale.** Filter and subscriber modules are separate
   assemblies loaded at runtime with `Assembly.LoadFile` from per-module folders.
   Every one must be retargeted, rebuilt against the upgraded contracts, and
   redeployed — a .NET 6-compiled module must not be loaded into the .NET 10 host
   with stale contract DLLs in its folder. With hundreds of modules this is a
   batch/scripting problem: [part1/04-plugin-modules-batch.md](part1/04-plugin-modules-batch.md).
7. **(Part 2) The AutoMapper surface.** One static `MapperFactory` holding three
   maps per entity (domain → entity with audit/navigation ignores, entity → domain,
   entity self-map used for both cloning and in-place EF-tracked updates), validated
   by `AssertConfigurationIsValid()` at runtime.
8. **(Part 2) The generic pipeline seam.** `Repository<TDbModel>` and
   `ActionProcessor<TDataModel, TDbModel>` call `IMapper` generically. Mapwright's
   mappings are per-pair static methods, so the pipeline receives them as injected
   delegates — a small, mechanical rewiring, described with the exact before/after.

## Folder layout

```
catalog-api-net10-upgrade/
├── README.md                        ← you are here
├── challenges-explained.html        ← for humans: the full story, junior-dev friendly
├── troubleshooting.md               ← error → cause → fix, both parts
├── part1/
│   ├── 00-ground-rules.md           ← hard rules, variance handling, prerequisites
│   ├── 01-discovery.md              ← build the work lists (GATE 0)
│   ├── 02-retarget-and-packages.md  ← TFM swap + version matrix (GATE 1)
│   ├── 03-known-breakages.md        ← conn strings, email sink, Swashbuckle, nullability (GATE 2)
│   ├── 04-plugin-modules-batch.md   ← batch-upgrading hundreds of filter/subscriber modules (GATE 3)
│   └── 05-verification.md           ← build, tests, smoke, acceptance (GATE 4)
└── part2/
    ├── 06-install-mapwright.md      ← packages + how Mapwright works (GATE 5)
    ├── 07-declare-mappers.md        ← derive [Mapper] declarations from the AutoMapper profile (GATE 6)
    ├── 08-rewire-pipeline.md        ← delegate injection; delete AutoMapper (GATE 7)
    └── 09-verification.md           ← the compiler is the verifier + final acceptance (GATE 8)
```

## How to drive this with an LLM

Give the LLM one file at a time, in order. Each file ends with a **GATE**: a list of
objectively checkable conditions. Do not feed the next file until the current GATE
passes. If a GATE fails, [troubleshooting.md](troubleshooting.md) maps the error
text to the cause and the fix.

The guide is **variance-tolerant**: your copy of the system may have more entities,
more plugin modules, different project names, or a different solution file name.
Phase 01 (discovery) produces *your* work lists from *your* code; every later phase
iterates over those lists instead of assuming fixed names or counts.
