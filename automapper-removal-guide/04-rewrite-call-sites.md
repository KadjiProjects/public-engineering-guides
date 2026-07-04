> **AutoMapper → RECO.Mapping generic migration guide — part 5 of 9.**
> Previous: [Migration patterns](03-migration-patterns.md) · Next: [Special cases](05-special-cases.md)

## 4. Rewrite call sites and remove AutoMapper's DI wiring

With every mapping pair's replacement code written (§3), work through every call site
recorded in [discovery](01-discovery.md) §1.4.

### 4.1 Simple construction call sites

| Old | New (interface tier) | New (fallback tier) |
| --- | --- | --- |
| `_mapper.Map<TDest>(source)` | `source.MapTo()` or `TDest.MapFrom(source)` (either is correct — see below) | `source.ToDest()` / `DestMapper.FromSource(source)` |
| `_mapper.Map<List<TDest>>(sources)` | `sources.MapFromAll<TDest,TSource>()` | `sources.MapAll(DestMapper.FromSource)` |

When both `MapTo()` and `TDest.MapFrom(source)` are available for the same pair (i.e.
you implemented `IDualMapped`), pick whichever reads more naturally at each call site:
`source.MapTo()` when the call site already holds a `source` instance and is projecting
it outward; `TDest.MapFrom(source)` when the call site is constructing a `TDest` and the
source is just an input (e.g. inside a repository's insert method). Consistency within
one file matters more than which you pick.

### 4.2 In-place update call sites

| Old | New (interface tier) | New (fallback tier) |
| --- | --- | --- |
| `_mapper.Map(source, existingInstance)` | `source.CopyScalarsTo(existingInstance)` | `SourceMapper.CopyScalarsTo(source, existingInstance)` |
| `var old = _mapper.Map<T>(existingInstance);` (pre-update snapshot) | `var old = existingInstance.CloneScalars();` | `var old = existingInstance.CloneScalars();` (extension method, fallback tier) |

### 4.3 Generic pipeline / service classes parameterized by `<TSource, TDest>`

If discovery (§1.4) flagged a class like:

```csharp
public class ActionProcessor<TDataModel, TDbModel>
{
    public ActionProcessor(IMapper mapper, /* ... */) { _mapper = mapper; }
    // internally calls _mapper.Map<TDbModel>(x), _mapper.Map<TDataModel>(y), etc.
}
```

this needs one of two treatments depending on whether you own both `TDataModel` and
`TDbModel`'s source for *every* instantiation of this generic class in the codebase:

**You own both type families (interface tier available everywhere this class is used):**
add the interface as a generic constraint on the class itself — the mapping is now
resolved at compile time per closed generic type, with no injected mapper at all:

```csharp
public class ActionProcessor<TDataModel, TDbModel>
    where TDbModel : IDualMapped<TDbModel, TDataModel>
{
    // no IMapper constructor parameter anymore
    private void Save(TDataModel data)
    {
        TDbModel dbModel = TDbModel.MapFrom(data);
        // ...
    }

    private TDataModel Load(TDbModel dbModel) => dbModel.MapTo();
}
```

**You don't own the types everywhere this class is used** (fallback tier, or some
instantiations use third-party/sealed types): replace the injected `IMapper` with two
plain delegates supplied by the caller — this preserves the "pluggable mapping"
flexibility AutoMapper's runtime `IMapper` gave you, but each delegate is a concrete
compiled method reference resolved at the composition root, not a runtime type lookup:

```csharp
public class ActionProcessor<TDataModel, TDbModel>
{
    private readonly Func<TDataModel, TDbModel> _toDbModel;
    private readonly Func<TDbModel, TDataModel> _toDataModel;

    public ActionProcessor(Func<TDataModel, TDbModel> toDbModel, Func<TDbModel, TDataModel> toDataModel, /* ... */)
    {
        _toDbModel = toDbModel;
        _toDataModel = toDataModel;
    }

    private void Save(TDataModel data) { TDbModel dbModel = _toDbModel(data); /* ... */ }
    private TDataModel Load(TDbModel dbModel) => _toDataModel(dbModel);
}

// composition root / DI registration, one line per concrete type pair:
services.AddScoped<ActionProcessor<PersonDto, PersonEntity>>(sp =>
    new ActionProcessor<PersonDto, PersonEntity>(PersonMapper.ToEntity, PersonMapper.ToDto, /* ... */));
```

Either way, **every previously-dynamic mapping call becomes visible and compile-time
checked at its instantiation site** — this is a genuine improvement AutoMapper's
`IMapper` (which resolves any `Map<T>()` call at runtime against its whole
configuration) could not offer, but it does mean each generic instantiation needs an
explicit one-line wiring instead of "it just works because AutoMapper knows about it".

### 4.4 Remove AutoMapper's DI registration and package references

```bash
# For every project found in discovery §1.1:
dotnet remove <path-to-csproj> package AutoMapper
dotnet remove <path-to-csproj> package AutoMapper.Extensions.Microsoft.DependencyInjection  # if present
```

Delete the `services.AddAutoMapper(...)` call (found in discovery §1.5) and every
`Profile`-derived class — their contents are now fully absorbed into the pattern-specific
mapping methods from §3. If a `Profile` class still compiles after deletion of its
`AutoMapper` using directive, that's a sign something inside it wasn't actually
AutoMapper-specific (rare, but check before deleting the file outright).

### ✅ GATE 3

```bash
grep -rn "AutoMapper\|IMapper\b" --include="*.cs" --include="*.csproj" .
```

returns **nothing** in your target codebase (matches inside this guide's own files, if
you copied it alongside your project, don't count). Every call site from discovery §1.4
now calls explicit code. Every generic pipeline class from §1.4's last bullet has been
converted per §4.3.

---
