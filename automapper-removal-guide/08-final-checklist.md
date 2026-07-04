> **AutoMapper → RECO.Mapping generic migration guide — part 9 of 9 (final).**
> Previous: [Troubleshooting](07-troubleshooting.md) · Back to [README](README.md)

## 8. Final checklist and change summary

Run through this as the single acceptance pass. It summarizes every GATE from
§0 through §6 — if any line is unchecked, the migration is not done, regardless of
whether the build succeeds (a clean build proves the code compiles, not that every
mapping was migrated).

### Acceptance checklist

- [ ] **Discovery (§1) is complete**: every `CreateMap<>()`, every call site, every
      global option is recorded in your work-list table.
- [ ] **Library is in place (§2)**: correct tier chosen per project; `RECO.Mapping`
      (or its fallback variant) builds with 0 errors.
- [ ] **Every mapping pair has explicit replacement code (§3)**: every destination
      member is assigned somewhere, or is a recorded, deliberate unmapped-by-design
      entry. Every §3.14 flattening candidate was individually checked, not assumed.
- [ ] **Every call site rewritten (§4)**: `grep -rn "AutoMapper\|IMapper\b"
      --include="*.cs" --include="*.csproj" .` returns nothing in the target codebase.
- [ ] **Every special case resolved by conscious decision, not guesswork (§5)**:
      DI-dependent mappings are services, not statics; polymorphic mappings dispatch
      explicitly; reference cycles are broken deliberately.
- [ ] **Every mapping pair has a passing `MappingVerifier` test, both directions where
      applicable (§6)**, with `unmappedByDesign` lists matching your §3.2 records
      exactly.
- [ ] **Solution builds with no new warnings.**
- [ ] **Pre-existing functional/integration tests covering mapped data still pass,
      unmodified** — your genuine behavioral-regression signal, since the verifier
      proves coverage, not value-correctness.
- [ ] **AutoMapper package reference is gone from every project** that had it.

### Change-summary template (for the PR/commit description)

```
Replace AutoMapper with RECO.Mapping

- Migrated N mapping pairs across M projects (list: <Source→Dest, Source2→Dest2, ...>).
- Library tier used: [interface / fallback / mixed — specify per project if mixed].
- K mapping pairs required special handling: [DI-dependent services / inheritance
  dispatch / cycle-breaking / ProjectTo replacement — list which and why].
- Added P new MappingVerifier-based tests (Q before this change, Q+P after).
- Flattened properties found and made explicit: [list, or "none found"].
- AutoMapper package reference removed from: [project list].
```

Fill in the bracketed values from your own discovery table and test run — this summary
is the record a reviewer (human or a future LLM) needs to trust the migration without
re-deriving your discovery work from scratch.

---

This is the end of the AutoMapper removal guide. Return to the [README](README.md) for
the file map, or to the repository root for the RecoAPI-specific .NET 6 → .NET 10
upgrade playbook (a separate, independent document).
