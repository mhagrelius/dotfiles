---
name: working-with-ms-agent-framework
description: Use when building AI agents with Microsoft Agent Framework (Semantic Kernel + AutoGen unified); when implementing memory or context providers; when threads won't deserialize; when workflow checkpointing fails; when migrating from Semantic Kernel or AutoGen; when seeing ChatAgent or AgentThread errors; when exposing agents as MCP tools; when needing human-in-the-loop approvals; when hosting agents in Azure Functions
---

# Working with Microsoft Agent Framework

Microsoft Agent Framework unifies Semantic Kernel and AutoGen into one SDK. Both legacy frameworks are in maintenance mode. Currently in public preview.

**Core principle:** Agents are stateless. All state lives in threads. Context providers enforce **policy** about what enters the prompt, how, and when it decays.

> **Additional reference files in this skill:**
> - `context-providers.md` - Policy-based memory, capsule pattern, Mem0 integration
> - `orchestration-patterns.md` - The 5 orchestration patterns with when-to-use guidance
> - `design-patterns.md` - Production patterns, testing, migration

## When to Use

- Building AI agents with Microsoft's unified framework
- Implementing custom memory with Context Providers
- Creating multi-agent workflows with checkpointing
- Migrating from Semantic Kernel or AutoGen

## When NOT to Use

- Simple single-turn LLM calls (use chat client directly)
- Projects staying on legacy SK/AutoGen
- Non-Microsoft frameworks (LangChain, CrewAI)

## Technology Stack Hierarchy

**Official guidance (Jeremy Licknes, PM):** Start with ME AI, escalate only when needed.

| Layer | Use For | When to Escalate |
|-------|---------|------------------|
| **ME AI** (Microsoft.Extensions.AI) | Chat clients, structured outputs, embeddings, middleware | Need agents, workflows, memory |
| **Agent Framework** | Agents, threads, orchestration, context providers | Need specific SK adapters |
| **Semantic Kernel** | Specific adapters, utilities not in ME AI | Never start here |

```
ME AI (foundation) → Agent Framework (agents/workflows) → SK (specific utilities only)
```

**Key insight:** ME AI provides universal APIs that work across OpenAI, Ollama, Foundry Local, etc. Agent Framework builds on ME AI for agentic patterns. SK primitives migrated to ME AI; only use SK for specific adapters not yet in ME AI.

**ME AI features you get automatically:**
- Structured outputs (typed responses via extension methods)
- Middleware (OpenTelemetry, chat reduction)
- Universal chat client abstraction
- Embeddings generation

## Architecture Quick Reference

| Concept | C# Type | Purpose |
|---------|---------|---------|
| Agent | `AIAgent` | Stateless LLM wrapper |
| Thread | `AgentThread` | Stateful conversation container |
| Context Provider | `AIContextBehavior` | Policy-based memory/context injection |
| Orchestration | `SequentialOrchestration`, etc. | Multi-agent coordination |

## Installation

```bash
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet add package Azure.AI.OpenAI --version 2.1.0
```

## Agent Creation

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

// Azure OpenAI
AIAgent agent = new AzureOpenAIClient(
    new Uri("https://<resource>.openai.azure.com"),
    new AzureCliCredential())
        .GetChatClient("gpt-4o-mini")
        .CreateAIAgent(
            instructions: "You are a helpful assistant.",
            name: "Assistant");

// Direct OpenAI
var agent = new OpenAIClient("api-key")
    .GetChatClient("gpt-4o-mini")
    .AsIChatClient()
    .CreateAIAgent(instructions: "...", name: "Assistant");

// With tools
[Description("Gets weather for a location")]
static string GetWeather(string location) => $"Sunny in {location}";

AIAgent agent = chatClient.CreateAIAgent(
    instructions: "You help with weather queries.",
    tools: [AIFunctionFactory.Create(GetWeather)]
);
```

## Execution

```csharp
// Simple
Console.WriteLine(await agent.RunAsync("Hello!"));

// With thread for multi-turn
AgentThread thread = agent.GetNewThread();
await agent.RunAsync("My name is Alice.", thread);
await agent.RunAsync("What's my name?", thread); // Remembers "Alice"

// Streaming
await foreach (var update in agent.RunStreamingAsync("Tell me a story.", thread))
{
    Console.Write(update.Text);
}
```

## Streaming with Resilience

For production streaming, add cancellation support and resilience:

```csharp
// Basic streaming with cancellation
await foreach (var update in agent.RunStreamingAsync("Tell me a story.", thread)
    .WithCancellation(cancellationToken))
{
    Console.Write(update.Text);
}

