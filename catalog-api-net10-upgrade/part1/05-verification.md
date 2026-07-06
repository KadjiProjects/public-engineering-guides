# Part 1 · Phase 5 — Verification and acceptance

## 5.1 Full build + tests

```bash
dotnet build [SOLUTION] -c Release     # 0 errors; warnings only pre-existing ones
dotnet test  [SOLUTION] --nologo       # pass count == Phase-1 baseline
```

## 5.2 Runtime smoke test (the part the compiler cannot check)

Phase 3 §3.1 was about a **runtime-only** failure; this is where it would bite.
Start the service against a real (staging) database:

```bash
dotnet run --project src/Service/<ServiceProject>
```

Checklist:

1. Service starts; no `SqlException` in the first 30 seconds of logs.
2. Swagger UI loads (`/swagger`) and shows the same endpoints as the pre-upgrade
   snapshot from Phase 0.
3. One read endpoint returns data (proves the main connection string).
4. One write endpoint round-trips (proves EF Core 10 against the real schema).
5. The Serilog SQL sink writes rows (proves the sink connection string — check the
   log table timestamp advances).
6. If feasible, trigger an error path that sends email (proves the rewritten
   email sink against the real SMTP host).
7. Module loader log line reports the expected module count from Phase 4.

If tests are thin, diff the representative API responses captured in Phase 0
against the same calls now.

## 5.3 Deployment follow-ups — record for the operators

- Server needs the **ASP.NET Core 10 runtime / hosting bundle** (not just the
  .NET Runtime).
- `TrustServerCertificate=True` is a compatibility bridge; schedule proper SQL
  Server certificates and flip to `Encrypt=True` later.
- Old .NET 6 module DLLs kept as §4.5 bridges are technical debt with a name and
  an owner.

## ✅ GATE 4 — Part 1 acceptance

1. Release build: 0 errors.
2. Tests: pass count == baseline.
3. All seven smoke checks in §5.2 pass.
4. `grep -rl "net6.0" --include="*.csproj"` across solution **and module sources** → empty.
5. Committed and tagged: `git tag upgrade-net10-part1-done`.

**Part 1 is complete.** The system is fully on .NET 10 with AutoMapper still in
place. Part 2 is optional and can start any time from this tag:
[../part2/06-install-mapwright.md](../part2/06-install-mapwright.md)
