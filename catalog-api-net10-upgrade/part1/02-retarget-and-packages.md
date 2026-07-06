# Part 1 · Phase 2 — Retarget and upgrade packages

## 2.1 Retarget every project in `[PROJECTS]`

In **every** csproj, replace

```xml
<TargetFramework>net6.0</TargetFramework>
```
with
```xml
<TargetFramework>net10.0</TargetFramework>
```

One command does all of them:

```bash
# bash / Git Bash
find src tests -name "*.csproj" -exec sed -i 's|<TargetFramework>net6.0</TargetFramework>|<TargetFramework>net10.0</TargetFramework>|' {} +
```

```powershell
# PowerShell equivalent
Get-ChildItem src,tests -Recurse -Filter *.csproj | ForEach-Object {
  (Get-Content $_.FullName) -replace '<TargetFramework>net6.0</TargetFramework>','<TargetFramework>net10.0</TargetFramework>' |
    Set-Content $_.FullName -Encoding utf8
}
```

Verify none remain: `grep -rl "net6.0" --include="*.csproj" src tests` → empty.

## 2.2 Upgrade packages — the verified version matrix

Apply to whichever of your projects contains each package. Locate them with
`grep -rn "<PackageReference" --include="*.csproj" src tests`. Projects that
contain none of these packages need nothing beyond §2.1.

| Package | Set version to | Note |
|---|---|---|
| Microsoft.EntityFrameworkCore.SqlServer | **10.0.9** | brings the `Encrypt=true` default — Phase 3 §3.1 is mandatory |
| Microsoft.EntityFrameworkCore.Design | **10.0.9** | |
| Microsoft.Extensions.Configuration | **10.0.9** | |
| Microsoft.Extensions.Configuration.FileExtensions | **10.0.9** | |
| Microsoft.Extensions.Configuration.Json | **10.0.9** | |
| Microsoft.Extensions.Configuration.Abstractions | **10.0.9** | |
| Microsoft.Extensions.Logging.Abstractions | **10.0.9** | |
| Microsoft.Extensions.Options | **10.0.9** | |
| Microsoft.Extensions.DependencyInjection | **DELETE** from the Web-SDK project only | the project whose csproj starts `<Project Sdk="Microsoft.NET.Sdk.Web">` gets it from the framework; keeping it causes NU1510. In a non-web project set 10.0.9 instead. |
| Newtonsoft.Json | **13.0.4** | |
| Swashbuckle.AspNetCore | **10.2.3** | breaks a using — Phase 3 §3.3 |
| Serilog | **4.3.1** | |
| Serilog.AspNetCore | **10.0.0** | |
| Serilog.Expressions | **5.0.0** | |
| Serilog.Extensions.Hosting | **10.0.0** | |
| Serilog.Extensions.Logging | **10.0.0** | |
| Serilog.Settings.Configuration | **10.0.1** | |
| Serilog.Sinks.Console | **6.1.1** | |
| Serilog.Sinks.Email | **4.2.1** | breaks the wrapper — Phase 3 §3.2 |
| Serilog.Sinks.File | **7.0.0** | |
| Serilog.Sinks.MSSqlServer | **10.0.0** | its connection string is also on the `[CONNSTRINGS]` list |
| Moq | **4.20.72** | |
| NUnit | **3.14.0** | 3.x — **NOT** 4.x (hard rule 3) |
| NUnit3TestAdapter | **6.2.0** | |
| Microsoft.NET.Test.Sdk | **18.7.0** | |
| **AutoMapper** | **UNCHANGED** | hard rule 1 — netstandard2.1, runs on .NET 10 as-is |

If your copy contains a package not in this table, leave its version unchanged
unless a GATE fails because of it (then see [troubleshooting.md](../troubleshooting.md)).

## 2.3 Restore check

```bash
dotnet restore [SOLUTION]
```

Restore must succeed with **zero** NU1605/NU1107 (downgrade/conflict) errors.
NU1510 (unneeded reference) means §2.2's DELETE row was skipped.

## ✅ GATE 1

1. `grep -rl "net6.0" --include="*.csproj" src tests` → empty.
2. `dotnet restore` succeeds, no NU1605/NU1107/NU1510.
3. AutoMapper's version is byte-for-byte what it was before this phase.
4. Committed: `git commit -am "Retarget to net10.0; upgrade package matrix"`.

The solution will likely **not build yet** — the email sink and Swagger breakages
are fixed in the next phase. That is expected; GATE 1 is restore-level only.

Next: [03-known-breakages.md](03-known-breakages.md)
