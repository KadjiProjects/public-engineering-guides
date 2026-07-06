# Part 2 · Phase 8 — Rewire the generic pipeline; delete AutoMapper

## 8.1 The problem this phase solves

The pipeline classes are **generic** — `Repository<TDbModel>`,
`ActionProcessor<TDataModel, TDbModel>` — and today they call AutoMapper's
runtime-generic `IMapper`:

```csharp
// Repository<TDbModel>.Update — the two self-map uses:
var oldContent = _mapper.Map<TDbModel>(currentRecord);   // clone (snapshot before update)
_mapper.Map(dbModel, currentRecord);                     // in-place copy onto the EF-tracked instance

// ActionProcessor<TDataModel, TDbModel>:
ModifiedRecord = _mapper.Map<TDataModel>(saveResult.NewContent);   // entity -> domain
TDbModel data  = _mapper.Map<TDbModel>(request.Parameter);         // domain -> entity
var list       = _mapper.Map<List<TDbModel>>(request.Parameters);  // domain list -> entity list
```

Mapwright's methods are per-pair statics (`CodeMapper.ToEntity`), so a generic
class can't call them directly. The bridge is **delegate injection**: the generic
class asks for plain delegates in its constructor, and the one place that already
knows the concrete types (the factory / DI registration, where `Repository<Code>`
etc. is constructed) passes the entity's mapper methods. No interfaces, no
reflection, no constraints — mechanically simple and AOT-safe.

## 8.2 `Repository<TDbModel>`

Constructor and fields — replace the `IMapper` with two delegates:

```diff
-using AutoMapper;
...
-        private readonly IMapper _mapper;
+        private readonly Func<TDbModel, TDbModel> _clone;
+        private readonly Action<TDbModel, TDbModel> _copyScalars;

-        internal Repository(Func<IContextAdapter<TDbModel>> contextAdapterFactory, IMapper mapper,
-            ILogger<Repository<TDbModel>> logger, IRelatedRecordsLoader<TDbModel> relatedRecordsLoader = null)
+        internal Repository(Func<IContextAdapter<TDbModel>> contextAdapterFactory,
+            Func<TDbModel, TDbModel> clone, Action<TDbModel, TDbModel> copyScalars,
+            ILogger<Repository<TDbModel>> logger, IRelatedRecordsLoader<TDbModel> relatedRecordsLoader = null)
         {
             _contextAdapterFactory = contextAdapterFactory;
-            _mapper = mapper;
+            _clone = clone;
+            _copyScalars = copyScalars;
```

Call sites inside the class:

```diff
-            var oldContent = _mapper.Map<TDbModel>(currentRecord);
+            var oldContent = _clone(currentRecord);
...
-            _mapper.Map(dbModel, currentRecord);
+            _copyScalars(dbModel, currentRecord);
```

## 8.3 `ActionProcessor<TDataModel, TDbModel>`

Same move, three delegates:

```diff
-using AutoMapper;
...
-        private readonly IMapper _mapper;
+        private readonly Func<TDbModel, TDataModel> _toDomain;
+        private readonly Func<TDataModel, TDbModel> _toEntity;
+        private readonly Func<IEnumerable<TDataModel>, List<TDbModel>> _toEntities;
```

(constructor: replace the `IMapper mapper` parameter with the three delegates,
assign them), and at the call sites:

```diff
-                ModifiedRecord = saveResult.Success ? _mapper.Map<TDataModel>(saveResult.NewContent) : default
+                ModifiedRecord = saveResult.Success ? _toDomain(saveResult.NewContent) : default
...
-            TDbModel data = _mapper.Map<TDbModel>(request.Parameter);
+            TDbModel data = _toEntity(request.Parameter);
...
-            var data = _mapper.Map<List<TDbModel>>(request.Parameters);
+            var data = _toEntities(request.Parameters);
```

## 8.4 The construction sites — where concrete types meet

Find every place the generic classes are constructed:

```bash
grep -rn "new Repository<\|new ActionProcessor<\|CreateRepository<" --include="*.cs" src | grep -v obj
```

Baseline: a `RepositoryFactory` in Persistence and the processor registration in
the Service/DI layer, one line per entity. Each registration now passes the
entity's Mapwright methods — method groups convert to the delegates implicitly:

```csharp
// per entity, e.g. Code:
new Repository<CodeEntity>(adapterFactory, CodeMapper.Clone, CodeMapper.CopyScalars, logger, loader);
new ActionProcessor<DomainCode, CodeEntity>(..., CodeMapper.ToDomain, CodeMapper.ToEntity, CodeMapper.ToEntities, ...);
```

This is one edited line per entity per seam — for large `[ENTITIES]` lists it is
the same batch pattern as Phase 7: fix one entity, build, then sweep the rest;
the compiler lists every construction site that still has the old arity.

## 8.5 Delete AutoMapper

Only after the solution builds with the new seams:

1. Delete the factory files:
   ```bash
   grep -rln "MapperConfiguration\|IMapperFactory" --include="*.cs" src | grep -v obj
   # delete each listed file (baseline: MapperFactory.cs, IMapperFactory.cs)
   ```
2. Remove every DI registration of `IMapper` / the factory (grep `IMapper` in the
   Service and Persistence DI setup files).
3. Remove the package from **every** project in `[AUTOMAPPER-PROJECTS]`
   (including the dead references found in discovery):
   ```bash
   dotnet remove <csproj> package AutoMapper   # per project
   ```
4. Delete the old runtime configuration test if one exists
   (`AssertConfigurationIsValid` in tests) — the compiler owns that job now.

## ✅ GATE 7

1. `grep -rn "AutoMapper" --include="*.csproj" --include="*.cs" src tests | grep -v obj` → empty.
2. `dotnet build [SOLUTION]` → 0 errors, 0 MW0001.
3. `dotnet test [SOLUTION]` → pass count == Part 1 baseline (minus any deleted
   `AssertConfigurationIsValid` test, which you record).
4. Committed.

Next: [09-verification.md](09-verification.md)
