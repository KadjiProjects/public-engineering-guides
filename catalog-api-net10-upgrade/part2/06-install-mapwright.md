# Part 2 · Phase 6 — Install Mapwright and understand what replaces what

> **Precondition:** Part 1 GATE 4 passed (system green on .NET 10, AutoMapper
> still in place). Work on a fresh branch from that tag.

## 6.1 What Mapwright is, in one paragraph

[Mapwright](https://github.com/lodestar-labs/Mapwright) replaces AutoMapper's
runtime reflection engine with a **Roslyn source generator**: you declare each
mapping as a bodyless `static partial` method on a `[Mapper]` class, and the
generator writes the implementation as plain C# **during compilation**. Unmapped
destination properties become compiler diagnostics (`MW0001`), stale ignore lists
become build **errors** (`MW0003`) — so AutoMapper's runtime
`AssertConfigurationIsValid()` test is replaced by the build itself. Nothing
executes at runtime; there is no `IMapper` and nothing to register in DI.

## 6.2 What maps to what in THIS system

| Today (AutoMapper) | After (Mapwright) |
|---|---|
| Static `MapperFactory` with `MapperConfiguration`, 3 `CreateMap`s per entity | One `[Mapper]` static class per entity with 3–4 declarations (file per entity, next phase) |
| `CreateMap<Domain, Entity>()` + `ForMember(...Ignore())` list | `static partial Entity ToEntity(Domain source);` + `[MapIgnore(nameof(...), ...)]` |
| `CreateMap<Entity, Domain>()` | `static partial Domain ToDomain(Entity source);` |
| Self-map `CreateMap<Entity, Entity>()` used for **cloning** (`_mapper.Map<TDbModel>(x)`) | `static partial Entity Clone(Entity source);` with the same ignores |
| Self-map used for **in-place update** (`_mapper.Map(src, dest)` on the EF-tracked instance) | `static partial void CopyScalars(Entity source, Entity target);` |
| `_mapper.Map<List<TDbModel>>(list)` | `static partial List<Entity> ToEntities(IEnumerable<Domain> source);` |
| `IMapper` injected into generic `Repository<TDbModel>` / `ActionProcessor<TDataModel,TDbModel>` | Delegates (`Func<...>`/`Action<...>`) injected at the same constructor seams — Phase 8 |
| `AssertConfigurationIsValid()` at startup/test | The compiler (MW0001/MW0003), every build |

## 6.3 Install the two packages

In **each** project from your `[AUTOMAPPER-PROJECTS]` *code-usage* list
(baseline: Persistence and Processing — NOT the projects with dead references):

```xml
<ItemGroup>
  <PackageReference Include="Mapwright" Version="0.1.0" />
  <PackageReference Include="Mapwright.Generator" Version="0.1.0" PrivateAssets="all" />
</ItemGroup>
```

Building from source instead: reference `Mapwright.csproj` normally and
`Mapwright.Generator.csproj` with
`OutputItemType="Analyzer" ReferenceOutputAssembly="false"`.

**Do not remove AutoMapper yet.** Both coexist until Phase 8; that keeps the
build green after every single step.

## 6.4 The diagnostics you will drive by

| Id | Severity | Meaning | Your reaction |
|---|---|---|---|
| MW0001 | Warning | Destination property not mapped | Map it, `[MapIgnore]` it, or `[AfterMap]` it — same decision the old `ForMember` list made |
| MW0002 | Info | Source property never read | `[MapIgnoreSource]` to document one-way fields |
| MW0003 | Error | Ignore/rename names a nonexistent property | Fix the name — this is rot the old system never caught |
| MW0004 | Error | No conversion between matched types | Check the pair; usually a rename collision |
| MW0005 | Error | Signature not a recognized shape | Compare against §6.2's shapes exactly |
| MW0006 | Warning | Init-only property in an in-place copy | `[MapIgnore]` it on the `CopyScalars` declaration |
| MW0008 | Error | Collection map missing its element map | Declare the single-object map in the same class |
| MW0009 | Error | Bad `[AfterMap]` | Signature must be `static void (TSource, TDest)` in the same class |

## ✅ GATE 5

1. `dotnet restore` succeeds with the two Mapwright references present in every
   code-usage project.
2. `dotnet build [SOLUTION]` still green (nothing declared yet, nothing removed).
3. AutoMapper still present and untouched.
4. Committed.

Next: [07-declare-mappers.md](07-declare-mappers.md)
