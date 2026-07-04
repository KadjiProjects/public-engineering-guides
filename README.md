# RecoAPI .NET 6 → .NET 10 upgrade guide (LLM-executable)

A complete, self-contained playbook for upgrading **any fork of the RecoAPI codebase**
from .NET 6 to .NET 10, including the removal of AutoMapper in favor of a hand-written,
compile-time-safe mapping library. It is written **for an AI coding agent**: every file
to create is given verbatim, every edit is an exact before/after, every phase ends with
a machine-checkable **GATE**, and every known failure has a troubleshooting entry — so
the least capable model can perform the upgrade as accurately as the most capable one.

Humans can follow it too; it is just deliberately over-specified.

## What the upgrade delivers

- All projects retargeted `net6.0` → `net10.0`, packages moved to a **pinned, tested
  version matrix** (EF Core 10, Microsoft.Extensions 10, current Serilog stack,
  Swashbuckle 10, NUnit 3.x kept deliberately).
- The three .NET-10-era breaking changes handled: **SqlClient `Encrypt=true` default**
  (runtime-only failure), **Serilog.Sinks.Email 3+ API removal** (full wrapper port to
  `EmailSinkOptions`/MailKit), **Swashbuckle 10 / Microsoft.OpenApi 2 namespace move**.
- **AutoMapper removed** and replaced by a small dependency-free library
  (static-abstract mapping contracts + a test-time mapping verifier), with three
  generated coverage tests per entity.
- End state of the verified baseline run: **build with 0 warnings / 0 errors**, all
  mapping tests green, identical runtime behavior.

## The AutoMapper replacement — also included as real, buildable code

The library the guide has you write (§3, "RECO.Mapping") isn't only described in
markdown — this repository ships it as an actual, standalone, dependency-free .NET 10
class library you can build and test right now, no RecoAPI checkout required:

```
src/RECO.Mapping/            the library — same 7 files as §3, copied verbatim from the
                              real, working RecoAPI source (not retyped from the guide)
tests/RECO.Mapping.Tests/    a self-contained NUnit suite: 8 tests demonstrating every
                              contract (IMappedFrom, IMapsTo, IScalarCopyable,
                              IDualMapped, the list helpers) against generic demo types,
                              plus 3 tests proving MappingVerifier catches a forgotten
                              property and an accidentally-copied ignored property
RECO.Mapping.sln              solution tying the two together
```

```bash
git clone https://github.com/KadjiProjects/RecoApi-Net10-Upgrade-Guide.git
cd RecoApi-Net10-Upgrade-Guide
dotnet build RECO.Mapping.sln     # 0 warnings, 0 errors
dotnet test  RECO.Mapping.sln     # 8/8 passed
```

The demo types (`PersonDto`/`PersonEntity`) stand in for whatever your domain/EF models
are — RecoAPI's real mapping partials for `TblCode` etc. follow the exact same shape and
live in [appendix-a-entity-mappings.md](appendix-a-entity-mappings.md); they aren't
included as compilable files here because they depend on RecoAPI's private domain and
persistence model types. Copy `src/RECO.Mapping/` as-is into your own fork (it has zero
package dependencies), then write one mapping partial per entity following §4.

## Using AutoMapper removal on a *different* project (not RecoAPI)

Everything above is specific to the RecoAPI codebase. If you have a **different**
project that uses AutoMapper and just want it gone — independent of any .NET 10 upgrade
— use the separate, generic, self-contained guide instead:

📁 **[automapper-removal-guide/](automapper-removal-guide/)** — applies to any .NET
project, any target framework (with a documented fallback for pre-C#11/.NET 7 codebases
that can't use static abstract interfaces), and covers the *entire* AutoMapper surface:
simple maps, ignored members, custom value resolvers (with and without DI dependencies),
`ReverseMap`, in-place updates, nested/collection mapping, automatic member-name
flattening, `ProjectTo<T>()` LINQ projections, inheritance mapping, and reference-cycle
handling — with a numbered pattern catalog giving the exact replacement code for each.

It does not touch or depend on anything RecoAPI-specific; it's a standalone document set
that happens to live in this repository because it shares the same underlying library
(`src/RECO.Mapping/`, described next).

## How to execute (instructions for the LLM)

1. Read [00-ground-rules.md](00-ground-rules.md) **completely** before touching
   anything. Its hard rules override anything you believe you know.
2. Execute the files **in numeric order**. Never start a phase before the previous
   phase's GATE passes. Never skip a GATE.
3. Your fork may differ from the baseline (more entities, more module projects, more
   connection strings). The variance rules in §0.2 and the discovery commands in
   §1 exist for exactly that — **enumerate, never assume baseline counts**.
4. If a GATE fails and [07-troubleshooting.md](07-troubleshooting.md) does not cover
   the failure: **stop and report**. Do not improvise.
5. After final acceptance (GATE 5), read
   [08-post-upgrade-review.md](08-post-upgrade-review.md) and propose its hardening
   items to the repository owner as separate change sets.

## File map

| File | Contents | Global § numbers |
| --- | --- | --- |
| [00-ground-rules.md](00-ground-rules.md) | Architecture, variance rules, hard rules, prerequisites | §0 |
| [01-discovery.md](01-discovery.md) | Phase 0: enumerate projects/entities/connection strings → your work lists · GATE 0 | §1 |
| [02-retarget-and-packages.md](02-retarget-and-packages.md) | Phase 1: TFMs, the package version matrix, SqlClient connection strings, Swashbuckle namespace, email-sink port, nullability fixes · GATE 1 | §2 |
| [03-mapping-library.md](03-mapping-library.md) | Phase 2: the complete mapping library source (7 files, verbatim — also buildable at [`src/RECO.Mapping/`](src/RECO.Mapping/)) · GATE 2 | §3 |
| [04-entity-mappings.md](04-entity-mappings.md) | Phase 3: one mapping file per entity — verbatim for baseline entities, derivation algorithm for the rest · GATE 3 | §4 |
| [05-rewire-pipeline.md](05-rewire-pipeline.md) | Phase 4: exact edits to Repository, RepositoryFactory, DI, ActionProcessor; delete AutoMapper code · GATE 4 | §5 |
| [06-verification.md](06-verification.md) | Phase 5: test project, coverage tests, verifier self-tests, final acceptance · GATE 5, smoke test, deployment notes | §6 |
| [07-troubleshooting.md](07-troubleshooting.md) | Error → cause → fix table for every known failure mode | §7 |
| [08-post-upgrade-review.md](08-post-upgrade-review.md) | Post-upgrade code-review findings: known issues & hardening (optional, after GATE 5) | §8 |
| [appendix-a-entity-mappings.md](appendix-a-entity-mappings.md) | Verbatim mapping files for the 8 baseline entities | App. A |
| [appendix-b-coverage-tests.md](appendix-b-coverage-tests.md) | Verbatim coverage test file for the 8 baseline entities | App. B |

Cross-references inside the text use the global § numbers (e.g. "§4.2" is in
`04-entity-mappings.md`, "§2.4" is in `02-retarget-and-packages.md`).

## Scope and provenance

- Derived from a real, completed, verified upgrade of the RecoAPI baseline (11 projects,
  8 mapped entities). The guide is **variance-tolerant**: it tells the executor how to
  discover and handle forks that grew beyond the baseline.
- **Sanitized**: contains no credentials, no connection strings to real servers, no
  machine names, and no private paths. Where the codebase contains developer-specific
  values (module paths, test database servers), the guide references them by *pattern*
  and § 8 tells you to make them portable.
- The guide performs a **conservative** upgrade: same architecture, same hosting model,
  same test framework, same behavior. Modernization beyond that is explicitly out of
  scope (and explicitly forbidden by the ground rules) so that the upgrade diff stays
  reviewable.

## License

MIT — use it, fork it, adapt the structure for your own upgrade playbooks.