// With Polly resilience pipeline
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
    .AddTimeout(TimeSpan.FromMinutes(2))
    .Build();

await pipeline.ExecuteAsync(async token =>
{
    await foreach (var chunk in agent.RunStreamingAsync(userMessage, thread)
        .WithCancellation(token))
    {
        Console.Write(chunk.Text);
    }
}, cancellationToken);
```

**Error differentiation:**
- `OperationCanceledException`: User cancelled
- `TimeoutRejectedException`: Polly timeout
- `HttpRequestException`: Network issues

## Development UI (DevUI)

Lightweight web interface for testing agents and workflows. **Development only—not for production.**

### Python Setup

**Install:**
```bash
pip install agent-framework-devui --pre
```

**Option 1: Programmatic Registration**
```python
from agent_framework import ChatAgent
from agent_framework.openai import OpenAIChatClient
from agent_framework.devui import serve

agent = ChatAgent(
    name="WeatherAgent",
    chat_client=OpenAIChatClient(),
    tools=[get_weather]
)

# Launch DevUI with tracing
serve(entities=[agent], auto_open=True, tracing_enabled=True)
# Opens browser to http://localhost:8080
```

**Option 2: Directory Discovery (CLI)**
```bash
devui ./entities --port 8080 --tracing
```

Directory structure for discovery:
```
entities/
    weather_agent/
        __init__.py      # Must export: agent = ChatAgent(...)
        .env             # Optional: API keys
    my_workflow/
        __init__.py      # Must export: workflow = WorkflowBuilder()...
```

### C# Setup

C# embeds DevUI as SDK component:
```csharp
var app = builder.Build();

app.MapOpenAIResponses();
app.MapConversation();

if (app.Environment.IsDevelopment())
{
    app.MapAgentUI(); // Accessible at /ui
}
```

### Features

| Feature | Description |
|---------|-------------|
| Web interface | Interactive testing of agents/workflows |
| OpenAI-compatible API | Use OpenAI SDK against local agents |
| Tracing | OpenTelemetry spans in debug panel |
| File uploads | Multimodal inputs (images, documents) |
| Auto-generated inputs | Workflow inputs based on first executor type |

### Tracing in DevUI

Enable with `--tracing` flag or `tracing_enabled=True`. View in debug panel:
```
Agent Execution
├── LLM Call (prompt → response)
├── Tool Call
│   ├── Tool Execution
│   └── Tool Result
└── LLM Call (prompt → response)
```

Export to external tools (Jaeger, Azure Monitor):
```bash
export OTLP_ENDPOINT="http://localhost:4317"
devui ./entities --tracing
```

### OpenAI SDK Integration

Interact with DevUI agents via OpenAI Python SDK:
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8080/v1", api_key="not-needed")
response = client.responses.create(
    metadata={"entity_id": "weather_agent"},
    input="What's the weather in Seattle?"
)
```

### CLI Options

```
devui [directory] [options]
  --port, -p      Port (default: 8080)
  --tracing       Enable OpenTelemetry tracing
  --reload        Auto-reload on file changes
  --headless      API only, no UI
  --mode          developer|user (default: developer)
```

## Structured Output

Get typed responses using JSON schema:

```csharp
public class PersonInfo
{
    [JsonPropertyName("name")] public string? Name { get; set; }
    [JsonPropertyName("age")] public int? Age { get; set; }
    [JsonPropertyName("occupation")] public string? Occupation { get; set; }
}

// Create schema from type
JsonElement schema = AIJsonUtilities.CreateJsonSchema(typeof(PersonInfo));

// Configure response format
ChatOptions chatOptions = new()
{
    ResponseFormat = ChatResponseFormatJson.ForJsonSchema(
        schema: schema,
        schemaName: "PersonInfo",
        schemaDescription: "Information about a person")
};

AIAgent agent = chatClient.CreateAIAgent(new ChatClientAgentOptions()
{
    Name = "Assistant",
    Instructions = "You extract person information.",
    ChatOptions = chatOptions
});

var response = await agent.RunAsync("John Smith is a 35-year-old software engineer.");
var personInfo = response.Deserialize<PersonInfo>(JsonSerializerOptions.Web);
```

## Agent as Function Tool

Use `.AsAIFunction()` to compose agents:

