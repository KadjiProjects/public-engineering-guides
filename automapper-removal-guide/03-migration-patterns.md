> **AutoMapper → RECO.Mapping generic migration guide — part 4 of 9.**
> Previous: [Add the library](02-add-the-library.md) · Next: [Rewrite call sites](04-rewrite-call-sites.md)

## 3. Migration patterns — every AutoMapper feature and its exact replacement

Work through your [discovery](01-discovery.md) table row by row. For each row, apply
every pattern number recorded against it, in the order given below when more than one
applies to the same pair (nested/collection patterns, §3.12–3.13, are usually applied
*around* whichever base pattern §3.1–3.11 the pair's top-level shape uses).

Each pattern gives **both library tiers' code** where the tier changes the shape;
where it doesn't (most of §3.2–§3.9 — these are about what goes *inside* a mapping
method, not the method's shape), one example suffices for both.

---

### 3.1 Simple map — `CreateMap<Source, Dest>()`, no modifiers

Every property is copied by matching name. Write every assignment explicitly — do not
rely on the compiler or a helper to "match names for you"; that reflection-based
convenience is exactly what AutoMapper provided and exactly what you're removing.

**Interface tier** (you own `Dest`'s source):

```csharp
public sealed class PersonDto
{
    public int Id { get; set; }
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
}

public sealed class PersonEntity : IMappedFrom<PersonEntity, PersonDto>
{
    public int Id { get; set; }
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";

    public static PersonEntity MapFrom(PersonDto source) => new()
    {
        Id = source.Id,
        FirstName = source.FirstName,
        LastName = source.LastName,
    };
}

// call site: var entity = PersonEntity.MapFrom(dto);
```

**Fallback tier** (or you don't own either type — e.g. `PersonEntity` is sealed/generated):

```csharp
public static class PersonMapper
{
    public static PersonEntity ToEntity(this PersonDto source) => new()
    {
        Id = source.Id,
        FirstName = source.FirstName,
        LastName = source.LastName,
    };
}

// call site: var entity = dto.ToEntity();
```

Both are equally valid replacements; the interface tier only buys you generic-constraint
call sites in pipeline code (§4.3) — pick per-pair based on whether you own the type and
whether anything downstream needs the generic constraint.

### 3.2 Ignored member — `.ForMember(d => d.X, opt => opt.Ignore())`

Simply don't assign `X` in the new method. Record `nameof(Dest.X)` — you will pass it as
`unmappedByDesign` to `MappingVerifier.AssertAllMembersMapped` in
[06-verification.md](06-verification.md), so the test proves it was *intentionally*
skipped rather than *forgotten*.

### 3.3 Custom member expression — `.ForMember(d => d.X, opt => opt.MapFrom(s => expr))`

Inline `expr` as `X`'s assignment:

```csharp
FullName = $"{source.FirstName} {source.LastName}",
```

If `expr` is non-trivial, extract a small private static helper in the same mapping
class rather than a one-liner — keep it in the same assembly, no runtime indirection.

### 3.4 Value resolver class — `.ForMember(d => d.X, opt => opt.MapFrom<SomeResolver>())`

Find `SomeResolver`'s `Resolve(source, destination, destMember, context)` method body.
If it has **no constructor dependencies**, inline its logic exactly like §3.3. If it
**does** have injected dependencies (a lookup service, current-user accessor, clock,
etc.), this mapping can no longer be a static method — see
[05-special-cases.md](05-special-cases.md) §5.1 (DI-dependent mapping services).

### 3.5 `ReverseMap()`

Implement both directions.

**Interface tier:**

```csharp
public sealed class PersonEntity : IDualMapped<PersonEntity, PersonDto>
{
    // ... properties ...

    public static PersonEntity MapFrom(PersonDto source) => new() { /* ... */ };
    public PersonDto MapTo() => new() { /* ... */ };
}
```

`IDualMapped<TSelf,TOther>` is exactly `IMappedFrom<TSelf,TOther> + IMapsTo<TOther>` —
use it whenever both directions are needed instead of implementing them separately.

**Fallback tier:** two extension methods, one per direction, in the same static class:

```csharp
public static class PersonMapper
{
    public static PersonEntity ToEntity(this PersonDto source) => new() { /* ... */ };
    public static PersonDto ToDto(this PersonEntity entity) => new() { /* ... */ };
}
```

If AutoMapper's `ReverseMap()` had its **own** additional `ForMember`/`Ignore` modifiers
for the reverse direction (this is allowed and common — e.g. `A→B` maps a computed field
that `B→A` can't reconstruct and must ignore), apply those modifiers **only** to that
direction's method, not both.

### 3.6 `ConvertUsing(...)` — whole-type custom conversion

AutoMapper skips member-by-member mapping entirely and calls your conversion function.
This is the simplest case in disguise — write `MapFrom`/`MapTo`/`ToEntity`/`ToDto` (per
your tier) with a single expression matching what the `ConvertUsing` lambda or
`ITypeConverter` did. There is nothing else to translate.

### 3.7 `ConstructUsing(...)` — custom construction, then normal member mapping continues

Put the custom construction expression in the `new(...)` call itself, then continue with
the usual per-property assignments in the object initializer for whatever
`ConstructUsing` left for AutoMapper to fill afterward:

```csharp
public static PersonEntity MapFrom(PersonDto source) => new PersonEntity(source.Id) // custom ctor call
{
    FirstName = source.FirstName, // normal member mapping continues
    LastName = source.LastName,
};
```

### 3.8 `.Condition(...)` — member mapped only if a predicate holds

Two distinct situations, tell them apart before choosing:

- **Constructing a new instance** (the common case): inline as a conditional expression.
  ```csharp
  Discount = source.IsPremiumMember ? source.Discount : 0m,
  ```
- **Updating an existing instance** where the condition's purpose is "leave the existing
  value alone if the condition is false" (the destination already has a value that must
  be *preserved*, not defaulted): this is actually the in-place-update pattern — see
  §3.10 and put the conditional inside `CopyScalarsTo`'s body instead:
  ```csharp
  public void CopyScalarsTo(PersonEntity target)
  {
      if (IsPremiumMember) target.Discount = Discount; // else target.Discount is untouched
      target.FirstName = FirstName; // unconditional members still copy normally
  }
  ```

### 3.9 `.NullSubstitute(...)`

Inline as a null-coalescing expression:

```csharp
DisplayName = source.PreferredName ?? "Anonymous",
```

### 3.10 In-place update — `_mapper.Map(source, existingDestination)` (two-argument overload)

This is `IScalarCopyable<T>.CopyScalarsTo` — updating an already-existing/tracked
instance without replacing it or disturbing its navigation/relationship properties
(very likely why the original code used the two-argument overload instead of
construction: to keep an EF Core change-tracked entity intact).

**Interface tier:**

```csharp
public sealed class PersonEntity : IScalarCopyable<PersonEntity>
{
    public int Id { get; set; }
    public string FirstName { get; set; } = "";
    public List<OrderEntity> Orders { get; set; } = []; // navigation — must NOT be touched

    public PersonEntity CloneScalars() => new() { Id = Id, FirstName = FirstName };

    public void CopyScalarsTo(PersonEntity target)
    {
        target.Id = Id;
        target.FirstName = FirstName;
        // target.Orders is intentionally left untouched.
    }
}

// call site (was: _mapper.Map(incoming, tracked)):
incoming.CopyScalarsTo(tracked);
```

**Fallback tier:**

```csharp
public static class PersonMapper
{
    public static PersonEntity CloneScalars(this PersonEntity source) => new()
    {
        Id = source.Id, FirstName = source.FirstName,
    };

    public static void CopyScalarsTo(this PersonEntity source, PersonEntity target)
    {
        target.Id = source.Id;
        target.FirstName = source.FirstName;
        // target.Orders is intentionally left untouched.
    }
}
```

If the old code also snapshotted "before" state prior to overwriting
(`var old = _mapper.Map<T>(existing);` immediately before the in-place `Map(...)` call),
that snapshot call is `CloneScalars()`.

### 3.11 Self-mapping / deep clone — `CreateMap<T, T>()`, `_mapper.Map<T>(instance)` where source == destination type

If this was used to snapshot scalar state, it's `CloneScalars()` from §3.10. If a
genuine **deep** clone (including navigation/collections) is actually required, write an
explicit manual clone instead of trying to force it through the scalar-only contract:

```csharp
public PersonEntity DeepClone() => new()
{
    Id = Id, FirstName = FirstName,
    Orders = [.. Orders.Select(o => o.DeepClone())],
};
```

For `record` types, prefer a `with` expression for the shallow case
(`instance with { }`, or `instance with { FirstName = newName }` for a modified copy) —
records already give you this for free; don't wrap it in a mapping method needlessly.

### 3.12 Nested / complex-type member (destination property is itself a mapped pair)

Give the nested pair its own mapping first (leaf types before parents — work through
your discovery table in dependency order, innermost types first), then the parent
mapping calls it directly:

```csharp
// AddressDto/AddressEntity already have their own MapFrom/MapTo from §3.1 or §3.5.
public static PersonEntity MapFrom(PersonDto source) => new()
{
    Id = source.Id,
    Address = source.Address is null ? null : AddressEntity.MapFrom(source.Address),
    // fallback tier: Address = source.Address?.ToEntity(),
};
```

### 3.13 Collection of a nested mapped pair

Use the library's list helpers after the nested pair has its own mapping (§3.12):

```csharp
// Interface tier:
Lines = source.Lines.MapFromAll<LineEntity, LineDto>(),

// Fallback tier:
Lines = source.Lines.MapAll(LineMapper.ToEntity),
```

### 3.14 Automatic member-name "flattening" — no explicit `ForMember`, but the property name pattern-matches a nested source path

**This is the single most common source of a silent regression in this kind of
migration.** AutoMapper's convention engine maps `Source.Address.City` to
`Dest.AddressCity` (and similar `Foo.Bar` → `FooBar` patterns) with **zero explicit
configuration** — nothing in the `Profile` file tells you this is happening. It compiles
fine either way; a forgotten flattened property just silently stays at its default value
at runtime, which is very easy to miss in review.

**Procedure:** for every destination type, list every property name. For each one that
doesn't correspond to a same-named source property, check whether it matches a
concatenation of the source type's *nested* property paths (case-insensitively,
ignoring word boundaries — AutoMapper's convention is fuzzy). If it does, write the
explicit expression:

```csharp
AddressCity = source.Address?.City ?? "",
AddressPostalCode = source.Address?.PostalCode ?? "",
```

Do this check for **every** destination property in **every** mapping pair from your
discovery table, not just the ones with obvious naming — it's exactly the mappings that
look "simple" (§3.1, no explicit `ForMember` at all) that are most likely relying
entirely on this invisible convention.

### 3.15 `ProjectTo<TDest>()` on an `IQueryable<TSource>` — LINQ-to-SQL projection

**Fundamentally different from every other pattern in this guide.** `ProjectTo` builds
an expression tree that EF Core's (or another ORM's) query provider translates to SQL —
it runs *before* materialization, inside the database. A compiled method call
(`MapFrom`/`ToEntity`/anything from §3.1–3.13) is opaque to that translator; calling it
inside a `.Select()` throws `InvalidOperationException: could not be translated` or
silently forces full materialization first (a serious, easy-to-miss performance
regression — pulling the whole table into memory before filtering).

**Replacement:** an explicit `.Select()` projection written directly in the query:

```csharp
// was: query.ProjectTo<PersonDto>(mapperConfig)
var results = query.Select(p => new PersonDto
{
    Id = p.Id,
    FullName = p.FirstName + " " + p.LastName, // SQL-translatable string concat
}).ToList();
```

If the same projection shape is used in multiple queries, extract a static
`Expression<Func<TSource,TDest>>` (an expression tree, not a compiled delegate) so every
call site shares one definition and one test (see §6):

```csharp
public static class PersonProjections
{
    public static Expression<Func<Person, PersonDto>> ToDto => p => new PersonDto
    {
        Id = p.Id,
        FullName = p.FirstName + " " + p.LastName,
    };
}

// call site: query.Select(PersonProjections.ToDto)
```

Only use expressions EF Core (or your ORM) can translate to SQL inside the projection
body — no calls to arbitrary C# methods, no string formatting helpers beyond what the
provider explicitly supports. If a projection genuinely needs code that can't run in
SQL, materialize first (`.AsEnumerable()`/`.ToList()`) then map in memory with the normal
compiled-method patterns — accept that performance tradeoff consciously rather than by
accident.

### 3.16 Global configuration options

`AllowNullCollections`, `AllowNullDestinationValues`, `ForAllMaps(...)`,
`ForAllPropertyMaps(...)`, naming-convention settings — these apply invisibly to *every*
mapping in the profile/configuration, not just one pair. Re-derive their effect
explicitly, per pair, wherever they'd have mattered. For example, if
`AllowNullCollections = true` was set (meaning a null source collection maps to an empty
destination collection rather than a null one — AutoMapper's default is actually the
reverse, `false`, so check which your codebase had):

```csharp
Items = source.Items?.Select(i => LineEntity.MapFrom(i)).ToList() ?? [],
```

Don't assume "it just works" — write out what the global option actually did, once per
affected pair.

---

### Quick-reference: trigger → pattern

| What you found in the old `CreateMap<>()` chain | Apply |
| --- | --- |
| Nothing (bare `CreateMap<A,B>()`) | §3.1, then check §3.14 for hidden flattening |
| `.Ignore()` | §3.2 |
| `.MapFrom(s => expr)` | §3.3 |
| `.MapFrom<Resolver>()` | §3.4 (→ §5.1 if the resolver has DI dependencies) |
| `.ReverseMap()` | §3.5 |
| `.ConvertUsing(...)` | §3.6 |
| `.ConstructUsing(...)` | §3.7 |
| `.Condition(...)` | §3.8 |
| `.NullSubstitute(...)` | §3.9 |
| `Map(source, existing)` two-arg call site | §3.10 |
| `CreateMap<T,T>()` | §3.11 |
| Nested object property | §3.12 |
| Nested collection property | §3.13 |
| Property name looks flattened, no `ForMember` | §3.14 |
| `.ProjectTo<T>()` / `IQueryable` `.Select` misuse | §3.15 |
| `AllowNull*`/`ForAllMaps`/naming convention settings | §3.16 |
| `.Include<Derived,DerivedDto>()` | §5.2 |
| `PreserveReferences()` | §5.3 |

### ✅ GATE 2

Every row in your discovery table has explicit replacement code written for every
pattern # recorded against it. Every destination property in every pair has been
explicitly assigned somewhere, or is recorded as unmapped-by-design (§3.2's list). Every
§3.14 candidate has been checked against the source type's nested structure, not assumed
either way.

---
