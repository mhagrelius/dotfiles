# Aspire App Types

Language-specific integration patterns for Aspire 13+.

## .NET Integration

### Service Defaults Pattern

Every .NET service should reference `ServiceDefaults`:

```csharp
// In your web API Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();  // Adds OTEL, health checks, resilience

var app = builder.Build();
app.MapDefaultEndpoints();  // Maps health check endpoints
app.Run();
```

### OpenTelemetry for .NET (Built-in via ServiceDefaults)

ServiceDefaults automatically configures OTEL. For custom instrumentation:

```csharp
// Program.cs - Custom tracing
using System.Diagnostics;

var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();

// Add custom activity source
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("MyApp.CustomTracing"));

var app = builder.Build();

// Create traces
var activitySource = new ActivitySource("MyApp.CustomTracing");
using var activity = activitySource.StartActivity("ProcessOrder");
activity?.SetTag("order.id", orderId);
```

### Structured Logging (.NET)

```csharp
// Automatic correlation with OTEL
public class OrderService(ILogger<OrderService> logger)
{
    public void ProcessOrder(Order order)
    {
        // Structured logging - properties become searchable in dashboard
        logger.LogInformation("Processing order {OrderId} for {CustomerId}",
            order.Id, order.CustomerId);

        using var scope = logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderTotal"] = order.Total,
            ["ItemCount"] = order.Items.Count
        });

        logger.LogInformation("Order details logged with scope");
    }
}
```

## JavaScript Integration

### Vite Applications

```csharp
var api = builder.AddProject<Projects.Backend>("api");

var frontend = builder.AddViteApp("frontend", "../frontend")
    .WithPnpm()  // or WithNpm(), WithYarn()
    .WithReference(api)
    .WithHttpEndpoint(port: 5173, env: "PORT")
    .WithEnvironment("VITE_API_URL", api.GetEndpoint("http"))
    .WithExternalHttpEndpoints();
```

### Generic JavaScript Apps

```csharp
// For non-Vite JavaScript applications
var nodeApi = builder.AddJavaScriptApp("api", "../api", "server.js")
    .WithPnpm()
    .WithReference(postgres)
    .WithHttpEndpoint(env: "PORT");
```

### Dockerfile Auto-Generation

Aspire generates optimized multi-stage Dockerfiles for JavaScript apps:
- Detects Node version from `.nvmrc`, `.node-version`, or `package.json engines`
- Defaults to `node:22-slim` base image
- Configures build steps based on detected package manager

### OpenTelemetry for Node.js

**Required packages:**
```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-grpc @opentelemetry/exporter-metrics-otlp-grpc \
  @opentelemetry/exporter-logs-otlp-grpc @grpc/grpc-js
```

**Create `instrumentation.ts`** (must load before app):
```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

// Aspire sets OTEL_EXPORTER_OTLP_ENDPOINT automatically
if (process.env.OTEL_EXPORTER_OTLP_ENDPOINT) {
    new NodeSDK({
        traceExporter: new OTLPTraceExporter(),
        instrumentations: [getNodeAutoInstrumentations()],
    }).start();
}
```

**Key points:**
- Load instrumentation BEFORE your app code (use `--import` or `--require`)
- Aspire auto-sets `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_SERVICE_NAME`
- For metrics/logs, add `OTLPMetricExporter` and `OTLPLogExporter` (see [OTEL JS docs](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/))

### Structured Logging (Node.js)

Use JSON logging for dashboard searchability:
```typescript
import pino from 'pino';
const logger = pino({ level: 'info' });

// Properties become searchable in Aspire dashboard
logger.info({ orderId: '123', customerId: '456' }, 'Processing order');
```

## Python Integration

Aspire 13 provides four methods for Python apps:

| Method | Use Case |
|--------|----------|
| `AddUvicornApp()` | ASGI web apps (FastAPI, Starlette, Quart) |
| `AddPythonApp()` | Run Python scripts directly (experimental) |
| `AddPythonModule()` | Run Python modules (`-m` flag) |
| `AddPythonExecutable()` | Execute venv binaries |