```csharp
// Create a specialist agent
AIAgent weatherAgent = chatClient.CreateAIAgent(
    instructions: "You answer questions about the weather.",
    name: "WeatherAgent",
    description: "An agent that answers questions about the weather.",
    tools: [AIFunctionFactory.Create(GetWeather)]);

// Use it as a tool in a main agent
AIAgent mainAgent = chatClient.CreateAIAgent(
    instructions: "You are a helpful assistant who responds in French.",
    tools: [weatherAgent.AsAIFunction()]);

// Main agent can now delegate to weather agent
Console.WriteLine(await mainAgent.RunAsync("What's the weather in Paris?"));
```

## MCP Integration

Expose agents as Model Context Protocol (MCP) tools:

```csharp
// Install packages
// dotnet add package ModelContextProtocol --prerelease

using ModelContextProtocol.Server;
using Microsoft.Extensions.Hosting;

// Create agent
AIAgent agent = chatClient.CreateAIAgent(
    instructions: "You are good at telling jokes.",
    name: "Joker");

// Convert to MCP tool (name and description from agent)
McpServerTool tool = McpServerTool.Create(agent.AsAIFunction());

// Setup MCP server over stdio
HostApplicationBuilder builder = Host.CreateEmptyApplicationBuilder(settings: null);
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithTools([tool]);

await builder.Build().RunAsync();
```

## Human-in-the-Loop Approvals

Require user approval before executing sensitive functions:

```csharp
[Description("Process a refund for an order")]
static string ProcessRefund(string orderId) => $"Refund processed for {orderId}";

// Wrap function to require approval
AIFunction refundFunction = AIFunctionFactory.Create(ProcessRefund);
AIFunction approvalRequired = new ApprovalRequiredAIFunction(refundFunction);

AIAgent agent = chatClient.CreateAIAgent(
    instructions: "You help with customer orders.",
    tools: [approvalRequired]);

// Run agent
AgentThread thread = agent.GetNewThread();
AgentRunResponse response = await agent.RunAsync("Process refund for order #12345", thread);

// Check for approval requests
var approvalRequests = response.Messages
    .SelectMany(m => m.Contents)
    .OfType<FunctionApprovalRequestContent>()
    .ToList();

if (approvalRequests.Any())
{
    var request = approvalRequests.First();
    Console.WriteLine($"Approval needed for: {request.FunctionCall.Name}");

    // Create approval/rejection response
    var approvalMessage = new ChatMessage(ChatRole.User,
        [request.CreateResponse(approved: true)]); // or false to reject

    // Continue with approval
    Console.WriteLine(await agent.RunAsync(approvalMessage, thread));
}
```

## Durable Agents (Azure Functions)

Host agents in Azure Functions with automatic state management:

```csharp
// Program.cs
using Microsoft.Agents.AI.Hosting.AzureFunctions;

AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
    .GetChatClient("gpt-4o-mini")
    .CreateAIAgent(
        instructions: "You are a helpful assistant.",
        name: "MyDurableAgent");

using IHost app = FunctionsApplication
    .CreateBuilder(args)
    .ConfigureFunctionsWebApplication()
    .ConfigureDurableAgents(options => options.AddAIAgent(agent))
    .Build();

app.Run();
```

**Features:**
- Automatic HTTP endpoints for agent interaction
- Thread state persisted in Durable Task Scheduler
- Survives process restarts
- Scales to zero when idle

**Endpoints created automatically:**
```
POST /api/agents/MyDurableAgent/run  # Start/continue conversation
```

**Continuing conversations:**
```bash
# First request - note x-ms-thread-id header in response
curl -X POST http://localhost:7071/api/agents/MyDurableAgent/run \
  -H "Content-Type: text/plain" \
  -d "Hello!"

# Continue same conversation using thread_id
curl -X POST "http://localhost:7071/api/agents/MyDurableAgent/run?thread_id=<thread-id>" \
  -H "Content-Type: text/plain" \
  -d "What did I just say?"
```

**Multi-Agent Orchestration with Durable Functions:**

