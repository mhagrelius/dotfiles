---
name: working-with-aspire
description: Use when building distributed apps with Aspire 13+; when orchestrating .NET, JavaScript, Python, or polyglot services; when environment variables or service discovery aren't working; when migrating from .NET Aspire 9 or Community Toolkit; when seeing AddNpmApp deprecated errors; when OTEL not appearing in dashboard; when ports change on restart breaking OAuth; when configuring MCP server for AI assistants
---

# Working with Aspire

Aspire 13 (late 2025) rebranded from ".NET Aspire" to just "Aspire" and became a polyglot platform. JavaScript and Python are first-class citizens alongside .NET. Aspire can also orchestrate Rust, Go, Java, and any containerized service.

**Core principle:** Aspire orchestrates your distributed app's development experience. Get the app model right, and environment variables, service discovery, telemetry, and containerization follow.

> **For language-specific integrations** (.NET, JavaScript, Python, MAUI) including OpenTelemetry setup and full stack examples, see `app-types.md` in this skill directory.

## When to Use

- Building distributed apps with multiple services (.NET, JavaScript, Python, Go, Rust)
- Orchestrating local development with databases, caches, message queues
- Setting up OpenTelemetry observability across polyglot services
- Migrating from .NET Aspire 8/9 or Community Toolkit extensions
- Debugging environment variable or service discovery issues
- Adding mobile app development with MAUI + emulators

## When NOT to Use

