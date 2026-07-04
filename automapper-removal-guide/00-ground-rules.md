> **AutoMapper → RECO.Mapping generic migration guide — part 1 of 9.** Start at the
> [README](README.md) and execute files in order. This guide is independent of the
> RecoAPI-specific upgrade playbook in this repository's parent folder — do not mix their
> instructions or section numbers; each guide's § numbers are local to itself.
> Next: [Discovery](01-discovery.md)

## 0. Ground rules — read completely before doing anything

### 0.1 What "replace AutoMapper" means here, precisely

By the end of this guide, in the target codebase:

- The `AutoMapper` NuGet package (and companions: `AutoMapper.Extensions.Microsoft.
  DependencyInjection`, `AutoMapper.Collection`, etc.) is removed from every project.
- Every `Profile`-derived class and every `CreateMap<>()` configuration is gone.
- Every place that used to call `_mapper.Map<T>(x)`, `_mapper.Map(x, existing)`, or
  `.ProjectTo<T>()` now calls explicit, compiled C# — either a static method implementing
  a `RECO.Mapping` interface, or a plain static/extension method, or (for LINQ
  `ProjectTo` call sites specifically) a compiled `Expression<Func<TSource,TDest>>`.
- Every mapping pair has at least one test built on `RECO.Mapping.Verification.
  MappingVerifier`, proving every destination member is populated (or explicitly
  declared unmapped-by-design) — the direct replacement for AutoMapper's
  `configuration.AssertConfigurationIsValid()`.
- The codebase's **behavior is unchanged**. This is a mechanical translation of existing
  mapping logic into explicit code, not a redesign of what gets mapped or how.

### 0.2 Your codebase is unknown — discover, never assume

This guide is generic on purpose: it has no baseline entity count, no baseline project
count, no assumption about whether mappings are simple or elaborate. Every phase after
[01-discovery.md](01-discovery.md) operates on the table *that phase* produces for *your*
codebase. Do not skip discovery because "it's probably simple" — AutoMapper's fluent API
makes it trivial to hide a custom value resolver, a conditional map, or a `ProjectTo` call
inside a large `Profile` file; find every one before writing replacement code.

### 0.3 Hard rules (do NOT deviate)

1. **DO NOT** introduce a different mapping library (Mapperly, TinyMapper, a hand-rolled
   reflection-based mapper) as the replacement. The target is `RECO.Mapping` (or, where
   the interface tier doesn't fit, plain hand-written static methods) — full stop.
2. **DO NOT** let a previously-mapped destination member go unpopulated without a
   *conscious, recorded* decision that it's unmapped-by-design. A member that silently
   stops being populated is a behavioral regression, not a refactor.
3. **DO NOT** assume AutoMapper's convention-based member-name matching (including
   "flattening", e.g. `Source.Address.City` → `Dest.AddressCity`) happened the way you'd
   guess. Verify what the old `Profile` actually configured (or, if the `Profile` relied
   purely on convention with no explicit `ForMember`, verify the *destination type's*
   property names against the *source type's* nested structure yourself) before writing
   the explicit replacement. See §3.14 for the exact procedure.
4. **DO NOT** attempt to reuse a compiled mapping method inside an EF Core (or other
   ORM) `IQueryable` LINQ query that used to call `.ProjectTo<T>()`. Compiled method calls
   are opaque to SQL translators; this needs its own pattern — see §3.15 and §5.2.
5. **DO NOT** silently approximate a mapping you can't mechanically translate (a value
   resolver with injected dependencies, `PreserveReferences()`-based cycle handling,
   polymorphic `Include<>()` inheritance mapping). Flag these per §5 and get an explicit
   decision before writing code for them.
6. **DO** run every GATE in every file. If a GATE fails and
   [07-troubleshooting.md](07-troubleshooting.md) doesn't cover it: stop and report.

### 0.4 Prerequisites — pick your library tier before starting

`RECO.Mapping`'s four interface files (`IMappedFrom`, `IMapsTo`, `IScalarCopyable`,
`IDualMapped`) use **static abstract interface members**, a C# 11 feature requiring the
.NET 7 SDK or later (the *language* version matters, not the target framework runtime —
you can target `net472` with a `net7.0`+ SDK and modern `LangVersion`, but the far more
common case is that your target framework's SDK also determines your available
`LangVersion`). Two tiers:

| Your situation | Tier to use |
| --- | --- |
| Target framework is .NET 7, 8, 9, 10, or any framework buildable with a .NET 7+ SDK and `<LangVersion>11</LangVersion>` or later | **Interface tier** — use all four interfaces as shipped in [`../src/RECO.Mapping/`](../src/RECO.Mapping/). |
| Target framework is .NET 6 or earlier, .NET Standard, or .NET Framework (4.x), and you are not raising the language version independently of the target | **Fallback tier** — skip the four interface files; use plain static/extension methods instead. `MappingVerifier` and `MappingVerificationException` are unaffected by this choice — they use only reflection and delegates, and work identically on both tiers and on every .NET version since .NET Framework 4.5. |

Determine your tier now — [02-add-the-library.md](02-add-the-library.md) branches on it
immediately. Mixed solutions (some projects on .NET 8, others still on .NET Framework)
may use different tiers per project; that's fine, the tiers interoperate at the call-site
level as long as each project consistently follows one tier internally.

### 0.5 What "done" looks like

The [08-final-checklist.md](08-final-checklist.md) is the authoritative acceptance list.
In summary: zero AutoMapper references remain anywhere in the solution; every discovered
mapping pair has explicit code and at least one passing verifier-based test; the
solution builds with no new warnings; and any pre-existing functional/integration tests
that exercised mapped data still pass unmodified (your true behavioral-regression signal
— `MappingVerifier` proves *coverage*, not *correctness of values*).

---
