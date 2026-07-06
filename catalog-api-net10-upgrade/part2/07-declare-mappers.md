# Part 2 · Phase 7 — Declare one Mapwright mapper per entity

You will create **one file per entity** in `[ENTITIES]`, in the Persistence
project, folder `Mapping/`. Every file follows the same template; the content is
**derived mechanically from the old `MapperFactory`** — you never invent
mapping decisions, you transcribe the ones AutoMapper already recorded.

## 7.1 The derivation algorithm (apply per entity)

For entity `X` (domain record `X`, persistence entity also commonly named `X` in
a different namespace — alias one side in usings as below):

1. Open `MapperFactory` and find the **three** maps for `X`:
   - `CreateMap<Domain.X, Models.X>()` with a `ForMember(...Ignore())` list → this
     becomes `ToEntity` and its `[MapIgnore(...)]`.
   - `CreateMap<Models.X, Domain.X>()` (usually no ignores) → becomes `ToDomain`.
   - `CreateMap<Models.X, Models.X>()` with an ignore list → becomes **both**
     `Clone` and `CopyScalars`, each with that same ignore list.
2. Transcribe every `ForMember(m => m.P, opt => opt.Ignore())` into
   `nameof(EntityX.P)` inside `[MapIgnore(...)]`. **Do not drop any; do not add
   any.** The typical list is the audit trio (`User`, `Created`, `Modified`) plus
   every navigation/relationship property.
3. If the entity→domain direction has source properties the domain lacks
   (navigations, audit fields), the build will show MW0002 (Info). Add
   `[MapIgnoreSource(nameof(...))]` for each to document them. This is optional
   but keeps the build output clean.
4. Any `ForMember(...MapFrom(...))` (rare in this system's factory — baseline had
   none): a property rename becomes `[MapProperty("SourceName", "DestName")]`; a
   computed expression becomes an `[AfterMap]` method containing that expression.

## 7.2 The template (worked example: entity `Code`)

`src/Infrastructure/<Persistence>/Mapping/CodeMapper.cs`:

```csharp
using Mapwright;
using DomainCode = YourCompany.Domain.DataModels.Code;   // domain record
using CodeEntity = YourCompany.Persistence.Models.Code;  // EF entity

namespace YourCompany.Persistence.Mapping
{
    [Mapper]
    internal static partial class CodeMapper
    {
        // From: CreateMap<Domain.Code, Models.Code>() + its Ignore() list, transcribed 1:1
        [MapIgnore(nameof(CodeEntity.User), nameof(CodeEntity.Created), nameof(CodeEntity.Modified),
                   nameof(CodeEntity.CodeType), nameof(CodeEntity.ExternalRels),
                   nameof(CodeEntity.RelManies), nameof(CodeEntity.RelOnes),
                   nameof(CodeEntity.DataValues), nameof(CodeEntity.Synonyms),
                   nameof(CodeEntity.Relation_CodeType), nameof(CodeEntity.Relation_ParentCodeTypes))]
        public static partial CodeEntity ToEntity(DomainCode source);

        // From: CreateMap<Models.Code, Domain.Code>()
        public static partial DomainCode ToDomain(CodeEntity source);

        // From: the self-map — used by Repository.Update to snapshot the tracked instance
        [MapIgnore(nameof(CodeEntity.User), nameof(CodeEntity.Created), nameof(CodeEntity.Modified),
                   nameof(CodeEntity.CodeType), nameof(CodeEntity.ExternalRels),
                   nameof(CodeEntity.RelManies), nameof(CodeEntity.RelOnes),
                   nameof(CodeEntity.DataValues), nameof(CodeEntity.Synonyms),
                   nameof(CodeEntity.Relation_CodeType))]
        public static partial CodeEntity Clone(CodeEntity source);

        // From: the same self-map — used by Repository.Update to copy scalars onto the tracked instance
        [MapIgnore(nameof(CodeEntity.User), nameof(CodeEntity.Created), nameof(CodeEntity.Modified),
                   nameof(CodeEntity.CodeType), nameof(CodeEntity.ExternalRels),
                   nameof(CodeEntity.RelManies), nameof(CodeEntity.RelOnes),
                   nameof(CodeEntity.DataValues), nameof(CodeEntity.Synonyms),
                   nameof(CodeEntity.Relation_CodeType))]
        public static partial void CopyScalars(CodeEntity source, CodeEntity target);

        // From: the _mapper.Map<List<TDbModel>>(...) call site in the processing pipeline
        public static partial List<CodeEntity> ToEntities(IEnumerable<DomainCode> source);
    }
}
```

Property names above are illustrative — **use the names from YOUR
`ForMember` lists.** The `nameof` transcription is what makes rot impossible: a
name that doesn't exist is build error MW0003, today and forever.

Notes that prevent confusion:

- Domain records with `init`-only properties are fine for `ToDomain` (object
  creation). They are **not** valid targets for `CopyScalars` (in-place) — but in
  this architecture `CopyScalars` targets the mutable EF entity, so this doesn't
  arise. If MW0006 fires anyway, a mutable entity has an init-only property; add
  it to that declaration's `[MapIgnore]`.
- `bool?` entity columns → non-nullable domain `bool` collapse automatically to
  `GetValueOrDefault()` in the generated code — same behavior AutoMapper had,
  now visible.
- Read what got generated at least once:
  `obj/generated/` appears if you add `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>`
  and `<CompilerGeneratedFilesOutputPath>$(BaseIntermediateOutputPath)generated</CompilerGeneratedFilesOutputPath>`
  to the csproj; otherwise view it in the IDE under Dependencies → Analyzers.

## 7.3 Batching this across many entities

The per-entity work is a pure transcription with identical structure —
**parallelize it mechanically**:

- Do ONE entity end-to-end first (declare → build → fix MW0001s → clean). This
  calibrates the pattern against your real property lists.
- Then process the rest of `[ENTITIES]` in bulk: create all remaining files from
  the template, build **once**, and work the MW diagnostic list top to bottom.
  The build output is itself your work list — every MW0001 line names the file,
  property, and method that needs a decision.
- For fleets of dozens of entities: a throwaway script can pre-generate the
  skeleton files by parsing `MapperFactory.cs` (regex per `CreateMap<`/`ForMember(`
  block) and emitting the template with the transcribed ignore lists — then the
  MW diagnostics verify the transcription. This is safe precisely because the
  compiler checks the result; a transcription typo cannot survive the build.

## 7.4 Drive MW0001 to zero

```bash
dotnet build [SOLUTION] 2>&1 | grep "MW000"
```

For each MW0001: the property either (a) has a matching source property under
case-insensitive matching — then something's misspelled; or (b) was on the old
ignore list — transcribe it; or (c) is genuinely new since the factory was last
touched — make the decision explicitly (map / ignore / AfterMap). Case (c) is
Mapwright doing its job: those were silently unmapped before.

## ✅ GATE 6

1. One `<Entity>Mapper.cs` file exists per entry in `[ENTITIES]`.
2. `dotnet build [SOLUTION]` → 0 errors, **0 MW0001 warnings** (MW0002 Infos
   acceptable; zero preferred).
3. The old `MapperFactory` is still present and untouched (deleted next phase).
4. Committed.

Next: [08-rewire-pipeline.md](08-rewire-pipeline.md)
