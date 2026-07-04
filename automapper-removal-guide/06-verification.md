> **AutoMapper → RECO.Mapping generic migration guide — part 7 of 9.**
> Previous: [Special cases](05-special-cases.md) · Next: [Troubleshooting](07-troubleshooting.md)

## 6. Verification — one test per mapping pair, per direction

`MappingVerifier.AssertAllMembersMapped<TSource,TDest>(map, params unmappedByDesign)` is
the direct replacement for AutoMapper's `configuration.AssertConfigurationIsValid()`. It
takes any `Func<TSource,TDest>` — it does not require `TSource`/`TDest` to implement any
`RECO.Mapping` interface, so it works identically for both library tiers and for
DI-dependent mapping services (§5.1). Add the test project's reference to
`RECO.Mapping` (and, if separate, `RECO.Mapping.Verification`'s namespace — it lives in
the same project) alongside your existing test framework (NUnit, xUnit, MSTest — the
verifier throws a plain exception on failure and doesn't care which framework asserts on
it).

### 6.1 The basic test shape

One test per **direction**, per pair — construction and, if `ReverseMap`/`IDualMapped`
is used, the reverse projection too:

```csharp
[Test]
public void PersonDto_maps_to_PersonEntity_completely()
{
    MappingVerifier.AssertAllMembersMapped<PersonDto, PersonEntity>(
        PersonEntity.MapFrom,                  // or dto => dto.ToEntity() on the fallback tier
        /* unmappedByDesign: */ nameof(PersonEntity.Tags));
}

[Test]
public void PersonEntity_maps_to_PersonDto_completely()
{
    MappingVerifier.AssertAllMembersMapped<PersonEntity, PersonDto>(e => e.MapTo());
}
```

The `unmappedByDesign` list **must exactly match** the members you recorded as ignored
in [03-migration-patterns.md](03-migration-patterns.md) §3.2 for that pair and direction
— cross-check against your discovery table. A mismatch here is the test catching either
a forgotten mapping or a stale ignore-list entry; see
[07-troubleshooting.md](07-troubleshooting.md) for both failure messages.

### 6.2 In-place update pairs (§3.10)

Wrap `CopyScalarsTo` as a construction-shaped delegate for the verifier (which always
maps `TSource → TDest` by construction) by copying onto a fresh baseline instance:

```csharp
[Test]
public void CopyScalarsTo_covers_every_scalar_member()
{
    MappingVerifier.AssertAllMembersMapped<PersonEntity, PersonEntity>(
        source => { var target = new PersonEntity(); source.CopyScalarsTo(target); return target; },
        nameof(PersonEntity.Orders)); // the navigation property CopyScalarsTo must NOT touch
}
```

This simultaneously proves two things: every scalar is covered (main assertion) and the
navigation property is untouched (it's in `unmappedByDesign`, so the verifier confirms
it stayed at its constructor default rather than picking up the source's value).

### 6.3 DI-dependent mapping services (§5.1)

Construct the service with test doubles, pass its instance method:

```csharp
[Test]
public void PersonMappingService_covers_every_member()
{
    var service = new PersonMappingService(new FakeCodeLookupService(), FakeTimeProvider());
    MappingVerifier.AssertAllMembersMapped<PersonEntity, PersonDto>(service.ToDto);
}
```

### 6.4 `ProjectTo`-replaced LINQ expressions (§3.15)

`MappingVerifier` takes a compiled delegate, not an expression tree — compile the
expression inside the test to prove the *projection definition* is complete (this does
not test SQL translation; that's what your existing integration tests against a real or
in-memory provider are for):

```csharp
[Test]
public void PersonProjection_covers_every_destination_member()
    => MappingVerifier.AssertAllMembersMapped<Person, PersonDto>(PersonProjections.ToDto.Compile());
```

### 6.5 Inheritance dispatch (§5.2)

Add one coverage test per concrete derived pair (each `DerivedDto.MapFrom(Derived)` is
just another mapping pair, tested exactly like §6.1), plus — if you built one — the
reflection-based "every subtype has a switch case" test suggested in §5.2.

### ✅ GATE 4 — acceptance

1. Every row in your [discovery](01-discovery.md) table has at least one
   `MappingVerifier`-based test (both directions, if bidirectional).
2. All such tests pass.
3. Every `unmappedByDesign` list matches the ignored-members list you recorded in §3.2,
   pair for pair, direction for direction.
4. Every §3.14 flattened property and every §1.4 `ProjectTo` call site was explicitly
   handled — check this against your discovery table one more time; a verifier test
   only proves what it was told to check, not that you didn't forget an entire row.
5. Full solution builds with no new warnings introduced by this migration.
6. Any pre-existing functional/integration tests that exercised mapped data end-to-end
   still pass, unmodified — this is your genuine behavioral-regression signal.
   `MappingVerifier` proves *coverage* (every member got assigned something), not
   *correctness* (the right value) — that's what these existing tests, and manual review
   of every §3.3/§3.4/§3.14 custom expression against the original AutoMapper
   configuration, are for.

---