> **Note:** `AddPythonApp` is experimental and generates the `ASPIREHOSTINGPYTHON001` diagnostic warning.

### AddUvicornApp (Web Applications)

```csharp
var pythonApi = builder.AddUvicornApp("python-api", "../python-app", "main:app")
    .WithUv()  // or WithPip()
    .WithReference(postgres)
    .WithHttpEndpoint(env: "PORT");
```

### AddPythonApp (Scripts) - Experimental

For non-HTTP Python scripts (data processing, batch jobs):

```csharp
// Run a Python script
#pragma warning disable ASPIREHOSTINGPYTHON001
var dataProcessor = builder.AddPythonApp("processor", "../scripts", "process_data.py")
    .WithUv()
    .WithReference(postgres)
    .WithArgs("--batch-size", "100");
#pragma warning restore ASPIREHOSTINGPYTHON001
```

### AddPythonModule (Module Execution)

Run Python modules using `-m` flag:

```csharp
// Run: python -m mypackage.cli
var worker = builder.AddPythonModule("worker", "../worker", "mypackage.cli")
    .WithUv()
    .WithReference(redis);
```

### AddPythonExecutable (venv Binaries)

Execute binaries installed in virtual environment:

```csharp
// Run celery worker from venv
var celery = builder.AddPythonExecutable("celery", "../backend", "celery")
    .WithUv()
    .WithArgs("worker", "--app=tasks", "--loglevel=info")
    .WithReference(redis);

// Run gunicorn
var api = builder.AddPythonExecutable("api", "../api", "gunicorn")
    .WithUv()
    .WithArgs("app:app", "--bind", "0.0.0.0:8000");
```

### Package Manager Support

```csharp
.WithUv()   // Modern, fast (10-100x faster installs) - runs uv sync
.WithPip()  // Traditional pip
```

Aspire auto-detects and configures package management when you add a Python application.

### OpenTelemetry for Python

**Zero-code instrumentation (recommended):**
```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp

# Run with auto-instrumentation
opentelemetry-instrument python -m uvicorn main:app --port $PORT
```

Aspire auto-sets these environment variables:
- `OTEL_EXPORTER_OTLP_ENDPOINT` - Dashboard collector URL
- `OTEL_SERVICE_NAME` - Resource name from AppHost
- `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`

