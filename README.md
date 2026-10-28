# Backwork .NET SDK

[![CI](https://github.com/tylergibbs1/backwork-dotnet/actions/workflows/ci.yml/badge.svg)](https://github.com/tylergibbs1/backwork-dotnet/actions/workflows/ci.yml)

Official .NET client for the [Backwork API](https://backworkhealth.com): Medicare coverage policies, medical code intelligence, prior authorization checks, claim validation, compliance review, and drug formulary evidence.

The SDK targets .NET Standard 2.0 and is compatible with .NET Framework 4.6.1+, .NET Core 2.0+, and modern .NET releases.

## Installation

```bash
dotnet add package Backwork.SDK --version 2.0.0
```

If NuGet.org doesn't list `Backwork.SDK` yet, reference the project from a clone instead:

```bash
git clone https://github.com/tylergibbs1/backwork-dotnet.git
dotnet add reference backwork-dotnet/src/Backwork.SDK/Backwork.SDK.csproj
```

Get an API key from the [Backwork dashboard](https://backworkhealth.com/dashboard).

## Quick start

```csharp
using Backwork.SDK;

using var client = new BackworkClient("bwk_live_YOUR_API_KEY");

var code = await client.LookupCodeAsync(
    code: "76942",
    include: new[] { "rvu", "policies" }
);

Console.WriteLine(code.Data?.Description);

var priorAuth = await client.CheckPriorAuthAsync(
    procedureCodes: new[] { "76942" },
    diagnosisCodes: new[] { "M54.5" },
    state: "TX",
    payer: "medicare"
);

Console.WriteLine(priorAuth.Data?.PaRequired);
```

## Core workflows

### Code lookup

```csharp
var result = await client.LookupCodeAsync(
    code: "76942",
    codeSystem: "CPT",
    jurisdiction: "JM",
    include: new[] { "rvu", "policies", "rates" },
    fuzzy: true
);
```

### Policy search and retrieval

```csharp
var policies = await client.ListPoliciesAsync(
    query: "ultrasound guidance",
    mode: "keyword",
    policyType: "LCD",
    status: "active",
    limit: 25
);

var policy = await client.GetPolicyAsync(
    "L33831",
    include: new[] { "criteria", "codes" }
);
```

### Prior authorization and claim validation

```csharp
var priorAuth = await client.CheckPriorAuthAsync(
    procedureCodes: new[] { "76942" },
    diagnosisCodes: new[] { "M54.5" },
    state: "TX",
    payer: "medicare"
);

var claim = await client.ValidateClaimWithDateOfServiceAsync(
    procedureCodes: new[] { "99213" },
    diagnosisCodes: new[] { "E11.9" },
    payer: "Medicare",
    state: "TX",
    dateOfService: "2026-05-23"
);

Console.WriteLine($"{claim.Data?.CoverageStatus} {claim.Data?.DenialRisk}");
Console.WriteLine(string.Join(", ", claim.Data?.Issues ?? new()));
```

### Coverage, Spending, and Compliance

```csharp
var criteria = await client.SearchCriteriaAsync(
    query: "diabetes",
    section: "indications",
    limit: 10
);
Console.WriteLine($"{criteria.Data?.FirstOrDefault()?.PolicyId}: {criteria.Data?.FirstOrDefault()?.PolicyTitle}");

var spending = await client.GetSpendingByCodeAsync(
    codes: new[] { "T1019", "T1020" },
    year: 2023
);

var changes = await client.ListUnreviewedChangesAsync(limit: 10);
var stats = await client.GetComplianceStatsAsync();
```

### Drug formulary evidence

```csharp
var formulary = await client.SearchDrugFormularyEvidenceAsync(
    query: "ozempic",
    payer: "all",
    limit: 5
);
```

## Error handling

```csharp
using Backwork.SDK;

try
{
    var result = await client.LookupCodeAsync("76942");
}
catch (BackworkException ex) when (ex.StatusCode == 401)
{
    Console.WriteLine($"Invalid API key: {ex.Message}");
}
catch (BackworkException ex) when (ex.StatusCode == 404)
{
    Console.WriteLine($"Resource not found: {ex.Message}");
}
catch (BackworkException ex) when (ex.StatusCode == 429)
{
    Console.WriteLine($"Rate limit exceeded: {ex.Message}");
}
catch (BackworkException ex)
{
    Console.WriteLine($"Backwork API error ({ex.Code}): {ex.Message}");
}
```

## Dependency injection

```csharp
services.AddSingleton<BackworkClient>(_ =>
    new BackworkClient(configuration["Backwork:ApiKey"]));
```

## Configuration and disposal

The second constructor argument overrides the base URL (default shown):

```csharp
using var client = new BackworkClient(
    "bwk_live_YOUR_API_KEY",
    "https://backworkhealth.com/api/v1"
);
```

`BackworkClient` implements `IDisposable`. Use a `using` statement or register it with your dependency injection container.

## Development

```bash
dotnet restore
dotnet build src/Backwork.SDK/Backwork.SDK.csproj
dotnet pack src/Backwork.SDK/Backwork.SDK.csproj --configuration Release --output artifacts
```

## Release

The SDK publishes to NuGet.org as `Backwork.SDK` using NuGet Trusted Publishing.

1. Configure a NuGet trusted publishing policy for `tylergibbs1/backwork-dotnet`, workflow `release.yml`, environment `nuget`.
2. Save the NuGet profile username as the repository secret `NUGET_USER`.
3. Update `src/Backwork.SDK/Backwork.SDK.csproj` to the new package version.
4. Push a matching tag, for example `v2.0.0`.
5. The release workflow restores, builds, packs, exchanges GitHub OIDC for a short-lived NuGet API key, and pushes the `.nupkg` to NuGet.

## Support

- Documentation: https://backworkhealth.com/docs
- Issues: https://github.com/tylergibbs1/backwork-dotnet/issues
- Email: support@backworkhealth.com

## License

MIT (declared as the package license in `Backwork.SDK.csproj`).
