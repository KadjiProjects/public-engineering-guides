# Part 2 · Phase 9 — Verification and acceptance

## 9.1 The compiler is the primary verifier

What AutoMapper verified at runtime (if the test ran), the build now verifies on
every keystroke. Lock it in — add to `.editorconfig` at the solution root:

```ini
[*.cs]
dotnet_diagnostic.MW0001.severity = error
```

From now on an unmapped destination property **cannot compile**, which is
strictly stronger than `AssertConfigurationIsValid()` ever was.

## 9.2 Behavioral spot-checks (optional but recommended once)

The generated code is reviewable — for one representative entity, read the
generated mapper (IDE: Dependencies → Analyzers → Mapwright.Generator) and
confirm against the old factory:

1. Every property NOT on the ignore list is assigned.
2. Ignored properties are absent from the generated method.
3. `bool?` → `bool` sites use `GetValueOrDefault()` (AutoMapper's old
   null-to-default behavior, preserved).
4. `CopyScalars` assigns onto `target` and never touches ignored/audit members.

If the test suite has mapping round-trip tests, they should pass unchanged. If it
has none, add one per representative entity (domain → entity → domain, assert
scalar equality) — cheap and permanent.

## 9.3 Runtime smoke test

Same checklist as Part 1 §5.2 (start service, read, write, logs). Mapping
regressions would surface on the write path (create/update round-trips).

## 9.4 What you got

- The ~200-line runtime `MapperFactory` and its `AssertConfigurationIsValid()`
  test are gone; per-entity declarations of 3–5 lines each replaced them.
- Model drift is now a build error, not a silent `null` in production.
- No reflection at runtime; the generated mappers are ordinary C# you can
  breakpoint; Native AOT/trimming are no longer blocked by mapping.
- Plugin modules are untouched by Part 2 — mapping is internal to
  Persistence/Processing, so no module rebuild is needed for this part.

## ✅ GATE 8 — Part 2 acceptance

1. `dotnet build [SOLUTION] -c Release` → 0 errors with MW0001 promoted to error.
2. `dotnet test [SOLUTION]` → green.
3. Runtime smoke checklist passes.
4. `grep -ri "automapper" src tests --include="*.cs" --include="*.csproj" | grep -v obj` → empty.
5. Committed and tagged: `git tag upgrade-net10-part2-done`.

**Done.** Both parts complete: .NET 10, compile-time-verified mapping, zero
runtime mapping machinery.
