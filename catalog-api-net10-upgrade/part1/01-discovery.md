# Part 1 · Phase 1 — Discovery: build YOUR work lists

Everything later iterates over the lists you produce here. Run each command from
the solution root and record the output verbatim in a scratch file
(`UPGRADE-WORKLISTS.md`) that you keep updating as you work.

## 1.1 `[SOLUTION]` — the solution file

```bash
ls *.sln
```

## 1.2 `[PROJECTS]` — every project that gets retargeted

```bash
find src tests -name "*.csproj" | sort
```

All of them get retargeted in the next phase — no exceptions.

## 1.3 `[AUTOMAPPER-PROJECTS]` — where AutoMapper is referenced vs. actually used

```bash
# csproj references:
grep -rln "AutoMapper" --include="*.csproj" src tests

# actual code usage:
grep -rln "using AutoMapper" --include="*.cs" src tests | grep -v obj
```

Expect the code-usage list to be **shorter** than the reference list (baseline:
Persistence and Processing use it; Domain and Application referenced it without
using it — dead references). Record both lists. Part 1 leaves all of this alone;
Part 2 consumes these lists.

## 1.4 `[ENTITIES]` — the mapping surface (needed in Part 2, cheap to capture now)

Find the AutoMapper configuration class:

```bash
grep -rln "MapperConfiguration\|CreateMap" --include="*.cs" src | grep -v obj
```

Open the file (baseline name: `MapperFactory.cs` in the Persistence project) and
list every **distinct entity name** appearing in `cfg.CreateMap<...>()` calls.
Baseline had 8: `Code, CodeExternalRel, CodeRel, CodeType, CodeTypeRel, DataValue,
DataValueType, Synonym` (yours will differ — use YOUR names). Each appears as a
domain-record/persistence-entity pair, typically with 3 maps per entity.

## 1.5 `[CONNSTRINGS]` — every SQL connection string ⚠️ most important list

Both configuration **and** hardcoded in C# (test files included):

```bash
grep -rn "Data Source=\|Server=\|Initial Catalog=" --include="*.json" --include="*.cs" src tests | grep -v obj
```

Baseline sites: two app connection strings + one Serilog `MSSqlServer` sink
`connectionString` in the Service project's `appsettings.json`, and one hardcoded
string in a queue test class. Yours may have more — **every single one** matters
(Phase 3 explains why).

## 1.6 `[MODULES]` — the plugin module projects (filters & subscribers)

Plugin modules are class libraries implementing the provider base class from the
Filter/Subscriber projects. They may live in this repository, sibling
repositories, or a dedicated modules repository. Find local candidates:

```bash
# projects referencing the Filter or Subscriber project/package:
grep -rln "\.Filter\.csproj\|\.Subscriber\.csproj" --include="*.csproj" .. 2>/dev/null

# deployed module folders on a server/staging copy (one folder per module):
ls <path-to-deployed-service>/Modules/
```

Record: the list of module **source projects** you can find, and the list of
deployed module **folder names**. If some deployed modules have no source you can
locate, flag them now — they are the long pole (file 04, §4.5).

## 1.7 Sanity numbers

Record for later comparison:

```bash
dotnet build [SOLUTION]          # must be green (still .NET 6)
dotnet test [SOLUTION] --nologo  # record pass/total
```

## ✅ GATE 0

1. `UPGRADE-WORKLISTS.md` contains all six lists with real values.
2. `[PROJECTS]` is non-empty and every path exists.
3. `[CONNSTRINGS]` has at least one entry (a system of this shape always has one;
   an empty list means the grep missed a pattern — try `ConnectionString` as a
   search term too).
4. Baseline build green, baseline test count recorded.

Next: [02-retarget-and-packages.md](02-retarget-and-packages.md)
