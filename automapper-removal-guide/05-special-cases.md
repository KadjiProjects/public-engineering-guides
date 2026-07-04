> **AutoMapper → RECO.Mapping generic migration guide — part 6 of 9.**
> Previous: [Rewrite call sites](04-rewrite-call-sites.md) · Next: [Verification](06-verification.md)

## 5. Special cases — slow down and think, don't mechanically apply

The patterns in [03-migration-patterns.md](03-migration-patterns.md) are mechanical
translations. The cases below are not — each requires a design decision. Flag these to
whoever owns the codebase rather than guessing; a wrong guess here is silent and
expensive precisely because it usually still compiles and often still runs.

### 5.1 DI-dependent mappings (custom value resolver with injected dependencies)

A value resolver that needs a service — a code-lookup table, the current user, a clock,
an external API client — cannot become a static method (§3.4 assumed no dependencies;
this is the case where it has some). Convert it to an **injectable mapping service**
instead:

```csharp
public sealed class PersonMappingService(ICodeLookupService lookup, TimeProvider clock)
{
    public PersonDto ToDto(PersonEntity entity) => new()
    {
        Id = entity.Id,
        CountryName = lookup.NameFor(entity.CountryCode),
        SnapshotTakenUtc = clock.GetUtcNow(),
    };
}
```

Register it in DI at the same lifetime its dependencies need (usually `Scoped`, matching
`ICodeLookupService`'s own lifetime — check, don't assume). It's still verified the same
way as everything else (see [06-verification.md](06-verification.md)) by constructing
the service with test doubles for its dependencies and passing its method as the
`Func<TSource,TDest>`:

```csharp
var service = new PersonMappingService(fakeLookup, fakeClock);
MappingVerifier.AssertAllMembersMapped<PersonEntity, PersonDto>(service.ToDto);
```

This is the correct outcome — not every mapping can be a pure static function, and
`MappingVerifier` doesn't require one; it only needs a delegate.

### 5.2 Inheritance / polymorphic mapping — `.Include<Derived, DerivedDto>()`

AutoMapper resolves the concrete runtime type automatically at map time. There is no
equivalent automatic mechanism here — write one mapping method per concrete pair (a
`DerivedDto.MapFrom(Derived)` alongside the base `BaseDto.MapFrom(Base)`), then dispatch
explicitly at the call site with a type switch:

```csharp
BaseDto dto = source switch
{
    Derived1 d1 => Derived1Dto.MapFrom(d1),
    Derived2 d2 => Derived2Dto.MapFrom(d2),
    _ => BaseDto.MapFrom(source),
};
```

If new derived types are added over time, this switch is the one place that must be
kept in sync — consider a test that enumerates all subtypes of `Base` via reflection
(test-time only, mirroring how `MappingVerifier` uses reflection) and asserts each has a
case in the switch, so a forgotten case fails a test instead of silently falling through
to the base mapping at runtime.

### 5.3 Circular / self-referencing object graphs — `PreserveReferences()`

AutoMapper's `PreserveReferences()` detects reference cycles at runtime and reuses the
already-mapped instance instead of recursing infinitely. Hand-written recursive
`MapFrom`/`MapTo` calls have no such protection — a true cycle (`Person.Manager.Reports`
containing the original `Person` again) will stack-overflow.

This needs a genuine design decision, not a mechanical fix:
- **Most common resolution:** map the back-reference as an identifier only
  (`ManagerId = source.Manager?.Id`), not the full nested object. This is usually what
  the data actually needs and avoids the cycle entirely.
- **If the full nested object is genuinely required** in both directions: cap the
  recursion depth explicitly (an optional `int depth = 0` parameter threaded through,
  returning `null`/an ID-only stub past a fixed limit), or maintain your own
  already-visited set for the duration of one mapping call (a `Dictionary<object,object>`
  identity map passed through recursive calls) — essentially reimplementing what
  `PreserveReferences()` did, deliberately, because your codebase turned out to need it.

Flag every `PreserveReferences()` hit from discovery and resolve each one individually —
don't apply a blanket pattern across all of them; the right answer depends on what the
cyclic data actually means in each case.

### 5.4 Global naming conventions beyond flattening

If discovery (§1.6) found `SourceMemberNamingConvention`/`DestinationMemberNamingConvention`
set to something other than the default (e.g. mapping `snake_case` source properties —
common when the source model came from a JSON API or a legacy database — to `PascalCase`
destination properties), this is a **more aggressive** version of the flattening problem
in §3.14: property name matching happened via a convention transform, not identity. Treat
every destination property as needing explicit verification against the source's actual
member names (not just its structure) — the naming convention makes the "obviously
matches" heuristic in §3.14 unreliable, so check every property, not just the
suspicious-looking ones.

---
