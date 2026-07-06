# Part 1 · Phase 4 — Batch-upgrading the plugin modules (filters & subscribers)

## 4.1 Why plugins need special care

The host loads filter/subscriber modules **at runtime** with `Assembly.LoadFile`
from per-module folders (`Modules/<Name>/<Name>.dll`). Each module is a separate
class library implementing a provider base class from the host's contract
assemblies (the Application/Filter/Subscriber projects).

Three facts drive everything in this phase:

1. **Type identity.** A module's `ModuleProviderBase` implementation must be the
   *same type* as the host's — which means the module must resolve the contract
   assembly to the **host's copy**, not a stale one sitting in its own folder. A
   module folder containing an old .NET 6 build of the contract DLLs will load,
   and then fail `IsAssignableTo` / cast checks with the maddening
   `InvalidCastException: Unable to cast object of type 'X' to type 'X'` (same
   name twice). **Module folders must contain the module's own DLL and its
   private third-party dependencies — never the host contract assemblies.**
2. **A .NET 6-compiled module usually loads on a .NET 10 host** (assembly-level
   compat), but you get no compile-time check that it still matches the upgraded
   contracts, and its own dependencies may not resolve. Rebuild everything;
   do not ship mixed builds.
3. **At hundreds of modules, this is a pipeline, not a task.** Everything below is
   scripted and idempotent — rerunnable until the failure list is empty.

## 4.2 Batch retarget

From the root folder that contains the module source projects (adjust the path):

```powershell
# retarget-modules.ps1 — idempotent
$failures = @()
Get-ChildItem -Path .\modules-src -Recurse -Filter *.csproj | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    $new = $content -replace '<TargetFramework>net6\.0</TargetFramework>', '<TargetFramework>net10.0</TargetFramework>'
    if ($new -ne $content) {
        Set-Content $_.FullName $new -Encoding utf8
        Write-Host "retargeted: $($_.FullName)"
    }
}
```

If modules reference the host contracts as **project references**, nothing more is
needed — they now build against the upgraded projects. If they reference contracts
as **DLL references or a NuGet package**, update every reference to the new build
(one `dotnet add package <ContractsPackage> --version <new>` per project, also
scriptable from the same loop).

## 4.3 Batch build with a failure list

```powershell
# build-modules.ps1 — produces module-build-report.csv
$report = @()
Get-ChildItem -Path .\modules-src -Recurse -Filter *.csproj | ForEach-Object {
    $r = dotnet build $_.FullName -c Release --nologo -v q 2>&1
    $ok = ($LASTEXITCODE -eq 0)
    $report += [pscustomobject]@{ Module = $_.BaseName; Ok = $ok; FirstError = ($r | Select-String "error" | Select-Object -First 1) }
}
$report | Export-Csv module-build-report.csv -NoTypeInformation
$report | Where-Object { -not $_.Ok } | Format-Table
```

Triage the failures. In a fleet of near-identical modules the failures cluster:
the same missing package or the same API use repeats across dozens of modules.
**Fix one instance by hand, then script that fix across the fleet** — same
pattern as §4.2. Typical clusters and fixes are in
[troubleshooting.md](../troubleshooting.md).

## 4.4 Batch deploy layout + load smoke test

Deploy each rebuilt module to a staging `Modules/` tree, then verify the host can
actually load every one **before** production. The host's loader logs a warning
per folder it cannot load — but don't wait for production logs. Add a tiny smoke
test to the test suite (adapt namespaces to your loader project):

```csharp
[Test]
public void EveryDeployedModuleFolderLoads()
{
    var modulesRoot = TestContext.Parameters["ModulesRoot"] ?? @"C:\staging\Modules";
    var failures = new List<string>();
    foreach (var dir in Directory.GetDirectories(modulesRoot))
    {
        var name = Path.GetFileName(dir);
        var dll = Path.Combine(dir, name + ".dll");
        if (!File.Exists(dll)) { failures.Add($"{name}: no {name}.dll"); continue; }
        try
        {
            var asm = Assembly.LoadFile(dll);
            var providerType = asm.GetTypes().SingleOrDefault(t =>
                t.IsAssignableTo(typeof(YourFilterProviderBase))     // ← your contract base
                || t.IsAssignableTo(typeof(YourSubscriberProviderBase)));
            if (providerType is null) failures.Add($"{name}: no provider type found");
        }
        catch (Exception ex) { failures.Add($"{name}: {ex.GetType().Name} {ex.Message}"); }
    }
    Assert.That(failures, Is.Empty, string.Join("\n", failures));
}
```

Also verify the **negative** of §4.1 fact 1 across the whole tree in one shot:

```powershell
# No module folder may contain host contract assemblies:
Get-ChildItem C:\staging\Modules -Recurse -Include "*Application.dll","*Domain.dll" |
  Group-Object Directory | Format-Table Name
# → must produce no output
```

## 4.5 Modules with lost source

Deployed module folders with no findable source cannot be rebuilt. Options, in
order of preference: (1) locate the source (check old CI artifacts / package
feeds); (2) rewrite the module — they are typically small, one provider class +
one filter/subscriber class; (3) as a **temporary** bridge, test whether the old
.NET 6 DLL loads and functions on the .NET 10 host in staging — if it does, ship
it flagged for replacement, and if it does not, the module is out of service
until rewritten. Record every module in category (3) in the ops follow-up list.

## 4.6 Rollout strategy at fleet scale

- **Wave 0:** the host + zero modules in staging — service starts, health checks
  green, module loader logs "0 modules" cleanly.
- **Wave 1:** one representative filter + one representative subscriber. Exercise
  an end-to-end request that triggers both.
- **Wave N:** the rest in batches (alphabetical or by business area), smoke test
  (§4.4) between waves. The per-wave cost is minutes because everything is scripted.

## ✅ GATE 3

1. `module-build-report.csv` exists and has `Ok = True` for every module (or the
   exceptions are explicitly recorded under §4.5).
2. The load smoke test passes against the staging `Modules/` tree.
3. The contract-DLL-in-module-folder check (§4.4) produces no output.
4. Committed (scripts + report included).

Next: [05-verification.md](05-verification.md)
