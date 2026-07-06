# Part 1 · Phase 0 — Ground rules

Read this file completely before touching anything.

## 0.1 What you are upgrading

A layered .NET 6 solution of roughly this shape (names on your copy may differ —
discovery in the next file finds the real ones):

```
src/
  Domain/          <Prefix>.Domain          — domain records (immutable, init-only)
  Common/          <Prefix>.Utils           — logging helpers incl. an email-sink wrapper
  Application/     <Prefix>.Application     — module contracts (IFilter, ISubscriber, holders)
                   <Prefix>.Filter          — filter plugin base classes
                   <Prefix>.Subscriber      — subscriber plugin base classes
  Infrastructure/  <Prefix>.Persistence     — EF Core, entities, repository, AutoMapper profile
                   <Prefix>.Processing      — ActionProcessor pipeline
                   <Prefix>.ModulesLoading  — runtime plugin loader (Assembly.LoadFile)
                   <Prefix>.PersistentQueue — EF-backed queue
  Service/         <Prefix>.Service         — ASP.NET Core host, Serilog, Swagger
tests/             *.Tests                  — NUnit test projects
```

Plus, **outside or alongside the solution**: filter/subscriber plugin module
projects — separately compiled class libraries deployed as
`Modules/<ModuleName>/<ModuleName>.dll` folders that the host loads at runtime.
There may be **hundreds**. They are handled in batch in file 04.

## 0.2 Your copy may differ from the baseline

The baseline this guide was verified against had 11 projects, 8 mapped entities,
and a handful of modules. Your copy may have more of everything. The rules:

- **Never assume a count.** Phase 01 builds work lists (`[PROJECTS]`,
  `[AUTOMAPPER-PROJECTS]`, `[ENTITIES]`, `[CONNSTRINGS]`, `[MODULES]`) from your
  actual code. Every later step iterates those lists.
- **Never assume a name.** Where this guide shows `Catalog.Persistence` or
  `CodeEntity`, substitute the name discovery found on your copy.
- **If a file this guide says to edit does not exist**, find its equivalent with
  the grep command given at that step. If nothing matches, the step does not apply
  to your copy — record that in your notes and move on.

## 0.3 Hard rules — do NOT deviate

1. **Do not upgrade AutoMapper in Part 1.** It stays at its current version
   (netstandard2.1 assemblies load fine on .NET 10). Upgrading it is pointless
   churn if Part 2 will remove it, and AutoMapper 12+ changes licensing/API.
2. **Do not change any public contract type** in the Application project
   (`IFilter`, `ISubscriber`, `IModule*`, provider base classes). Every plugin
   module compiles against these; changing them multiplies the plugin work.
3. **Do not upgrade NUnit 3.x to 4.x.** NUnit 4 renames the assertion API
   (`Assert.AreEqual` → `Assert.That`) and would turn a retarget into a rewrite.
   The verified matrix keeps NUnit 3.14.0 + NUnit3TestAdapter 6.2.0 which run fine
   on .NET 10.
4. **Do not reformat, rename, or "improve" code you are not instructed to touch.**
   The diff at the end of Part 1 should be: csproj files, connection strings, the
   email-sink wrapper, one using-directive per Swagger file, and the two
   nullability fixes. Nothing else.
5. **One phase at a time, GATE before the next.** Never continue past a failing GATE.
6. **Commit after every passing GATE** so any later mistake can be rolled back to
   a known-good point.

## 0.4 Prerequisites

- .NET 10 SDK installed: `dotnet --list-sdks` shows a `10.x` entry.
- The solution builds green on .NET 6 **before you start** (`dotnet build` at the
  solution root). If it does not, stop — fix that first; you cannot distinguish
  upgrade breakage from pre-existing breakage otherwise.
- All work happens on a fresh branch: `git checkout -b upgrade/net10`.
- A snapshot of current behavior if tests are thin: capture the Swagger JSON and a
  handful of representative API responses now, to diff after GATE 4.

## ✅ GATE — ground rules

1. `dotnet --list-sdks` lists a 10.x SDK.
2. `dotnet build` on the untouched .NET 6 solution succeeds.
3. You are on a dedicated upgrade branch.

Next: [01-discovery.md](01-discovery.md)
