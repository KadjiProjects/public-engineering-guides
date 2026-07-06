# Part 1 · Phase 3 — The four known breakages

These four are the entire "hard" part of Part 1. Three break the build (visible);
one breaks only at runtime (invisible — do it first, while you remember).

## 3.1 SQL connection strings ⚠️ CRITICAL — compiles clean, fails at runtime

EF Core 10's `Microsoft.Data.SqlClient` defaults to `Encrypt=true`. Every
connection string that does not explicitly set `Encrypt` or
`TrustServerCertificate` **will fail at runtime** with:

```
Microsoft.Data.SqlClient.SqlException: A connection was successfully established
with the server, but then an error occurred during the login process. ...
The certificate chain was issued by an authority that is not trusted.
```

For **every** entry in your `[CONNSTRINGS]` list — JSON config values AND strings
hardcoded in C#, test files included: if the string contains neither `Encrypt=`
nor `TrustServerCertificate=`, append:

```
TrustServerCertificate=True;
```

> `TrustServerCertificate=True` keeps the pre-upgrade behavior (encrypted when the
> server offers it, certificate not validated). The *better* long-term fix is a
> properly trusted certificate on the SQL Server and `Encrypt=True` — record that
> as an ops follow-up, but do not block the upgrade on it.

Do not forget the Serilog `MSSqlServer` sink's `connectionString` inside
`appsettings.json` — it fails silently (logging just stops) rather than loudly.

## 3.2 The email sink wrapper — API removed upstream

`Serilog.Sinks.Email` 3.0+ deleted the `EmailConnectionInfo` API. Locate the
wrapper files:

```bash
grep -rln "EmailConnectionInfo" --include="*.cs" src
```

Baseline: two files in the Utils project, `EmailSink/EmailSinkSettings.cs` and
`EmailSink/EmailSinkExtensions.cs`. **Replace the entire class body of each**,
keeping YOUR namespace:

`EmailSinkSettings.cs`:

```csharp
using MailKit.Security;
using Serilog.Events;
using Serilog.Formatting.Display;
using Serilog.Sinks.Email;
using System;
using System.Net;

namespace YourCompany.Utils.EmailSink   // keep YOUR existing namespace
{
    public class EmailSinkSettings
    {
        public string? FromEmail { get; set; }
        public string? ToEmail { get; set; }
        public string? MailServer { get; set; }
        public LogEventLevel RestrictedToMinimumLevel { get; set; } = LogEventLevel.Information;
        public string? MailSubject { get; set; }
        public bool UseDefaultCredentials { get; set; } = true;
        public int Port { get; set; } = 25;
        public bool EnableSsl { get; set; } = false;
        public bool IsBodyHtml { get; set; } = false;
        public string EmailSubject { get; set; } = "Log Email";
        public string? Username { get; set; }
        public string? Password { get; set; }
        public LogEventLevel MinimumLogLevel { get; set; } = LogEventLevel.Information;

        public EmailSinkOptions ToEmailSinkOptions()
        {
            var result = new EmailSinkOptions
            {
                From = FromEmail ?? throw new InvalidOperationException($"{nameof(FromEmail)} is required for the email sink."),
                To = [ToEmail ?? throw new InvalidOperationException($"{nameof(ToEmail)} is required for the email sink.")],
                Host = MailServer ?? throw new InvalidOperationException($"{nameof(MailServer)} is required for the email sink."),
                Port = Port,
                IsBodyHtml = IsBodyHtml,
                // EnableSsl=true previously meant STARTTLS via System.Net.Mail; false meant plain SMTP.
                ConnectionSecurity = EnableSsl ? SecureSocketOptions.StartTls : SecureSocketOptions.None,
                Subject = new MessageTemplateTextFormatter(EmailSubject),
            };
            if (!UseDefaultCredentials)
            {
                result.Credentials = new NetworkCredential(Username, Password);
            }
            return result;
        }
    }
}
```

`EmailSinkExtensions.cs`:

```csharp
using Serilog;
using Serilog.Configuration;
using System;
using System.Net.Security;
using System.Security.Cryptography.X509Certificates;

namespace YourCompany.Utils.EmailSink   // keep YOUR existing namespace
{
    public static class EmailSinkExtensions
    {
        public static LoggerConfiguration EmailSink(this LoggerSinkConfiguration loggerConfiguration, EmailSinkSettings emailSinkSettings)
        {
            if (emailSinkSettings == null) throw new ArgumentNullException(nameof(emailSinkSettings));
            var options = emailSinkSettings.ToEmailSinkOptions();
            options.ServerCertificateValidationCallback = ValidateServerCertificate;
            return loggerConfiguration.Email(
                options: options,
                restrictedToMinimumLevel: emailSinkSettings.MinimumLogLevel);
        }

        private static bool ValidateServerCertificate(object? sender, X509Certificate? certificate, X509Chain? chain, SslPolicyErrors sslPolicyErrors)
        {
            return true;
        }
    }
}
```

**Variance rules:** if your `EmailSinkSettings` has additional public properties,
keep them (they are configuration-bound and harmless). Never rename the existing
properties — `appsettings.json` binding depends on the names. If your namespace
differs, keep yours.

## 3.3 The Swashbuckle / OpenAPI namespace move

Swashbuckle 10 depends on Microsoft.OpenApi 2.x, where `OpenApiInfo` moved
namespaces. Find every affected file:

```bash
grep -rln "Microsoft.OpenApi.Models" --include="*.cs" src
```

(baseline: only `Startup.cs`). In each, replace:

```diff
-using Microsoft.OpenApi.Models;
+using Microsoft.OpenApi;
```

## 3.4 Nullability warnings surfaced by the new compiler

Two known pre-existing spots (skip any that don't exist on your copy):

**`Modules/ProviderBase.cs`** in the Application project —
`Path.GetDirectoryName` may return null:

```diff
-            _configuration = new ConfigurationBuilder()
-                .SetBasePath(System.IO.Path.GetDirectoryName(this.GetType().Assembly.Location))
+            string assemblyDirectory = System.IO.Path.GetDirectoryName(this.GetType().Assembly.Location)
+                ?? System.AppContext.BaseDirectory;
+            _configuration = new ConfigurationBuilder()
+                .SetBasePath(assemblyDirectory)
```

**`Startup.cs`** in the Service project — a `Configuration` property declared but
never assigned (CS8618). Inside the constructor, after the existing
`_configuration = configuration;` line, add:

```csharp
            Configuration = configuration;
```

Any **other** new warnings your copy produces: fix only `CS86xx` nullability
warnings, with null-coalescing or null-forgiveness at the site — nothing
structural.

## ✅ GATE 2

1. `dotnet build [SOLUTION]` → **Build succeeded, 0 errors.**
2. Every `[CONNSTRINGS]` entry carries `TrustServerCertificate=True;` or an
   explicit `Encrypt=` it already had.
3. `grep -rln "EmailConnectionInfo" --include="*.cs" src` → empty.
4. `grep -rln "Microsoft.OpenApi.Models" --include="*.cs" src` → empty.
5. `dotnet test [SOLUTION]` → same pass count as the Phase-1 baseline.
6. Committed.

Next: [04-plugin-modules-batch.md](04-plugin-modules-batch.md)