- Single-service apps (just use standard hosting)
- Production deployment only (Aspire is dev-focused; use `azd` or k8s for prod)
- Projects not using .NET AppHost (Aspire requires a C# orchestrator)

## Critical Changes in Aspire 13

| Before | After |
|--------|-------|
| `.NET Aspire 9.x` | `Aspire 13` |
| `Aspire.Hosting.NodeJs` | `Aspire.Hosting.JavaScript` |
| `AddNpmApp()` | `AddJavaScriptApp()` or `AddViteApp()` |
| Community Toolkit Vite support | Built into core Aspire |
| `dotnet workload install aspire` | `aspire` CLI (standalone) |
| Random ports for JS apps | Configurable with `WithHttpEndpoint` |

## Polyglot Quick Reference

| Language | Method | Use Case |
|----------|--------|----------|
| **JavaScript** | `AddViteApp()` | Vite projects (React, Vue, Svelte) |
| | `AddJavaScriptApp()` | General npm scripts |
| | `AddNodeApp()` | Run specific JS files with Node |
| **Python** | `AddUvicornApp()` | ASGI web apps (FastAPI, Starlette) |
| | `AddPythonModule()` | Run Python modules (`-m` flag) |
| | `AddPythonExecutable()` | Run venv binaries (celery, gunicorn) |
| **.NET** | `AddProject<T>()` | .NET projects |

**Common modifiers:**
```csharp
.WithReference(otherService)     // Service discovery
.WaitFor(dependency)             // Startup ordering
.WithHttpEndpoint(port: 5173, env: "PORT")  // Fixed port
.WithEnvironment("KEY", "value") // Environment variables
```

### Which JavaScript Method?

```dot
digraph js_method {
    rankdir=TB;
    node [shape=diamond];

    "Using Vite?" -> "AddViteApp()" [label="yes"];
    "Using Vite?" -> "Running npm script?" [label="no"];
    "Running npm script?" -> "AddJavaScriptApp()" [label="yes"];
    "Running npm script?" -> "AddNodeApp()" [label="no\n(direct .js file)"];

    "AddViteApp()" [shape=box];
    "AddJavaScriptApp()" [shape=box];
    "AddNodeApp()" [shape=box];
}
```

**Builder syntax:**
```csharp
var builder = DistributedApplication.CreateBuilder(args);  // CORRECT
// NOT: DistributedApplicationBuilder.CreateBuilder
```

## Aspire CLI

Install via platform package managers, not `dotnet workload`:

```bash
# macOS/Linux
curl -sSL https://aspire.dev/install.sh | bash

# Windows
irm https://aspire.dev/install.ps1 | iex
```

### Key Commands

| Command | Purpose |
|---------|---------|
| `aspire new` | Interactive project creation with templates |
| `aspire init` | Initialize existing project for Aspire (can create single-file AppHost) |
| `aspire run` | Start orchestrated apps (auto-terminates previous instances in 13.1+) |
| `aspire update` | Update Aspire packages in solution |
| `aspire update --self` | Update the CLI itself |
| `aspire publish` | Serialize resources to disk (generates manifests, Dockerfiles) |
| `aspire deploy` | Deploy solutions to cloud (Preview) |
| `aspire exec` | Run commands in resource context (e.g., EF migrations) |
| `aspire config` | Manage CLI settings |
| `aspire mcp init` | Configure MCP server for AI assistants |

```bash
# Create new project with template
aspire new
# Templates: aspire-starter, aspire-ts-cs-starter, aspire-py-starter

# Initialize Aspire in existing solution
aspire init
# Discovers existing projects and configures AppHost
# Can create single-file AppHost with #:package directives

# Run with specific launch profile
aspire run --launch-profile Development

# Update all Aspire packages to latest
aspire update

# Run EF migrations in context of a resource
aspire exec api -- dotnet ef database update

# Configure MCP for AI assistants (Claude Code, GitHub Copilot, etc.)
aspire mcp init
```

### aspire init Workflow

When running `aspire init` in an existing solution:

1. Discovers all projects in solution
2. Prompts to select which projects to orchestrate
3. Creates AppHost project if missing (or single-file AppHost)
4. Adds ServiceDefaults project if missing
5. Configures project references automatically

### Deployment Commands

For production deployment, `azd` (Azure Developer CLI) remains the recommended path with first-class Aspire support:

```bash
# Initialize azd for Aspire project
azd init

# Provision infrastructure and deploy
azd up
```

The `aspire publish` and `aspire deploy` commands provide more granular control:
- `aspire publish` generates Docker Compose, Kubernetes manifests, or Bicep templates
- `aspire deploy` (Preview) orchestrates full deployment including infrastructure provisioning

## Dashboard MCP Server

Aspire 13+ includes a **built-in MCP server in the dashboard** that AI assistants can query. This enables AI coding assistants to:

- List all resources with their state and endpoints
- Access console logs in real-time
- Retrieve structured logs and traces
- Execute commands on resources

### Setup with `aspire mcp init`

```bash
# Auto-configure for your AI environment
aspire mcp init
# Detects and configures: VS Code, GitHub Copilot CLI, Claude Code, Open Code
```

Or click the **MCP button** in the Dashboard's top right corner for manual setup.

### Available MCP Tools

| Tool | Purpose |
|------|---------|
| `list_resources` | Lists all resources with state, health, endpoints, commands |
| `list_console_logs` | Console logs for a specific resource |
| `list_structured_logs` | Structured logs, optionally filtered by resource |
| `list_traces` | Distributed traces across services |

> **Note:** Some AI assistants don't support self-signed certificates. Use `aspire config` to configure an HTTP endpoint if needed.

## Environment Variables

### Pattern 1: Fixed Values
```csharp
.WithEnvironment("NODE_ENV", "development")
.WithEnvironment("VITE_FEATURE_FLAG", "true")
```

### Pattern 2: Port Binding (env parameter)
```csharp
// PORT env var set to allocated port value
.WithHttpEndpoint(env: "PORT")

// Or fixed port (for OAuth redirect URLs)
.WithHttpEndpoint(port: 5173, env: "PORT")
```

### Pattern 3: Service Discovery with WithReference
```csharp
var api = builder.AddProject<Projects.Api>("api");
var frontend = builder.AddViteApp("frontend", "../frontend")
    .WithReference(api);

// Aspire automatically sets:
// API_HTTP -> http://localhost:5xxx
// API_HTTPS -> https://localhost:5xxx
```

### Pattern 4: Connection Strings (Polyglot Formats)
```csharp
var postgres = builder.AddPostgres("postgres").AddDatabase("appdb");
var nodeApi = builder.AddNodeApp("api", "../api", "server.js")
    .WithReference(postgres);

// Aspire sets multiple formats for cross-language compatibility:
// APPDB_URI -> postgresql://user:pass@host:port/appdb (URI format)
// APPDB_CONNECTIONSTRING -> Host=...;Database=...;... (ADO.NET format)
// APPDB_HOST, APPDB_PORT, APPDB_USERNAME, APPDB_PASSWORD, APPDB_DATABASE (individual)
```

### VITE_ Prefix (Critical for Frontend)

Vite only exposes env vars prefixed with `VITE_` to client code:

```typescript
// Works
const apiUrl = import.meta.env.VITE_API_URL;

// Won't work (server-side only)
const secret = import.meta.env.API_SECRET;  // undefined
```

**In AppHost:**
```csharp
// For client-side access
.WithEnvironment("VITE_API_URL", api.GetEndpoint("http"))

// For server-side only
.WithEnvironment("DATABASE_URL", db.ConnectionString)
```

## Integrations Quick Reference

### PostgreSQL

```csharp
// AppHost
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()           // Persist data
    .WithPgAdmin();             // Optional admin UI

var appDb = postgres.AddDatabase("appdb");

builder.AddProject<Projects.Api>("api")
    .WithReference(appDb);

// In .NET service
builder.AddNpgsqlDbContext<AppDbContext>("appdb");
```

### Redis

```csharp
// AppHost
var redis = builder.AddRedis("cache")
    .WithDataVolume()
    .WithRedisInsight();  // Optional UI

builder.AddProject<Projects.Api>("api")
    .WithReference(redis);

// In .NET service
builder.AddRedisClient("cache");
// Or for output caching
builder.AddRedisOutputCache("cache");
```

### SQLite

```csharp
// AppHost - SQLite is simpler (file-based)
builder.AddProject<Projects.Api>("api");

// In .NET service Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=app.db"));

// Or with Aspire Community Toolkit SQLite integration
builder.AddSqliteDbContext<AppDbContext>("sqlite");
```

### Integration Packages

| Integration | NuGet Package |
|-------------|---------------|
| PostgreSQL | `Aspire.Npgsql` / `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` |
| Redis | `Aspire.StackExchange.Redis` |
| SQLite (EF) | `CommunityToolkit.Aspire.Hosting.SQLite.EntityFrameworkCore` |
| MongoDB | `Aspire.MongoDB.Driver` |
| RabbitMQ | `Aspire.RabbitMQ.Client` |
| Azure Service Bus | `Aspire.Azure.Messaging.ServiceBus` |
| JavaScript | `Aspire.Hosting.JavaScript` |
| Python | `Aspire.Hosting.Python` |
| MAUI | `Aspire.Hosting.Maui` (preview) |

## Common Issues and Fixes

### Issue: Environment Variables Not Reaching Vite App

**Causes:**
1. Missing `VITE_` prefix
2. Using deprecated `AddNpmApp`
3. Env var set after process starts

**Fix:**
```csharp
var frontend = builder.AddViteApp("frontend", "../frontend")
    .WithEnvironment("VITE_API_URL", api.GetEndpoint("http"));
```

### Issue: Port Changes on Every Restart (Breaking OAuth)

**Fix:** Specify fixed port
```csharp
.WithHttpEndpoint(port: 5173, env: "PORT")
```

### Issue: OTEL Not Showing in Dashboard

**Causes:**
1. Missing instrumentation packages
2. Wrong endpoint URL
3. Missing service name

**Fix for Node.js:**
```bash
# Ensure OTEL_EXPORTER_OTLP_ENDPOINT is set (Aspire does this)
# Check logs for connection errors
```

### Issue: Python Telemetry Missing

**Fix:** Use opentelemetry-instrument wrapper:
```bash
opentelemetry-instrument python -m uvicorn main:app
```

### Issue: Community Toolkit AddViteApp Bugs

**Fix:** Upgrade to Aspire 13 (Vite support moved to core)
```xml
<!-- Remove old package -->
<!-- <PackageReference Include="CommunityToolkit.Aspire.Hosting.NodeJS.Extensions" /> -->

<!-- Use built-in -->
<PackageReference Include="Aspire.Hosting.JavaScript" Version="13.*" />
```

### Issue: JavaScript Apps Build-Only (Can't Deploy Standalone)

**Known limitation:** `AddJavaScriptApp` and `AddViteApp` resources are hardcoded as build-only containers. For deployment scenarios where JS apps need to run as standalone services, you may need custom Dockerfile handling.

### Issue: MAUI App Doesn't Launch

**Expected behavior:** MAUI apps won't auto-launch. Start each .NET MAUI target manually through the dashboard.

## Breaking Changes in Aspire 13

### Package Rename
`Aspire.Hosting.NodeJs` → `Aspire.Hosting.JavaScript`

### Removed APIs
- `AddNpmApp()` - use `AddJavaScriptApp()` or `AddViteApp()`
- Publishing infrastructure (`PublishingContext`, `WithPublishingCallback`)
- Old debugging APIs with `debugAdapterId` parameters

### Constructor Changes
- `AllocatedEndpoint` now requires `NetworkIdentifier` parameter
- `AddNodeApp()` uses relative `scriptPath` instead of absolute paths
- `InteractionInput` properties changed to init-only

### AppHost SDK Change
```xml
<!-- Before (9.x) -->
<Project Sdk="Microsoft.NET.Sdk">
  <Sdk Name="Aspire.AppHost.Sdk" Version="9.5.2" />
  <PackageReference Include="Aspire.Hosting.AppHost" Version="9.5.2" />

<!-- After (13.0) - simplified -->
<Project Sdk="Aspire.AppHost.Sdk/13.0.0">
  <!-- Aspire.Hosting.AppHost included automatically -->
```

### Aspire 13.1 Enhancements
- Channel persistence: `aspire update --self` saves channel selection globally
- `--channel` option on `aspire new` and `aspire init`
- Auto-termination of running instances when running `aspire run`
- stdio-based MCP server via CLI (`aspire mcp init`)

## Anti-Patterns

| Don't | Do |
|-------|-----|
| `dotnet workload install aspire` | Use `aspire` CLI |
| `AddNpmApp()` | `AddViteApp()` or `AddJavaScriptApp()` |
| Hardcode connection strings | Use `WithReference()` |
| Put secrets in AppHost code | Use `builder.AddParameter()` with secrets |
| Skip `AddServiceDefaults()` in .NET | Always add for OTEL + health |
| Use `API_URL` for Vite client | Use `VITE_API_URL` |
| Expect random ports for OAuth | Specify fixed port |
| Console.WriteLine for logging | Use ILogger with structured logging |
| Skip instrumentation in non-.NET | Add OTEL packages to Python/Node |
| Use `aspire do build/publish` | Use `aspire publish` / `aspire deploy` |

## Red Flags - STOP

- Using `Aspire.Hosting.NodeJs` (renamed to `Aspire.Hosting.JavaScript`)
- Using `AddNpmApp` (deprecated, use `AddViteApp` or `AddJavaScriptApp`)
- Using `AddPythonApp` for FastAPI/web apps (use `AddUvicornApp` instead)
- Using Community Toolkit for Vite (moved to core in Aspire 13)
- Missing `VITE_` prefix for client-side env vars
- Not calling `AddServiceDefaults()` in .NET services
- Using `dotnet workload` instead of `aspire` CLI
- Expecting env vars without `.WithReference()` or `.WithEnvironment()`
- Not specifying port when OAuth/auth requires fixed URLs
- Using `DistributedApplicationBuilder.CreateBuilder` (use `DistributedApplication.CreateBuilder`)
- Logging without structured properties (breaks dashboard search)
- Using `aspire do build` or `aspire do publish` (these aren't real commands)
