# Troubleshooting — error → cause → fix

Ordered roughly by when you'd hit them. Search this file for the error text.

## Part 1

| Error / symptom | Cause | Fix |
|---|---|---|
| `NU1510: PackageReference Microsoft.Extensions.DependencyInjection will not be pruned` | Web SDK project explicitly references a framework-provided package | Delete that PackageReference from the Web-SDK csproj (Phase 2 matrix DELETE row) |
| `NU1605` / `NU1107` package downgrade or version conflict on restore | One project pinned an old Microsoft.Extensions.* / Serilog version the matrix missed | Grep the package name across all csproj; align every occurrence to the matrix version |
| `error CS0234: The type or namespace name 'Models' does not exist in the namespace 'Microsoft.OpenApi'` | Swashbuckle 10 / Microsoft.OpenApi 2.x namespace move | Phase 3 §3.3 — `using Microsoft.OpenApi.Models` → `using Microsoft.OpenApi` |
| `error CS0246: The type or namespace name 'EmailConnectionInfo' could not be found` | Serilog.Sinks.Email 3.0+ removed the API | Phase 3 §3.2 — replace both wrapper files with the provided source |
| `SqlException: ...certificate chain was issued by an authority that is not trusted` at runtime | Microsoft.Data.SqlClient `Encrypt=true` default | Phase 3 §3.1 — `TrustServerCertificate=True;` on every `[CONNSTRINGS]` entry |
| Service runs but the SQL log table stops getting rows | The Serilog MSSqlServer sink's own connection string missed §3.1 | Same fix, in the sink's `connectionString` inside `appsettings.json` |
| `CS8604 / CS8618` warnings new after retarget | .NET 10 compiler + newer nullable analysis on old code | Phase 3 §3.4 patterns; fix only at the warned site |
| `InvalidCastException: Unable to cast object of type 'X' to type 'X'` (same name!) when loading a module | The module folder contains its own stale copy of a host contract DLL — two assemblies with the same identity in play | Delete host contract DLLs from every module folder; module folders carry only the module DLL + its private deps (Phase 4 §4.1/§4.4) |
| `Could not load file or assembly ... The located assembly's manifest definition does not match` | Module built against old contract version | Rebuild the module against the upgraded solution (Phase 4 §4.2–4.3) |
| Module loader logs `Could not find module dll in directory: <name>` | Folder/DLL name mismatch — loader expects `Modules/<Name>/<Name>.dll` | Rename the deployed folder or the assembly to match |
| `BadImageFormatException` on module load | Native/x86 dependency, or corrupted DLL | Check the module's PlatformTarget; rebuild AnyCPU unless it wraps native code |
| Tests fail with `Method not found` after retarget | A test project still pins an old package (often Microsoft.NET.Test.Sdk) | Align to matrix: Test.Sdk 18.7.0, NUnit 3.14.0, adapter 6.2.0 |
| App starts, Swagger 404 | Hosting bundle mismatch on server — service runs Kestrel-only | Install the ASP.NET Core 10 hosting bundle on the server (Part 1 §5.3) |

## Part 2

| Error / symptom | Cause | Fix |
|---|---|---|
| `MW0005: not a recognized mapping shape` | Signature deviates (instance method, wrong param count, generic method, nested class) | Match §6.2 shapes exactly: `static partial` on a top-level non-generic `[Mapper]` class |
| A flood of `MW0001` on first build | Expected — that's the work list | Phase 7 §7.4: transcribe the old ignore lists; each warning is one decision |
| `MW0003: 'X' does not exist on 'Y'` | Typo in transcription, or the old factory ignored a property that has since been removed | Fix the `nameof` — note MW0003 catching a stale entry is the feature working |
| `MW0004: no conversion` between two same-named properties | Types genuinely differ (e.g. `string` vs `int`) | Old AutoMapper either had a converter (port to `[AfterMap]`) or silently did something surprising — investigate before porting |
| `MW0006` on `CopyScalars` | The mutable entity has an init-only property | Add it to that declaration's `[MapIgnore]` and set it at construction instead |
| `MW0008: collection map has no element map` | `ToEntities` declared but `ToEntity` missing/renamed in the same class | Declare the single-object map with exactly the element types |
| `MW0009` | `[AfterMap]` method wrong signature or on a projection | `static void Name(TSource, TDest)` in the same class; never on `Expression<...>` shapes |
| `CS0103: The name 'CodeMapper' does not exist` at a construction site | Missing `using` for the Mapping namespace at the factory/DI file | Add the using; mapper classes are `internal` to Persistence by design |
| `CS7036: There is no argument given that corresponds to...` on `new Repository<...>` | Construction site not yet updated to the new delegate arity | Phase 8 §8.4 — the compiler is listing your remaining work sites |
| Generated file not visible | No `EmitCompilerGeneratedFiles` | Phase 7 §7.2 note — IDE Analyzers node shows it without any csproj change |
| Runtime `NullReferenceException` in a mapped flow that used to work | A property the old factory mapped is now on an ignore list (over-transcription) | Diff the generated method's assignments against the old factory's non-ignored members (Phase 9 §9.2 checklist) |
