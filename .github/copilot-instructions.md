# Key Vault Acmebot - Copilot Instructions

## Architecture Overview

This is an **Azure Functions v4 (.NET 6)** application that automates ACME certificate issuance and renewal using **Azure Durable Functions**. The core workflow orchestrates DNS-01 challenges through multiple DNS providers and stores certificates in Azure Key Vault.

### Key Components

- **Orchestrator Functions**: `SharedOrchestrator.IssueCertificate()` coordinates the entire ACME workflow
- **Activity Functions**: Defined in `ISharedActivity` interface, implemented in `SharedActivity` class
- **DNS Providers**: Plugin architecture with `IDnsProvider` interface supporting 11+ providers (Azure DNS, Cloudflare, Route53, etc.)
- **Configuration**: Centralized in `AcmebotOptions` with provider-specific nested options

## Critical Patterns

### Durable Functions Architecture
```csharp
// Orchestrator calls activities in sequence with retries and delays
var orderDetails = await activity.Order(certificatePolicy.DnsNames);
await context.CreateTimer(context.CurrentUtcDateTime.AddSeconds(propagationSeconds), CancellationToken.None);
```

**Activity Pattern**: Each step is an isolated `[FunctionName]` method that can be retried independently.

### DNS Provider Pattern
All DNS providers implement `IDnsProvider` with:
- `Name`: Human-readable provider name
- `PropagationSeconds`: DNS propagation delay (varies by provider)
- `ListZonesAsync()`: Discover manageable zones
- `CreateTxtRecordAsync()` / `DeleteTxtRecordAsync()`: ACME challenge management

### Configuration Pattern
Options classes follow naming convention: `{Provider}Options` (e.g., `AzureDnsOptions`, `CloudflareOptions`)
All registered in `Startup.cs` using the TryAdd extension pattern:
```csharp
dnsProviders.TryAdd(options.AzureDns, o => new AzureDnsProvider(o, environment, credential));
```

### Function Naming Convention
Functions use interpolated naming: `$"{nameof(ClassName)}_{nameof(MethodName)}"` to ensure uniqueness across orchestrators and activities.

## Development Workflows

### Local Development
```bash
# Build and run locally (uses tasks.json)
dotnet build
func host start --cwd KeyVault.Acmebot/bin/Debug/net6.0
```

### Deployment
- **Bicep Template**: `azuredeploy.bicep` provisions Function App, Key Vault, and dependencies
- **ARM Template**: `azuredeploy.json` (compiled from Bicep)
- **Publishing**: Uses `dotnet publish` task, outputs to `bin/Release` structure

### Project Structure Rules
- **Functions/**: All Azure Functions (orchestrators, activities, HTTP triggers)
- **Providers/**: DNS provider implementations
- **Options/**: Configuration classes with validation attributes
- **Models/**: DTOs for ACME and certificate data
- **Internal/**: Utilities, extensions, and cross-cutting concerns

## Key Integration Points

### ACME Protocol
Uses **ACMESharpCore** library (referenced as submodule in `ACMESharpCore/`). The `AcmeProtocolClientFactory` manages ACME client lifecycle with account persistence.

### Azure Services Integration
- **Key Vault**: Certificate storage and CSR generation via `CertificateClient`
- **Identity**: Uses `DefaultAzureCredential` for Azure authentication
- **DNS**: Resource Manager SDKs for Azure DNS management

### Webhook Integration
Supports Slack and Teams webhooks with custom payload builders determined by webhook URL:
```csharp
if (options.Webhook.Host.EndsWith("hooks.slack.com", StringComparison.OrdinalIgnoreCase))
    return new SlackPayloadBuilder();
```

## Code Quality Standards

- **Validation**: All options classes use `[Required]` and `[Range]` attributes
- **Logging**: Japanese comments in orchestrator for workflow clarity
- **Error Handling**: Custom exception types (`RetriableActivityException`, `PreconditionException`)
- **Formatting**: Uses `dotnet format` with specific exclusions for ACMESharpCore

## Testing & Debugging

**No unit tests present** - testing relies on integration testing in Azure environment.
For local debugging:
1. Configure `local.settings.json` with Key Vault and DNS provider settings
2. Use Azurite for storage emulation
3. Monitor Application Insights for telemetry

## Configuration Essentials

Required settings in `AcmebotOptions`:
- `Endpoint`: ACME CA endpoint (Let's Encrypt, Buypass, etc.)
- `VaultBaseUrl`: Target Key Vault URL
- `Contacts`: Email for ACME account registration
- At least one DNS provider configuration block

Providers are auto-discovered and registered if their options are configured.
