# Replacing AutoMapper with RECO.Mapping — a generic, LLM-executable guide

> **This guide is independent of the RecoAPI upgrade playbook** in the parent folder of
> this repository. That playbook upgrades one specific codebase (RecoAPI) from .NET 6 to
> .NET 10 end-to-end, of which AutoMapper removal is one phase among many. **This guide
> is scoped to exactly one thing — removing AutoMapper and replacing it with the
> `RECO.Mapping` library — and is written to apply to *any* .NET project**, on any
> target framework, whether or not that project is also being upgraded to .NET 10.
>
> If you are specifically upgrading the RecoAPI codebase itself, use the parent folder's
> guide instead (its own §3–§5 already cover this migration for that codebase precisely).
> Use *this* folder when you have a **different** project that uses AutoMapper and you
> want it gone, replaced with explicit, compiled, verifiable mapping code.

## What this guide assumes about you (the executor)

You are an AI coding agent (or a human following along) with:

- Read/write access to a .NET codebase that references the `AutoMapper` NuGet package
  somewhere — `Profile` classes with `CreateMap<>()` calls, `IMapper` injected via DI,
  `services.AddAutoMapper(...)` registration, or `.ProjectTo<T>()` LINQ usage.
- No prior knowledge of this specific codebase's mapping layout. Every phase starts with
  a **discovery step** that produces *your* work list — this guide never assumes a fixed
  number of mappings, entities, or projects, because "any project" means the shape is
  unknown until you look.
- The ability to run `dotnet build` / `dotnet test` and read their output.

## The library you are migrating to

`RECO.Mapping` — a small, dependency-free library of mapping contracts and a test-time
coverage verifier, originally built to replace AutoMapper in the RecoAPI codebase and
generalized here for reuse. Its complete, buildable source lives in this repository at
[`../src/RECO.Mapping/`](../src/RECO.Mapping/) (7 files, zero package dependencies), with
a working demo/test project at [`../tests/RECO.Mapping.Tests/`](../tests/RECO.Mapping.Tests/)
you can build and run right now with `dotnet build ../RECO.Mapping.sln && dotnet test
../RECO.Mapping.sln` to see it work before touching your own project.

Two usage tiers, chosen per mapping pair depending on what you own and what C# version
you target (full decision rules in [02-add-the-library.md](02-add-the-library.md)):

1. **Interface-based** (`IMappedFrom`/`IMapsTo`/`IScalarCopyable`/`IDualMapped`) — requires
   C# 11 / .NET 7+ (static abstract interface members) and that you can add an interface
   to the mapped type's declaration. Gives you generic-constraint call sites
   (`TDest.MapFrom(source)`, `source.MapTo()`) with zero runtime indirection.
2. **Plain static methods** — works on **any** C# version, any target framework, and any
   type you don't own (sealed, generated, third-party). No interfaces, just ordinary
   static/extension methods. `MappingVerifier` (the AutoMapper `AssertConfigurationIsValid`
   replacement) works identically either way — it only needs a `Func<TSource,TDest>`.

## How to execute (instructions for the LLM)

1. Read [00-ground-rules.md](00-ground-rules.md) completely before touching anything.
2. Execute the files in numeric order. Each ends with a **GATE** — do not proceed past a
   failing GATE; if [07-troubleshooting.md](07-troubleshooting.md) doesn't cover the
   failure, stop and report rather than improvising.
3. [01-discovery.md](01-discovery.md) produces a table of every mapping pair in the
   target codebase and how it's used. Every later phase operates on **that table**, not
   on assumptions about what a typical project looks like.
4. [03-migration-patterns.md](03-migration-patterns.md) is the core of the guide: a
   numbered catalog of every AutoMapper feature (simple maps, ignored members, custom
   value resolvers, `ReverseMap`, in-place updates, collections, nested types, automatic
   flattening, `ProjectTo`, global options) with the exact replacement code for each.
   Classify every row of your discovery table against this catalog before writing code.
5. Finish with [08-final-checklist.md](08-final-checklist.md) as your acceptance gate.

## File map

| File | Contents |
| --- | --- |
| [00-ground-rules.md](00-ground-rules.md) | Scope, hard rules, what "done" means, prerequisites per library tier |
| [01-discovery.md](01-discovery.md) | Inventory every Profile/CreateMap/IMapper usage → the work-list table · GATE 0 |
| [02-add-the-library.md](02-add-the-library.md) | Bringing RECO.Mapping into your project; the pre-C#11 fallback variant · GATE 1 |
| [03-migration-patterns.md](03-migration-patterns.md) | The pattern catalog — every AutoMapper feature → its exact replacement · GATE 2 |
| [04-rewrite-call-sites.md](04-rewrite-call-sites.md) | Replacing `IMapper`/`_mapper.Map<>()` call sites, generic pipelines, DI registration removal · GATE 3 |
| [05-special-cases.md](05-special-cases.md) | DI-dependent mappings, inheritance, circular references, ProjectTo details |
| [06-verification.md](06-verification.md) | Writing `MappingVerifier` tests per pair, both directions, ProjectTo expressions · GATE 4 |
| [07-troubleshooting.md](07-troubleshooting.md) | Error → cause → fix, generic to any project |
| [08-final-checklist.md](08-final-checklist.md) | Single-page acceptance checklist and change-summary template |

## Scope boundary — what this guide does NOT do

- It does not upgrade your target framework. If you also need to move to .NET 10, do
  that as its own, separate step (before or after this migration — your choice; this
  guide works on any target framework once the per-tier prerequisites in
  [00-ground-rules.md](00-ground-rules.md) are met).
- It does not redesign your mapped types. The goal is behavioral parity with the
  AutoMapper configuration you had, expressed as explicit code — not a data-model
  refactor. Restructuring is a separate, later decision.
- It does not cover mapping libraries other than AutoMapper (Mapperly, TinyMapper,
  manual reflection-based mappers) — though the target pattern (explicit compiled code +
  `MappingVerifier`) is a reasonable replacement for any of them.

## License

MIT, same as the rest of this repository.