```csharp
// Register multiple agents
builder.ConfigureDurableAgents(options =>
{
    options.AddAIAgent(mainAgent);
    options.AddAIAgent(frenchTranslator);
    options.AddAIAgent(spanishTranslator);
});

// Orchestration function
[Function("translate_workflow")]
public static async Task<Dictionary<string, string>> TranslateWorkflow(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var input = context.GetInput<string>();

    // Get main response
    DurableAIAgent mainAgent = context.GetAgent("MainAgent");
    var mainResponse = await mainAgent.RunAsync<TextResponse>(input);

    // Fan out to translators concurrently
    DurableAIAgent french = context.GetAgent("FrenchTranslator");
    DurableAIAgent spanish = context.GetAgent("SpanishTranslator");

    var frenchTask = french.RunAsync<TextResponse>(mainResponse.Result.Text);
    var spanishTask = spanish.RunAsync<TextResponse>(mainResponse.Result.Text);

    await Task.WhenAll(frenchTask, spanishTask);

    return new Dictionary<string, string>
    {
        ["original"] = mainResponse.Result.Text,
        ["french"] = (await frenchTask).Result.Text,
        ["spanish"] = (await spanishTask).Result.Text
    };
}
```

## Thread Serialization (Critical Pattern)

```csharp
// Serialize for persistence
JsonElement serialized = await thread.SerializeAsync();
await File.WriteAllTextAsync("thread.json", serialized.GetRawText());

// Later: restore and resume
string json = await File.ReadAllTextAsync("thread.json");
JsonElement element = JsonSerializer.Deserialize<JsonElement>(json);
AgentThread restored = agent.DeserializeThread(element, JsonSerializerOptions.Web);
await agent.RunAsync("Continue...", restored);
```

**Key behaviors:**
- Service-managed threads: Only thread ID serialized
- In-memory threads: All messages serialized
- **WARNING**: Deserializing with different agent config may error

## Context Providers (Policy-Based Memory)

Context providers are not "memory injection" — they're **policy enforcement**:

| Policy | What It Decides |
|--------|-----------------|
| Selection | What becomes memory |
| Gating | When it's retrieved |
| Decay | When it expires |
| Noise avoidance | When NOT to use |

```csharp
ChatHistoryAgentThread thread = new();

// Long-term user memory
thread.AIContextProviders.Add(new Mem0Provider(httpClient, new() { UserId = "user123" }));

// Short-term conversation context
thread.AIContextProviders.Add(new WhiteboardProvider(chatClient));

// RAG integration
thread.AIContextProviders.Add(new TextSearchProvider(textSearch, new()
{
    SearchTime = TextSearchProviderOptions.RagBehavior.OnDemandFunctionCalling
}));
```

See `context-providers.md` for custom implementation patterns.

## Orchestration Patterns

| Pattern | Use When |
|---------|----------|
| **Sequential** | Clear dependencies (draft → review → polish) |
| **Concurrent** | Independent perspectives, ensemble reasoning |
| **Handoff** | Unknown optimal agent upfront, dynamic expertise |
| **GroupChat** | Collaborative ideation, human-in-the-loop |
| **Magentic** | Complex open-ended problems |

```csharp
// Sequential
SequentialOrchestration orchestration = new(analystAgent, writerAgent);

// Handoff - CRITICAL: Always set termination conditions!
var workflow = AgentWorkflowBuilder.StartHandoffWith(triageAgent)
    .WithHandoffs(triageAgent, [mathTutor, historyTutor])
    .WithHandoff(mathTutor, triageAgent)      // Allows routing back
    .WithHandoff(historyTutor, triageAgent)
    .WithMaxHandoffs(10)                      // REQUIRED: Prevent infinite loops
    .Build();

// Execute
InProcessRuntime runtime = new();
await runtime.StartAsync();
var result = await orchestration.InvokeAsync(task, runtime);
```

**CRITICAL for Handoffs**: Missing `.WithMaxHandoffs()` causes infinite loops. Always set termination conditions.

See `orchestration-patterns.md` for detailed patterns and when-to-use guidance.

## Workflow Checkpointing & Durability

For workflows that must survive process restarts:

```csharp
// Basic checkpointing with CheckpointManager
var checkpointManager = CheckpointManager.Default;

await using Checkpointed<StreamingRun> checkpointedRun =
    await InProcessExecution.StreamAsync(workflow, input, checkpointManager);

// Resume from checkpoint
await InProcessExecution.ResumeStreamAsync(savedCheckpoint, checkpointManager);
```

### Thread-Based Persistence Pattern

For long-running workflows, checkpoint thread state after each step:

```csharp
// Save thread state after each workflow step
var serialized = await thread.SerializeAsync();
await checkpointStore.SaveAsync(workflowId, currentStep, serialized.GetRawText());

// Resume after restart
var json = await checkpointStore.GetAsync(workflowId);
var element = JsonSerializer.Deserialize<JsonElement>(json);
var restored = agent.DeserializeThread(element, JsonSerializerOptions.Web);
await agent.RunAsync(nextStep, restored);
```

