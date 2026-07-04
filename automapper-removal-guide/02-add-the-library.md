> **AutoMapper → RECO.Mapping generic migration guide — part 3 of 9.**
> Previous: [Discovery](01-discovery.md) · Next: [Migration patterns](03-migration-patterns.md)

## 2. Add RECO.Mapping to your project

You determined your tier in [00-ground-rules.md](00-ground-rules.md) §0.4. Both tiers
start the same way, then diverge.

### 2.1 Bring in the source

Copy the folder [`../src/RECO.Mapping/`](../src/RECO.Mapping/) from this repository into
your solution — e.g. `src/Common/RECO.Mapping/` if you have a shared-libraries
convention, or directly inside an existing project's folder if you'd rather not add a
new project (both work; the code has zero package dependencies either way). The 7 files:

```
RECO.Mapping.csproj
IMappedFrom.cs
IMapsTo.cs
IScalarCopyable.cs
IDualMapped.cs
MappingExtensions.cs
Verification/MappingVerificationException.cs
Verification/MappingVerifier.cs
```

If added as its own project, wire it into the solution and reference it from every
project that used AutoMapper:

```bash
dotnet sln <your-solution>.sln add path/to/RECO.Mapping/RECO.Mapping.csproj
dotnet add <each consuming project>.csproj reference path/to/RECO.Mapping/RECO.Mapping.csproj
```

### 2.2 Interface tier (C# 11 / .NET 7+) — use everything as-is

Retarget the copied `RECO.Mapping.csproj`'s `<TargetFramework>` from `net10.0` to match
your project's actual target framework (e.g. `net8.0`, `net9.0`) — everything else in
the file (`Nullable`, `ImplicitUsings`, `LangVersion latest`) is framework-agnostic and
should stay as shipped. No other file needs modification. Proceed to
[03-migration-patterns.md](03-migration-patterns.md).

### 2.3 Fallback tier (pre-C# 11 / .NET Framework / .NET Standard) — swap 5 files

Static abstract interface members do not compile below C# 11. Do the following instead:

1. **Delete** (or simply don't add) these four files — they will not compile on your
   toolchain: `IMappedFrom.cs`, `IMapsTo.cs`, `IScalarCopyable.cs`, `IDualMapped.cs`.
2. **Replace** `MappingExtensions.cs` — the shipped version's generic constraints
   reference the interfaces you just removed. Use this delegate-based version instead:

   ```csharp
   namespace RECO.Mapping;

   /// <summary>
   /// Collection helpers over plain mapping delegates. Fallback-tier equivalent of the
   /// interface-based MapFromAll/MapToAll — takes the mapping function explicitly
   /// instead of resolving it through a static interface member, so it works on any
   /// C# version and with types you don't own.
   /// </summary>
   public static class MappingExtensions
   {
       /// <summary>Maps every element of <paramref name="sources"/> with <paramref name="map"/>.</summary>
       public static List<TDest> MapAll<TSource, TDest>(
           this IEnumerable<TSource> sources, Func<TSource, TDest> map)
           => sources.Select(map).ToList();
   }
   ```

   Call sites become `sources.MapAll(PersonMapper.ToDto)` instead of
   `sources.MapFromAll<TDest, TSource>()`. Functionally identical; just an explicit
   delegate parameter instead of a static-interface-member lookup.
3. **Keep unchanged**: `RECO.Mapping.csproj` (retarget `<TargetFramework>` to match your
   project — `<Nullable>`/`<ImplicitUsings>` may need to become `disable`/`false` if
   your target predates their availability, e.g. .NET Framework or old .NET Standard;
   check your other projects' csproj files for the convention already in use),
   `Verification/MappingVerificationException.cs`, and
   `Verification/MappingVerifier.cs` — **neither verification file uses static abstract
   members**; they use only reflection (`System.Reflection`) and a plain `Func<,>`
   delegate, both available since .NET Framework 4.5. This is the one piece of the
   library every tier gets identically.

Every mapping pair in this tier is written as plain static methods (see
[03-migration-patterns.md](03-migration-patterns.md), which gives both tiers' code side
by side for each pattern) — a static class per pair, or one static class per source
project holding all its mapping methods as extension methods, whichever matches your
codebase's existing conventions.

### ✅ GATE 1

```bash
dotnet build path/to/RECO.Mapping/RECO.Mapping.csproj
```

builds with 0 errors, using whichever variant (interface tier unmodified, or fallback
tier with the 4 interfaces removed and `MappingExtensions.cs` replaced) matches your
prerequisites from §0.4. If it doesn't build, recheck: target framework actually supports
your chosen tier; `LangVersion` is explicit and high enough for the interface tier;
`Nullable`/`ImplicitUsings` settings match what your target framework accepts.

---
