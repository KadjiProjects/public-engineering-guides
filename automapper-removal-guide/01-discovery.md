> **AutoMapper → RECO.Mapping generic migration guide — part 2 of 9.** Start at the
> [README](README.md) if you haven't read [Ground rules](00-ground-rules.md) yet.
> Previous: [Ground rules](00-ground-rules.md) · Next: [Add the library](02-add-the-library.md)

## 1. Discovery — inventory every AutoMapper usage

Run every command below from the solution root. Every later phase works from the table
you build at the end of this section — there is no baseline to compare against, because
this guide applies to a codebase this guide has never seen.

### 1.1 Locate the package references

```bash
grep -rln "AutoMapper" --include="*.csproj" .
```

Note every hit — these projects lose their `AutoMapper` (and any `AutoMapper.*`
companion package) reference in [04-rewrite-call-sites.md](04-rewrite-call-sites.md) §4.4.

### 1.2 Locate every `Profile` class

```bash
grep -rl "using AutoMapper" --include="*.cs" .
grep -rn ": *Profile\b" --include="*.cs" .
```

Open **every** file this finds. A `Profile` constructor is where `CreateMap<>()` chains
live; nothing about their contents is discoverable by grep alone once modifiers
(`.ForMember`, `.Ignore()`, `.ReverseMap()`, `.ConvertUsing()`, `.Condition()`, custom
`IValueResolver`/`ITypeConverter` classes) span multiple lines.

### 1.3 Locate every `CreateMap<>()` call and its modifiers

```bash
grep -rn "CreateMap<" --include="*.cs" .
```

For each hit, read the **full statement** (the fluent chain often continues for several
lines after the `CreateMap<Source, Dest>()` call). Record, per mapping pair:

- Source type, destination type, and which assembly/project each lives in.
- Whether **you own the source** of both types (can you add a `partial` modifier and an
  interface implementation, or is one/both sealed, generated, or from a NuGet package?).
- Every modifier present, classified against this list (full definitions and
  replacements for each are in [03-migration-patterns.md](03-migration-patterns.md) —
  this step only needs you to *notice and record*, not yet solve):

| Modifier / shape found | Pattern # (see §3) |
| --- | --- |
| No modifiers — plain member-name matching | 3.1 |
| `.ForMember(d => d.X, opt => opt.Ignore())` | 3.2 |
| `.ForMember(d => d.X, opt => opt.MapFrom(s => expr))` | 3.3 |
| `.ForMember(d => d.X, opt => opt.MapFrom<SomeResolver>())` (a class implementing `IValueResolver<,,>`) | 3.4 |
| `.ReverseMap()` | 3.5 |
| `.ConvertUsing(...)` | 3.6 |
| `.ConstructUsing(...)` | 3.7 |
| `.ForMember(..., opt => opt.Condition(...))` | 3.8 |
| `.ForMember(..., opt => opt.NullSubstitute(...))` | 3.9 |
| Called via `_mapper.Map(source, existingInstance)` (two-argument overload) anywhere in the codebase | 3.10 |
| `CreateMap<T, T>()` (identical source/destination type — clone/self-map) | 3.11 |
| Destination has a property whose type is itself a mapped pair (nested object) | 3.12 |
| Destination has a `List<>`/`IEnumerable<>` of a nested mapped pair | 3.13 |
| Destination property name looks like `Source.NestedType.Property` concatenated (e.g. `AddressCity` when `Source.Address.City` exists), with **no** explicit `ForMember` for it | 3.14 — AutoMapper's automatic flattening convention |
| `.Include<Derived, DerivedDto>()` (inheritance/polymorphic mapping) | 5.2 |
| `PreserveReferences()` | 5.3 |

A single `CreateMap<>()` frequently matches several rows at once (e.g. a mapping with
one ignored member, one custom expression, and a nested collection) — record all that
apply.

### 1.4 Locate every call site

```bash
grep -rn "IMapper\b\|_mapper\.\|Mapper\.Map<\|\.ProjectTo<" --include="*.cs" .
```

For each hit, record the file:line, which method it's in, and which mapping pair (from
§1.3) it invokes. **Every call site must trace back to a row in your §1.3 table** — if
one doesn't, the corresponding `CreateMap<>()` is either missing from your inventory or
the call resolves a mapping via an interface/base-type registered elsewhere; keep
looking until every call site is accounted for.

Separately flag:
- Call sites using the **two-argument** `Map(source, existingInstance)` overload — these
  are in-place updates (pattern 3.10), a different shape than construction.
  `_mapper.Map<TDest>(source)` and `_mapper.Map(source, existing)` are NOT the same
  pattern even for the same `TSource`/`TDest` pair — check the argument count.
- Call sites inside a `.Select(...)` or directly as `.ProjectTo<TDest>()` on an
  `IQueryable<TSource>` — these are LINQ-translated projections (pattern 3.15),
  fundamentally different from everything else in this guide.
- Generic pipeline/service classes parameterized by `<TSource,TDest>` (or similar) that
  take `IMapper` as a constructor dependency and call it internally with type parameters
  rather than concrete types — these need the generic-constraint or delegate-injection
  treatment in [04-rewrite-call-sites.md](04-rewrite-call-sites.md) §4.3, not a simple
  one-line replacement.

### 1.5 Locate the DI registration and configuration validation

```bash
grep -rn "AddAutoMapper\|AssertConfigurationIsValid\|MapperConfiguration" --include="*.cs" .
```

Record every hit — `AddAutoMapper(...)` registrations are removed in
[04-rewrite-call-sites.md](04-rewrite-call-sites.md); `AssertConfigurationIsValid()` test
calls are replaced one-for-one by `MappingVerifier` tests in
[06-verification.md](06-verification.md).

### 1.6 Locate global configuration options

Inside the same `MapperConfiguration`/`Profile` files, check for options that apply
across *every* mapping rather than to one pair specifically:

```bash
grep -rn "AllowNullCollections\|AllowNullDestinationValues\|ForAllMaps\|ForAllPropertyMaps\|SourceMemberNamingConvention\|DestinationMemberNamingConvention" --include="*.cs" .
```

Record any hits — see §3.16 for how to re-derive their effect explicitly per pair.

### Your discovery deliverable

A table with one row per mapping pair (source type, destination type, project/assembly
of each, own-both-types? yes/no, list of pattern #s from §1.3, list of call sites from
§1.4 with file:line, in-place-update usage? yes/no, ProjectTo usage? yes/no). This table
is what [03-migration-patterns.md](03-migration-patterns.md) and
[06-verification.md](06-verification.md) are executed against.

### ✅ GATE 0

- Every `CreateMap<>()` found in §1.3 has a row in your table.
- Every call site found in §1.4 traces to exactly one row (or is itself flagged as a
  generic pipeline needing special handling per §1.4's last bullet).
- Every global option found in §1.6 is recorded against the pairs it affects (all pairs,
  if truly global).
- You know, for every row, whether you own both mapped types' source code.

If any of the above is incomplete, do not proceed — an incomplete inventory means a
later phase will silently miss a mapping.

---