### Recovery on Startup

```csharp
public class WorkflowRecoveryService : BackgroundService
{
    private readonly ICheckpointStore _store;
    private readonly AIAgent _agent;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var pending = await _store.GetPendingWorkflowsAsync();
        foreach (var workflow in pending)
        {
            var thread = _agent.DeserializeThread(workflow.State, JsonSerializerOptions.Web);
            await _agent.RunAsync(workflow.NextStep, thread, cancellationToken: ct);
        }
    }
}
```

**Key principle**: Checkpoint **after** each step completes, not before. This ensures you can resume from the last successful step.

## Migration Quick Reference

### From Semantic Kernel

| SK | Agent Framework |
|----|-----------------|
| `Kernel` | `AIAgent` |
| `ChatHistory` | `AgentThread` |
| `[KernelFunction]` | `[Description]` on methods |
| `IPromptFilter` | `AIContextBehavior` |
| `KernelFunctionFactory.CreateFromMethod` | `AIFunctionFactory.Create` |

### From AutoGen

| AutoGen | Agent Framework |
|---------|-----------------|
| `AssistantAgent` | `AIAgent` via `CreateAIAgent()` |
| `FunctionTool` | `AIFunctionFactory.Create()` |
| GroupChat/Teams | `WorkflowBuilder` patterns |
| `TopicSubscription` | `AgentWorkflowBuilder.WithHandoffs()` |
| `BaseAgent, IHandle<>` | `AIAgent` with tools |

**Topic-Based to Handoff Migration:**

```csharp
// ❌ OLD AutoGen pattern (deprecated)
[TopicSubscription("queries")]
public class MyAgent : BaseAgent, IHandle<Query>
{
    public async Task Handle(Query msg, CancellationToken ct)
    {
        // Process and publish to another topic
        await PublishMessageAsync(new Response(...), "responses");
    }
}

// ✅ NEW Agent Framework pattern
var triageAgent = chatClient.CreateAIAgent(
    instructions: "Route queries to appropriate specialist.",
    name: "Triage");

var mathAgent = chatClient.CreateAIAgent(
    instructions: "Handle math queries.",
    name: "Math");

var workflow = AgentWorkflowBuilder.StartHandoffWith(triageAgent)
    .WithHandoffs(triageAgent, [mathAgent, otherAgent])
    .WithMaxHandoffs(10)  // REQUIRED
    .Build();

await workflow.InvokeStreamingAsync(input, runtime);
```

## Anti-Patterns

| Don't | Do |
|-------|-----|
| Store state in agent instances | Use `AgentThread` for all state |
| Serialize only messages | Serialize entire thread |
| Share agent instances in workflows | Use factory pattern |
| Mix thread types across services | Threads are service-specific |
| Use Magentic when Sequential suffices | Use simplest pattern that works |
| Skip `UseImmutableKernel` with `ContextualFunctionProvider` | Always set `UseImmutableKernel = true` |
| Start with Semantic Kernel for new projects | Start with ME AI, escalate to Agent Framework |
| Skip approval for sensitive operations | Use `ApprovalRequiredAIFunction` |
| Build custom HTTP handlers for agents | Use Durable Agents with auto-generated endpoints |
| Parse unstructured LLM output | Use structured output with JSON schema |

## Red Flags - STOP

- Using `Kernel` instead of `AIAgent` (old SK)
- Using `AssistantAgent` instead of `AIAgent` (old AutoGen)
- Thread deserialization fails (missing serialization constructor in context provider)
- Memory lost between sessions (serializing messages instead of thread)
- Infinite handoff loops (need termination conditions)

## Known Limitations

- **Distributed runtime**: In-process only; distributed execution via Durable Functions
- **C# Magentic**: Most examples are Python; use GroupChat or Handoff patterns in C#
- **C# message stores**: Redis/database need custom `ChatMessageStore` implementation
- **Token counting**: Budget calculation in providers undocumented
- **Preview status**: Agent Framework is in public preview (as of Jan 2026)

## Resources

- GitHub: `github.com/microsoft/agent-framework`
- Docs: `learn.microsoft.com/en-us/agent-framework/`
- Migration: `learn.microsoft.com/en-us/agent-framework/migration-guide/`