**Custom spans** (when auto-instrumentation isn't enough):
```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

@app.post("/orders")
async def create_order(order: Order):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order.id)
        # ... processing
```

See [OTEL Python docs](https://opentelemetry.io/docs/languages/python/getting-started/) for manual SDK setup.

### Structured Logging (Python)

Use `structlog` or JSON formatting for dashboard searchability:
```python
import structlog
logger = structlog.get_logger()

# Properties become searchable in Aspire dashboard
logger.info("processing_order", order_id="123", customer_id="456")
```

Or with standard logging + `python-json-logger`:
```python
from pythonjsonlogger import jsonlogger
handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter())
logging.root.addHandler(handler)
```

## MAUI Mobile Development

Aspire 13 supports .NET MAUI mobile apps with device emulators via `Aspire.Hosting.Maui` (preview).

> **Note:** MAUI apps won't auto-launch. Start each .NET MAUI target manually through the dashboard.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var api = builder.AddProject<Projects.Api>("api");

// MAUI mobile app with multiple device targets
var mauiApp = builder.AddMauiProject("myapp", "../MyApp/MyApp.csproj");

// Desktop targets (connect directly to localhost)
mauiApp.AddWindowsDevice().WithReference(api);
mauiApp.AddMacCatalystDevice().WithReference(api);

// Mobile emulators (need Dev Tunnel for localhost access)
var devTunnel = builder.AddDevTunnel("api-tunnel")
    .WithAnonymousAccess()
    .WithReference(api.GetEndpoint("https"));

mauiApp.AddiOSSimulator()
    .WithOtlpDevTunnel()
    .WithReference(api, devTunnel);

mauiApp.AddAndroidEmulator()
    .WithOtlpDevTunnel()
    .WithReference(api, devTunnel);

builder.Build().Run();
```

## Dev Tunnels

Dev Tunnels expose local services to external devices (mobile emulators, webhooks):

```csharp
var api = builder.AddProject<Projects.Api>("api");

// Public tunnel (anonymous access)
var publicTunnel = builder.AddDevTunnel("public-api")
    .WithAnonymousAccess()
    .WithReference(api.GetEndpoint("https"));

// Authenticated tunnel (requires login)
var privateTunnel = builder.AddDevTunnel("private-api")
    .WithReference(api.GetEndpoint("https"));

// Use tunnel URL in other services
var webhook = builder.AddProject<Projects.Webhook>("webhook")
    .WithEnvironment("CALLBACK_URL", publicTunnel.GetEndpoint("https"));
```

## C# Single-File Apps (Experimental)

Run single-file C# scripts without full projects (requires .NET 10 SDK):

```csharp
#pragma warning disable ASPIREHOSTINGCSHARP001
var script = builder.AddCSharpApp("myscript", "../scripts/process.cs");
#pragma warning restore ASPIREHOSTINGCSHARP001
```

## Full Stack Examples

### FastAPI + React/Vite Stack

Recommended pattern for a FastAPI backend with React frontend using Vite.

**AppHost Configuration:**

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()
    .AddDatabase("appdb");

var redis = builder.AddRedis("cache");

// FastAPI Backend
var api = builder.AddUvicornApp("api", "../backend", "main:app")
    .WithUv()
    .WithReference(postgres)
    .WithReference(redis)
    .WaitFor(postgres)
    .WithHttpEndpoint(port: 8000, env: "PORT")
    .WithEnvironment("DATABASE_URL", postgres.Resource.ConnectionStringExpression);

// React Frontend (Vite)
var frontend = builder.AddViteApp("frontend", "../frontend")
    .WithPnpm()
    .WithReference(api)
    .WaitFor(api)
    .WithHttpEndpoint(port: 5173, env: "PORT")
    .WithEnvironment("VITE_API_URL", api.GetEndpoint("http"))
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

**FastAPI Backend (main.py):**

```python
import os
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
import structlog

app = FastAPI()

# CORS for Vite dev server
app.add_middleware(
    CORSMiddleware,
    allow_origins=[os.getenv("FRONTEND_URL", "http://localhost:5173")],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Auto-instrument FastAPI for OTEL
FastAPIInstrumentor.instrument_app(app)

# Structured logging
logger = structlog.get_logger()

@app.get("/api/health")
async def health():
    return {"status": "healthy"}

@app.get("/api/items")
async def get_items():
    logger.info("fetching_items", count=10)
    return {"items": []}
```

**Vite Proxy Configuration (vite.config.ts):**

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: parseInt(process.env.PORT || '5173'),
    proxy: {
      '/api': {
        target: process.env.API_HTTP || 'http://localhost:8000',
        changeOrigin: true,
        secure: false,
      },
    },
  },
});
```

**React API Client:**

```typescript
// src/api/client.ts
const API_URL = import.meta.env.VITE_API_URL || '';

export async function fetchItems() {
  // In dev: uses proxy (/api/items -> backend)
  // In prod: uses VITE_API_URL directly
  const response = await fetch(`${API_URL}/api/items`);
  return response.json();
}
```

### Orchestrating Custom MCP Servers

MCP (Model Context Protocol) servers can be orchestrated alongside your other services.

> **Note:** This is different from the built-in Aspire Dashboard MCP server. This section covers orchestrating your own MCP servers as application services.

**Python MCP Server:**

```csharp
// AppHost
var mcpServer = builder.AddUvicornApp("mcp-server", "../mcp-python", "server:app")
    .WithUv()
    .WithHttpEndpoint(port: 3001, env: "PORT")
    .WithEnvironment("MCP_SERVER_NAME", "python-tools");
```

```python
# mcp-python/server.py
import os
from mcp.server import Server
from mcp.server.stdio import stdio_server
import structlog

logger = structlog.get_logger()

app = Server("python-tools")

@app.list_tools()
async def list_tools():
    logger.info("listing_tools")
    return [
        {"name": "analyze", "description": "Analyze data"},
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    logger.info("calling_tool", tool=name, args=arguments)
    if name == "analyze":
        return {"result": "analysis complete"}

if __name__ == "__main__":
    import asyncio
    asyncio.run(stdio_server(app).run())
```

**Node.js MCP Server:**

```csharp
// AppHost
var nodeMcp = builder.AddNodeApp("mcp-node", "../mcp-node", "server.js")
    .WithPnpm()
    .WithHttpEndpoint(port: 3002, env: "PORT")
    .WithReference(redis);  // For caching tool results
```

```typescript
// mcp-node/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import pino from 'pino';

const logger = pino({ level: 'info' });

const server = new Server(
  { name: "node-tools", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler("tools/list", async () => {
  logger.info({ event: "list_tools" });
  return {
    tools: [
      { name: "search", description: "Search the web" },
    ],
  };
});

server.setRequestHandler("tools/call", async (request) => {
  logger.info({ event: "call_tool", tool: request.params.name });
  // Tool implementation
  return { content: [{ type: "text", text: "result" }] };
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Full Polyglot Example

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()
    .AddDatabase("appdb");

var redis = builder.AddRedis("cache")
    .WithDataVolume();

// .NET Backend API
var api = builder.AddProject<Projects.Backend>("api")
    .WithReference(postgres)
    .WithReference(redis)
    .WaitFor(postgres);

// Python ML Service
var mlService = builder.AddUvicornApp("ml-service", "../ml-service", "main:app")
    .WithUv()
    .WithReference(redis)
    .WithHttpEndpoint(env: "PORT");

// Vite Frontend
var frontend = builder.AddViteApp("frontend", "../frontend")
    .WithPnpm()
    .WithReference(api)
    .WaitFor(api)
    .WithHttpEndpoint(port: 5173, env: "PORT")
    .WithEnvironment("VITE_API_URL", api.GetEndpoint("http"))
    .WithEnvironment("VITE_ML_URL", mlService.GetEndpoint("http"))
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

### Full Stack with Custom MCP Servers

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var postgres = builder.AddPostgres("postgres").AddDatabase("appdb");
var redis = builder.AddRedis("cache");

// FastAPI Backend
var api = builder.AddUvicornApp("api", "../backend", "main:app")
    .WithUv()
    .WithReference(postgres)
    .WithReference(redis)
    .WithHttpEndpoint(port: 8000, env: "PORT");

// Python MCP Server
var mcpPython = builder.AddUvicornApp("mcp-python", "../mcp-python", "server:app")
    .WithUv()
    .WithReference(postgres)
    .WithHttpEndpoint(port: 3001, env: "PORT");

// Node MCP Server
var mcpNode = builder.AddNodeApp("mcp-node", "../mcp-node", "server.js")
    .WithPnpm()
    .WithReference(redis)
    .WithHttpEndpoint(port: 3002, env: "PORT");

// React Frontend
var frontend = builder.AddViteApp("frontend", "../frontend")
    .WithPnpm()
    .WithReference(api)
    .WithReference(mcpPython)
    .WithReference(mcpNode)
    .WaitFor(api)
    .WithHttpEndpoint(port: 5173, env: "PORT")
    .WithEnvironment("VITE_API_URL", api.GetEndpoint("http"))
    .WithEnvironment("VITE_MCP_PYTHON_URL", mcpPython.GetEndpoint("http"))
    .WithEnvironment("VITE_MCP_NODE_URL", mcpNode.GetEndpoint("http"))
    .WithExternalHttpEndpoints();

builder.Build().Run();
```
