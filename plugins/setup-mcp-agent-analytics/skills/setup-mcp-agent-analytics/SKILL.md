---
name: setup-mcp-agent-analytics
description: >-
  Instrument an MCP server with Pendo MCP analytics in Python, TypeScript,
  Go, Java, or .NET. Detects which language/SDK the server is built with and
  wires up the matching Pendo integration — PendoMCPServer (Python),
  initMcp() (TypeScript), gosdk.Instrument() (Go), PendoMcpServer.wrapAll()
  (Java, io.pendo:pendo-sdk-mcp), or WithPendoAgentAnalytics() (.NET,
  Pendo.AgentAnalytics.Mcp) — then verifies data is flowing to Pendo Agent
  Analytics. Use this whenever the user wants Pendo analytics on an MCP
  server, mentions "MCP analytics", "track MCP tool calls", "PendoMCPServer",
  "PendoMcpServer", "initMcp", "pendo-sdk-mcp", "WithPendoAgentAnalytics",
  "Pendo.AgentAnalytics.Mcp", "instrument my MCP server", or has an MCP
  server (including a Spring AI or a .NET ModelContextProtocol MCP server)
  and asks for Pendo/agent analytics — even if they don't say "MCP analytics" verbatim.
  NOT for instrumenting regular (non-MCP) agents — use setup-agent-analytics
  for that.
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

<!-- Generated from pendo-io/pendo-sdk skills/setup/setup-mcp-agent-analytics at b8b5b05. Edit it there; this copy is replaced at the next mirror. -->

# Set up Pendo MCP Analytics

Instrument an MCP server so tool calls, user intent, and client info flow to
Pendo Agent Analytics. The guiding contract, set by product: **a customer
installing for MCP analytics gets ONLY MCP analytics** — never silently
instrument their other LLM/agent code. This applies identically across all
five SDKs below.

There are five SDKs, one per language — your first job is to detect which
one applies.

This skill names no SDK version on purpose. Install the newest stable release
of the SDK and pin the exact version you installed in the project's manifest.
The text below holds for every release until the wiring itself changes.

---

## Phase 0: Identify the SDK (Python, TypeScript, Go, Java, or .NET)

Look at the project for the MCP framework in use:

| Signal | SDK |
|--------|-----|
| `pyproject.toml` / `requirements.txt` with `mcp`; `Server(` from `mcp.server.lowlevel`, `FastMCP(` from `mcp.server.fastmcp` (mcp 1.x) or `MCPServer(` from `mcp.server.mcpserver` (mcp 2.x) | **Python** — `pendo-server-sdk` (pip, `[mcp]` extra) |
| `package.json` with `@modelcontextprotocol/sdk`; `new McpServer(` or `new Server(` | **TypeScript** — `pendo-server-sdk` (npm) |
| `go.mod` with `github.com/modelcontextprotocol/go-sdk` (or another Go MCP framework, e.g. `mark3labs/mcp-go`) | **Go** — `github.com/pendo-io/pendo-sdk/sdks/go` |
| `pom.xml` / `build.gradle(.kts)` with `io.modelcontextprotocol.sdk:mcp` or a Spring AI `spring-ai-starter-mcp-server*`; `McpServer.sync(` / `McpServer.async(`, or `@McpTool` methods | **Java** — one of four `io.pendo` artifacts: a Spring AI starter (`pendo-sdk-spring-ai`, `-0x`) when Spring AI builds the server, otherwise the adapter (`pendo-sdk-mcp`, `-0x`). The Java section's first table picks it. |
| A `.csproj` with `PackageReference Include="ModelContextProtocol"` or `"ModelContextProtocol.AspNetCore"`; `AddMcpServer(`, `[McpServerToolType]`, or `McpServer.Create(` | **.NET** — `Pendo.AgentAnalytics.Mcp` (NuGet) |

If several MCP servers exist (possibly in different languages, e.g. a
monorepo), confirm which to instrument — each gets its own agent ID. State
the detected language in one line ("I'll use the **TypeScript SDK** — this
is an `@modelcontextprotocol/sdk` server"; for Java also name the artifact:
"I'll use the **Java SDK**, artifact `pendo-sdk-spring-ai-0x` — this is a
Spring AI 1.1 server on MCP SDK 0.18.3") and confirm before proceeding, unless the
user already named the language or only one server is present.

Then jump to the matching section below: **[Python](#python-pendo-server-sdk)**,
**[TypeScript](#typescript-pendo-server-sdk)**, **[Go](#go-githubcompendo-iopendo-sdksdksgo)**,
**[Java](#java-iopendo)**,
or **[.NET](#net-pendoagentanalyticsmcp)**.

---

## Phase 1: Collect the required values (all languages)

Ask, don't guess:

- **API key** — the Pendo application API key (the key in
  `POST /data/agenticsdk/<api_key>`). Called `api_key` / `apiKey` / `APIKey`
  / the app's **Public App ID** depending on SDK.
- **Agent ID** — the Agent Analytics agent this data routes to; the user
  creates it in the Pendo UI first (Product → Agent Analytics → settings
  icon next to the agent name).
- **Endpoint** — only for non-US-prod (EU / dev stacks). All five SDKs
  default to `https://app.pendo.io`, sending to
  `<endpoint>/data/agenticsdk/<api_key>`.

If the user wants the wiring validated before they've gathered real values,
wire it with obvious placeholders (e.g. `api_key="test-api-key"`) to prove
the code runs and the schema/event shape is correct, then swap in the real
values once supplied for an end-to-end check — two clearly separated passes,
not two rounds of guessing.

---

## Python (`pendo-server-sdk`)

### Find the MCP server

Look for the `mcp` package (`modelcontextprotocol` Python SDK): `Server(`
from `mcp.server.lowlevel` (low-level), `FastMCP(` from `mcp.server.fastmcp`
(mcp 1.x) or `MCPServer(` from `mcp.server.mcpserver` (mcp 2.x). All are
accepted — `PendoMCPServer` finds the low-level server itself.

### Install

Requires the `mcp` extra (`pip install "pendo-server-sdk[mcp]"`, Python >= 3.10).
In the project's dependency file, pin the newest release
(`pendo-server-sdk[mcp]==<version>`; `pip index versions pendo-server-sdk`
shows it).

### Wire it in

```python
from mcp.server.fastmcp import FastMCP  # mcp 1.x
# from mcp.server.mcpserver import MCPServer  # mcp 2.x
from pendo_sdk.mcp import PendoMCPServer

server = FastMCP("my-mcp")  # MCPServer("my-mcp") on mcp 2.x

PendoMCPServer(
    server=server,
    api_key="<app api key>",
    agent_id="<AA agent id>",
    # endpoint="<non-prod base URL>",
    # visitor_id=lambda req: req.headers.get("x-user-id") if req else None,
    # account_id="static-account",
)

@server.tool()
def my_tool(...):
    ...  # register before or after PendoMCPServer(...) — both fine
```

On the low-level API, pass the `Server` instance the same way:
`PendoMCPServer(server=my_lowlevel_server, api_key=..., agent_id=...)`.

Rules that matter:

- **MCP-only is the default.** `PendoMCPServer.__init__` calls
  `pendo_sdk.init(..., auto_patch=False)` for you — do NOT also call
  `pendo_sdk.init()` yourself for a regular agent in the same process (see
  "Single global init" below for why that's more than a style preference
  here). Even if the server's tools call OpenAI/Anthropic internally,
  `auto_patch=False` means those calls emit nothing.
- **Single global init — no multi-agent coexistence (unlike the TS SDK).**
  `pendo_sdk.init()` is process-wide, one-shot state: a second call (different
  `agent_id` or not) logs a warning and no-ops, silently keeping the *first*
  call's config. Concretely: if the customer already called `pendo_sdk.init()`
  for a separate regular agent before constructing `PendoMCPServer`, the
  wrapper's internal `init()` call will no-op — MCP events then get stamped
  with the *first* agent's `agent_id`/config, not their own. There is currently
  no supported way to run two independently-configured agents (regular +
  MCP, or two MCP servers with different `api_key`/`redact`/etc.) in one
  process. If the user needs both, instrument them in separate processes.
- **Advanced: customizing redact / batch_size / flush_interval_ms.**
  `PendoMCPServer` only forwards `api_key`, `agent_id`, `visitor_id`,
  `account_id`, `user_intent_description`, and `endpoint` to `init()` — there's
  no direct passthrough for `redact`, `batch_size`, or `flush_interval_ms`.
  To set those, call `pendo_sdk.init(api_key=..., agent_id=..., redact=True,
  auto_patch=False, ...)` yourself **before** constructing `PendoMCPServer`
  with the *same* `agent_id` — its internal `init()` call then no-ops (per
  the rule above) and your config sticks. Pass `auto_patch=False` in that
  call too, or you break the MCP-only contract.
- **stdio transport**: stdout is the JSON-RPC channel. Never `print()` in tool
  handlers or elsewhere in the process — it corrupts the protocol stream. To
  see the SDK's own logs (`pendo_sdk` logger, e.g. the
  `Exported N event(s) to ...` line on successful export), configure logging
  to stderr, which is safe: `logging.basicConfig(level=logging.INFO,
  stream=sys.stderr)`. Without this, Python's logging module stays silent for
  INFO-level SDK logs by default.
- **Streamable HTTP with per-session servers**: construct `PendoMCPServer(...)`
  for each session's server with the same `agent_id` — safe; one pipeline per
  process, sessions become separate conversations.
- **Identity resolvers**: `visitor_id`/`account_id` accept a static string or
  `Callable[[req], Optional[str]]`. `req` is the Starlette `Request` on
  HTTP/SSE transports (`req.headers`, `req.query_params`) and `None` on
  stdio (use static values there, or return `None`).
- **Shutdown**: `pendo_sdk.flush()` blocks synchronously until buffered spans
  are exported (or the internal timeout elapses) — no extra sleep needed
  before a short-lived process exits, unlike SDKs where flush is
  fire-and-forget. Call `pendo_sdk.flush()` (or `shutdown()`) once the
  server's `run()`/serve loop returns.

### Verify data is flowing

1. **Export log**: with logging configured to stderr (see above),
   `[pendo_sdk] Exported N event(s) to <endpoint>/data/agenticsdk/<key> [200, attempt 1]`.
   No line → check `api_key`, endpoint region, and that `PendoMCPServer(...)`
   was constructed before traffic (and that logging is actually configured —
   silence can mean "not logged", not "not exported"). A
   `SSL: CERTIFICATE_VERIFY_FAILED` error here is usually a local dev-machine
   issue, not a config one — common on macOS python.org framework installs
   that never ran "Install Certificates.command". Fix by pointing at
   `certifi`'s bundle rather than touching api_key/endpoint:
   `SSL_CERT_FILE=$(python3 -c "import certifi; print(certifi.where())") python my_server.py`.
2. **Drive real traffic — real JSON-RPC traffic, not `FastMCP`'s own Python
   methods.** Calling `await server.list_tools()` / `await server.call_tool()`
   directly looks like a fast sanity check but bypasses the instrumentation
   entirely: `PendoMCPServer` wraps the lowlevel `Server`'s `request_handlers`
   dict, which those convenience methods never go through. You'll see no
   injected `user_intent` and no virtual `get_additional_tools`, and
   (falsely) conclude the wiring is broken. Verify over the real protocol
   instead, with Claude Code as the MCP client:
   `claude -p "<use the tools>" --mcp-config <cfg> --allowedTools "mcp__<name>__*"`
   exercises `tools/list` + `tools/call` including the injected `user_intent`
   parameter, exactly like a customer's AI client. For a stdio server, `<cfg>`
   points at the command that runs it (e.g. `python my_server.py`).
3. **Wire-level assertions**: in a test, swap `PendoSpanExporter._send_jzb`
   for a list-capturing stub via `SimpleSpanProcessor` to inspect exact wire
   events (`agentTraceType`, `prompt`/`agent_response` content) without a
   network call.
4. **Latency expectations**: export-200 ≠ visible in the UI. Raw storage in
   minutes; hourly expansion ~8 min past the hour; AA aggregations after
   that. Budget **about an hour** end to end. Redaction is applied
   server-side per the AA agent's config — stored content may show
   `[city_1]`-style tokens; that's correct behavior, not data loss.

---

## TypeScript (`pendo-server-sdk`)

### Find the MCP server

Look for `@modelcontextprotocol/sdk` usage: `new McpServer(` (high-level) or
`new Server(` (low-level). Both are accepted — the wrapper unwraps
`McpServer.server` itself.

### Install

`npm install pendo-server-sdk` (tell the user; don't run it yourself unless asked).
When you add it to `package.json` by hand, use the newest release
(`npm view pendo-server-sdk version` shows it).

### Wire it in

```ts
import { initMcp } from "pendo-server-sdk";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({ name: "my-mcp", version: "1.0.0" });

initMcp({
  server,
  apiKey: "<app api key>",
  agentId: "<AA agent id>",
  // endpoint: "<non-prod base URL>",
  // visitorId: (ctx) => ctx?.requestInfo?.headers?.["x-user-id"] ?? null,
  // accountId: "static-account",
  // redact: true, batchSize: 512, flushIntervalMs: 2000,
});

server.registerTool(/* ... */);   // before or after initMcp — both fine
```

On published SDK versions that predate `initMcp` (≤ 0.9.19), use the class
form with the same options: `new PendoMCPServer({ server, apiKey, agentId, ... })`.
Check with `typeof pendo.initMcp === "function"`.

Rules that matter:

- **MCP-only is the default.** Do NOT also call `pendo.init()` — `initMcp()`
  is the complete setup and never patches LLM client libraries. Even if the
  server's tools call OpenAI/Anthropic internally, those calls emit nothing.
- **Coexistence** (the user explicitly wants a regular agent traced in the
  same process): call both `initMcp()` and `init()` with **different
  agentIds** — they must surface as separate agents in Pendo. Either order
  works; the pipeline is shared. Reusing one agentId triggers an SDK warning.
- **stdio transport**: stdout is the JSON-RPC channel. The SDK (>= 0.11.0)
  logs only to stderr; the server's own code must use `console.error`, never
  `console.log`. Older SDKs log to stdout: upgrade rather than redirect.
- **Streamable HTTP with per-session servers**: call `initMcp()` for each
  session's server with the same agentId — safe; one pipeline per process,
  sessions become separate conversations.
- **Identity resolvers** receive the tool handler's `extra` on HTTP
  transports (`extra.requestInfo.headers`, `extra.sessionId`) and `null` on
  stdio (use static values there).
- **ESM apps**: MCP instrumentation is instance-based and ESM-safe. The
  general `init()` auto-patching is NOT (it patches the CJS copy of
  openai/anthropic; ESM imports bypass it) — known SDK issue; relevant only
  to coexistence setups.
- **Shutdown**: `flush()`/`shutdown()` cover the MCP pipeline but return
  `void` — for short-lived processes call `flush()` and allow ~1s before exit.

### Verify data is flowing

1. **Export log** (stderr):
   `[pendo_sdk] Exported N event(s) to <endpoint>/data/agenticsdk/<key> [200, attempt 1]`.
   No line → check apiKey, endpoint region, and that the wrapper was
   constructed before traffic.
2. **Drive real traffic** with Claude Code as the MCP client:
   `claude -p "<use the tools>" --mcp-config <cfg> --allowedTools "mcp__<name>__*"`
   — exercises tools/list + tools/call including the injected `user_intent`
   parameter, exactly like a customer's AI client.
3. **Wire-level assertions**: the SDK's `e2e/` directory (`sdks/typescript/e2e`
   in `pendo-io/pendo-sdk`) has the full harness — a local capture sink
   (`sink.cjs`), a dogfood server, and `check.mjs` (agentId purity,
   zero-LLM-leakage). Point the endpoint at the sink to inspect exact wire
   events. See its `README.md`.
4. **Latency expectations**: export-200 ≠ visible in the UI. Raw storage in
   minutes; hourly expansion ~8 min past the hour; AA aggregations after
   that. Budget **about an hour** end to end. Redaction is applied
   server-side per the AA agent's config — stored content may show
   `[city_1]`-style tokens; that's correct behavior, not data loss.

---

## Go (`github.com/pendo-io/pendo-sdk/sdks/go`)

### Prerequisites

- Go 1.25 or later for a server on the official MCP Go SDK (the adapter
  module's floor); the core module on its own needs Go 1.23.
- Full conversations activated for the agent in Pendo (Product → Agent
  Analytics → Add and configure AI agents).
- The app's **Public App ID** (Settings → Subscription settings →
  Applications, subscription admin).
- The **Agent ID** (Product → Agent Analytics → settings icon next to the
  agent name).

### Find the MCP server

Look for `github.com/modelcontextprotocol/go-sdk` (`mcp.NewServer(`) — the
official Go MCP SDK. Other frameworks (e.g. `mark3labs/mcp-go`) are also
supported, but through the core client rather than the adapter — see Custom
integrations below.

### Install

The core SDK has no third-party dependencies. The MCP-framework adapter is a
separate module, so the framework dependency only enters the tree if it's used:

```bash
# Servers built on the official MCP Go SDK (github.com/modelcontextprotocol/go-sdk)
go get github.com/pendo-io/pendo-sdk/sdks/go/adapters/gosdk

# Core only (custom integrations and other MCP frameworks)
go get github.com/pendo-io/pendo-sdk/sdks/go
```

### Wire it in

Add one line after constructing the server. Every tool call is then captured
and sent to Pendo automatically — prompts, tool requests, tool responses,
and agent responses.

```go
import (
    "github.com/modelcontextprotocol/go-sdk/mcp"
    "github.com/pendo-io/pendo-sdk/sdks/go/adapters/gosdk"
)

s := mcp.NewServer(&mcp.Implementation{Name: "my-server", Version: "1.0.0"}, nil)
client := gosdk.Instrument(s, "your-pendo-app-id", "your-agent-id")
defer client.Flush(context.Background())
```

Events are sent asynchronously so a slow send never delays a tool response.
`Flush` waits for in-flight events at shutdown — important for short-lived
servers (stdio transports especially), which can otherwise exit before the
last events send. Long-running HTTP/SSE servers don't need it.

**Set visitor and account identity.** For a server that always acts for one user:

```go
gosdk.Instrument(s, appID, agentID,
    gosdk.WithVisitor("user@example.com"),
    gosdk.WithAccount("acme-corp"),
)
```

For a server that handles many users, resolve identity per request:

```go
gosdk.Instrument(s, appID, agentID,
    gosdk.WithIdentityFunc(func(ctx context.Context, req *mcp.CallToolRequest) (visitorID, accountID string) {
        user := auth.UserFromContext(ctx)
        return user.Email, user.AccountID
    }),
)
```

**User intent is captured for you.** The adapter adds a required
`user_intent` string property to every tool's advertised input schema and
strips it from the arguments before the tool's own handler runs, so the
calling model supplies the end user's question on every call and the SDK
reports it as the conversation prompt. Don't add an intent parameter to the
tool schemas yourself. A server instrumented with an older SDK may still have
callers sending a `userQuery` argument: it is accepted as a deprecated alias,
with a one-time warning.

**Conversation grouping.** Tool calls group into conversations by the MCP
session ID (Streamable HTTP/SSE). For stdio or in-process transports, which
have no session ID, the SDK falls back to one conversation per server run
and logs one deprecation warning (`falling back to a fixed per-process
conversation id`). `gosdk.WithRequireConversationID()` turns the fallback
off: a call with no session ID is then not recorded, and `WithOnEmit`
receives a `pendo.MissingConversationIDError`.

### Custom integrations (no MCP framework)

The core client works with any Go service — build the tool call and emit it:

```go
import pendo "github.com/pendo-io/pendo-sdk/sdks/go"

client := pendo.New(pendo.Config{
    APIKey:  "your-pendo-app-id",
    AgentID: "your-agent-id",
})

err := client.EmitToolCall(ctx, pendo.ToolCall{
    ConversationID: threadID, // same id for every turn — groups the conversation
    ToolName:       "get_weather",
    UserIntent:     userMessage,
    Arguments:      map[string]any{"location": "Denver"},
    Output:         toolOutput,
    VisitorID:      currentUser.Email,
    AccountID:      currentUser.PendoAccountID,
    StartedAt:      start,
    EndedAt:        time.Now(),
})
```

`EmitToolCall` sends synchronously and returns the delivery error. The async
variants (`EmitToolCallAsync`, `EmitFinalResponseAsync`) send off the request
path; call `client.Flush(ctx)` before the process exits to wait for them.

### Verify data is flowing

1. **Watch the emit outcome.** Register a callback to log delivery results
   during rollout:

   ```go
   gosdk.WithOnEmit(func(conversationID string, eventCount int, err error) {
       if err != nil {
           log.Printf("pendo emit failed: %v", err)
       }
   })
   ```

   Delivery failures are otherwise silent by default.
2. **Drive a real conversation** in dev/staging with a test prompt.
3. **Check Agent Analytics.** Conversation data is processed in hourly
   batches — after the code is live, new messages typically appear within
   15 minutes after the start of the next batch. Go to Product → Agent
   Analytics, select the agent, and open the Conversations tab.
4. **Check for trace data.** Conversations with trace data show an activity
   icon next to the relevant prompt/response; select it to open the
   Conversation trace view.

### Troubleshooting

| Check | What to look for |
|-------|-------------------|
| Missing or incorrect `agentId` | Required, and must match an Agent ID configured in Pendo for an agent belonging to the app the API key resolves to. |
| Wrong App ID or endpoint | The SDK sends to `https://app.pendo.io/data/agenticsdk/<apiKey>`. Verify `apiKey` is the Pendo Public App ID (an application key, not a subscription/integration key) and that outbound traffic to this endpoint is allowed. |
| Emit errors are invisible | Delivery failures are silent by default — add `WithOnEmit` (or check the error from `EmitToolCall`) to surface HTTP errors during setup. |
| No prompt events | Prompts come from the `user_intent` argument, which the adapter adds to every tool schema — confirm `tools/list` shows it and the calling model populates it. |
| Everything in one conversation (stdio), or a one-time warning about a fallback conversation id | stdio/in-process transports have no MCP session ID; the SDK groups by one fallback conversation per server run and warns once. Use Streamable HTTP for real per-session grouping, or `gosdk.WithRequireConversationID()` to record no call that lacks a session ID. |
| Last events of a run missing | Emission is asynchronous; a process that exits right after a tool call can drop the in-flight send. Call `client.Flush(ctx)` (on the client returned by `Instrument`) before exit. |

**Note:** the adapter records tool calls only. A `user_reaction` event
(thumbs up/down) is sent explicitly, with `client.EmitReaction(ctx,
pendo.Reaction{...})` on the client `Instrument` returns.

---

## Java (`io.pendo`)

### Prerequisites

- JDK 17 or later.
- The **Application API Key** and the **Agent ID** (Phase 1).

### Pick the artifact

Java has four artifacts and a server takes exactly one. Two facts choose it:
whether Spring AI builds the MCP server, and which MCP Java SDK generation
the build resolves.

```bash
mvn -q dependency:tree -Dincludes=io.modelcontextprotocol.sdk,org.springframework.ai   # or: gradle dependencies | grep -E 'modelcontextprotocol|springframework.ai'
```

| Who builds the server | Resolved `io.modelcontextprotocol.sdk:mcp*` | Add | Wiring |
|---|---|---|---|
| Spring AI 2.x (Spring Boot 4): a `spring-ai-starter-mcp-server*` dependency; `@McpTool` methods, `ToolCallback` beans or tool specification beans; no explicit builder | 1.0 or later (2.x today) | `io.pendo:pendo-sdk-spring-ai` | none: [the starter](#spring-ai-the-starter) |
| Spring AI 1.1+ (Spring Boot 3.5), same shape | 0.x (0.18.x) | `io.pendo:pendo-sdk-spring-ai-0x` | none: [the starter](#spring-ai-the-starter) |
| The application: `McpServer.sync(` / `McpServer.async(` with `.tools(...)` | 1.0 or later (2.x today) | `io.pendo:pendo-sdk-mcp` | [the adapter](#any-other-server-the-adapter) |
| The application, same shape | 0.x (0.18.x) | `io.pendo:pendo-sdk-mcp-0x` | [the adapter](#any-other-server-the-adapter) |

Spring AI 1.0.x is not supported (its MCP Java SDK is older than the 0.18.3
the SDK needs): upgrade to Spring AI 1.1 or later first. Name the artifact in
the one-line confirmation (Phase 0).

### Install

Add the one artifact picked above (tell the user; don't run the build
yourself unless asked). The SDK is published to Pendo's Maven repository on
JFrog, `https://pendo.jfrog.io/artifactory/pendo-sdk-release`, not to Maven
Central, so the build needs the repository entry as well as the dependency,
and the entry stays. Put it in the build that resolves the dependency (with
Gradle that may be `settings.gradle(.kts)` when `dependencyResolutionManagement`
is in use) and, where the build tool allows, scope it to the `io.pendo` group
so nothing else is looked up there. Pin the exact release: the newest stable
version listed in
`https://pendo.jfrog.io/artifactory/pendo-sdk-release/io/pendo/pendo-sdk-mcp/maven-metadata.xml`
(the highest one with no suffix; every `io.pendo` artifact carries the same
number). `pendo-sdk-core` arrives transitively; the MCP SDK is `provided`, so
the customer's own version is the one used. A version with a suffix is a
pre-release build, offered for trying a fix; never pin a customer build to
one.

```xml
<repositories>
  <repository>
    <id>pendo-sdk-release</id>
    <url>https://pendo.jfrog.io/artifactory/pendo-sdk-release</url>
    <snapshots><enabled>false</enabled></snapshots>
  </repository>
</repositories>

<dependency>
  <groupId>io.pendo</groupId>
  <artifactId>pendo-sdk-mcp</artifactId>   <!-- the artifact picked above -->
  <version>X.Y.Z</version>   <!-- the newest stable release, see above -->
</dependency>
```

```kotlin
repositories {
    mavenCentral()
    exclusiveContent {
        forRepository { maven { url = uri("https://pendo.jfrog.io/artifactory/pendo-sdk-release") } }
        filter { includeGroup("io.pendo") }
    }
}

dependencies {
    implementation("io.pendo:pendo-sdk-mcp:X.Y.Z")   // the artifact picked above, at the newest stable release
}
```

### Spring AI: the starter

The dependency is the whole integration: write no code. At startup the
starter creates one `PendoMcpServer` bean, wraps every tool specification
list Spring AI collects (sync, async and stateless) and adds
`get_additional_tools` once.

Configure it with properties:

```properties
pendo.api-key=<app api key>
pendo.agent-id=<AA agent id>
# pendo.endpoint=<non-prod base URL>
# pendo.visitor-id=static-visitor
# pendo.account-id=static-account
```

or with the environment variables Spring binds to the same properties:
`PENDO_API_KEY`, `PENDO_AGENT_ID`, `PENDO_ENDPOINT`, `PENDO_VISITOR_ID`,
`PENDO_ACCOUNT_ID`. With no key or no agent id from either source the
application still starts: the SDK is disabled and logs one warning naming
what is missing.

- **Two off switches.** `pendo.spring-ai.enabled=false` turns the starter
  off, so nothing is wrapped. `pendo.disabled=true` (or `PENDO_DISABLED`)
  keeps the tools wrapped and sends nothing.
- **Per-request identity, redaction and every other builder option** go
  through a `PendoConfigCustomizer` bean, applied after the properties:

  ```java
  @Bean
  PendoConfigCustomizer pendoIdentity() {
      return builder -> builder
              .visitorIdResolver(context -> (String) context.transportAttribute("x-user-id"));
  }
  ```

  Keep it free of dependencies such as repositories: it is created while
  Spring is still post-processing beans. Spring AI's HTTP transports set no
  context extractor, so `transportAttribute(...)` holds a header only when the
  application replaces the transport bean with one that sets a
  `contextExtractor`; without that, HTTP callers are recorded as `anonymous`.
- **An application-declared `PendoMcpServer` bean wins:** the starter creates
  none and wraps with that one. A tool the application already wrapped is not
  wrapped twice.
- **Tools added after startup** with `server.addTool(...)` bypass the starter:
  inject the `PendoMcpServer` bean and call `server.addTool(pendo.wrap(spec))`.

### Any other server: the adapter

Create one `PendoMcpServer` per process and pass the tool specifications
through it on the way into the server builder. Wrapping the tools is the
whole integration.

```java
import io.pendo.sdk.PendoConfig;
import io.pendo.sdk.mcp.PendoMcpServer;

// The key and agent id come from PENDO_API_KEY and PENDO_AGENT_ID (PENDO_ENDPOINT for a non-prod stack).
PendoMcpServer pendo = PendoMcpServer.create(PendoConfig.builder()
        // .apiKey("<app api key>").agentId("<AA agent id>")   // or set them here; a value set in code wins
        // .endpoint("<non-prod base URL>")
        // .visitorId("static-visitor").accountId("static-account")
        // .visitorIdResolver(ctx -> (String) ctx.transportAttribute("user-id"))
        // .redact(true).requireConversationId(true).onDeliveryFailure(f -> ...)
        .build());

McpSyncServer server = McpServer.sync(transportProvider)
        .serverInfo("my-mcp", "1.0.0")
        .capabilities(ServerCapabilities.builder().tools(true).build())
        .tools(pendo.wrapAll(echoTool(), addTool()))   // was: .tools(echoTool(), addTool())
        .build();
```

With no key or no agent id from either source, `build()` still succeeds: the
SDK is disabled, logs one warning naming what is missing, and the server runs
as before. So write no environment lookup and no "skip when unset" branch of
your own.

- **Specs are wrapped, not the handler layer.** Unlike the other four SDKs,
  a tool that reaches the server without going through `wrap` / `wrapAll` is
  invisible to Pendo. Tools added at runtime: `server.addTool(pendo.wrap(spec))`;
  removed: `server.removeTool(name)` then `pendo.forgetTool(name)`. `wrap` and
  `wrapAll` take sync, async and stateless specifications; the list forms are
  `wrapSync`, `wrapAsync`, `wrapStatelessSync` and `wrapStatelessAsync`.
- **`get_additional_tools` exactly once.** `wrapAll` and the list forms
  append the SDK's virtual tool to every batch they wrap, and the MCP server
  rejects a duplicate tool name at build time. Wrap in one batch, or `wrap`
  each spec and register `pendo.virtualTool()` (stateless:
  `pendo.statelessVirtualTool()`) once.
- **One `PendoMcpServer` per process.** Each `create` starts its own delivery
  thread and shutdown hook. Streamable HTTP with per-session servers: wrap
  every session's tools with the same instance, same agentId.

### Rules for both

- **`user_intent` is required, and the 2.x server validates by default.** A
  host that omits it is refused before the handler runs. That is the intended
  behavior; `.validateToolInputs(false)` on the server builder is the escape
  hatch. The 0.x generation never validates arguments.
- **Tests that pin the tool list.** Instrumentation changes what the server
  advertises: every tool gains a required `user_intent` property, and one
  tool, `get_additional_tools`, is added. Keep a test that asserts the exact
  catalogue on the uninstrumented server: with the starter,
  `@SpringBootTest(properties = "pendo.spring-ai.enabled=false")` on that test
  (`pendo.disabled=true` does not restore the catalogue); with the adapter,
  build the test's server from the unwrapped specifications.
- **MCP-only.** Only wrapped tools are instrumented; nothing patches LLM
  client libraries.
- **stdio transport:** stdout is the JSON-RPC channel. Route all logging to
  stderr or a file (SLF4J Simple: `org.slf4j.simpleLogger.logFile=System.err`;
  Spring Boot: `logging.pattern.console=` plus `logging.file.name=...`).
- **Identity resolvers** run once per `tools/call`, before the handler, on the
  request thread. `RequestContext.transportAttribute(key)` reads what the
  transport's `contextExtractor` stored. Null/blank → the static value; a
  throw is logged and never affects the call. stdio has no request context:
  use static values or env vars. A blank visitor goes on the wire as
  `anonymous`, with a one-time WARN (`pendo: tool call "…" has no visitor
  configured or resolved; recording visitorId as "anonymous"`).
- **Conversation id = MCP session id.** No session id (stdio, or a client
  that sends none) → the call is recorded with an empty conversation id plus
  a one-time deprecation warning (`recorded without a conversationId`);
  `.requireConversationId(true)` skips such calls instead. No fallback id is
  generated.
- **Shutdown:** nothing to register. A JVM hook drains for up to 10 s (SIGTERM
  fine, SIGKILL not). Call `pendo.close()` when the server is disposed without
  exiting the JVM. Knobs: `batchSize` 512, `flushInterval` 2 s,
  `maxQueueSize` 2048, `shutdownTimeout` 10 s.

### Verify data is flowing

1. **Quiet on success, loud on failure.** The Java SDK logs no success line.
   Failures go to `System.Logger` `io.pendo.sdk` at WARNING (the JDK default
   prints them to stderr; with SLF4J add `org.slf4j:slf4j-jdk-platform-logging`).
   After a few tool calls, the only `pendo:` lines should be the one-time
   advisories (no visitor configured; no session id); any line mentioning
   `ingestion`, `delivery` or `discarded` is a failed batch. A rejected key:
   `pendo: ingestion at <endpoint> rejected the API key (HTTP 401 Unauthorized); check PendoConfig.Builder.apiKey; N event(s) discarded`.
   Don't trust silence alone — confirm the logger is actually routed.
2. **Drive real traffic** over the protocol (MCP Inspector, or Claude Code
   with `--mcp-config`), so `tools/list` shows `user_intent` and
   `get_additional_tools` and `tools/call` carries the intent.
3. **Wire-level assertions:** point the endpoint at a local sink and read
   the `/data/agenticsdk/<key>` requests (`contract/harness/sink.mjs` in
   `pendo-io/pendo-sdk`, or any HTTP server). Expect `tool_request` /
   `tool_response` pairs with `user_intent` stripped from the recorded
   arguments and the catalog attached once per conversation.
4. **`onDeliveryFailure`** for code-level visibility: the callback receives
   `DeliveryFailure(endpoint, eventCount, attempts, statusCode, responseDetail, cause, conversationIds)`.
5. **Latency expectations:** as for the other SDKs — a 2xx is acceptance, not
   persistence (a wrong `agentId` is dropped server-side silently); budget
   about an hour before the UI reflects new data.

### Troubleshooting

| Check | What to look for |
|-------|-------------------|
| `Could not find artifact io.pendo:…` at build time | The `pendo-sdk-release` repository entry is missing from the build that resolves the dependency (Gradle: the module's `repositories {}`, or `settings.gradle(.kts)` when `dependencyResolutionManagement` owns them), a Gradle content filter excludes `io.pendo`, or a proxy blocks `pendo.jfrog.io`. |
| `pendo: no API key … the SDK is disabled and will send nothing` at startup | The key or the agent id is unset: `PENDO_API_KEY` / `PENDO_AGENT_ID`, the `pendo.api-key` / `pendo.agent-id` properties, or the builder. |
| `IllegalArgumentException` from `PendoConfig.build()` | A value is present but invalid: whitespace inside the API key, or an endpoint that is not an http(s) URL. |
| Spring AI application, and `tools/list` shows no `user_intent` | The starter is off: `pendo.spring-ai.enabled=false`, the other generation's starter, Spring AI 1.0.x, or no Spring AI MCP server starter on the classpath. |
| `pendo: … rejected the API key (HTTP 401 …)` | Wrong Application API key, or the endpoint doesn't match the region. |
| No delivery warnings and nothing in Pendo | Aggregation lag (about an hour), or `agentId` matches no agent in the app (accepted, then dropped silently). Prove events leave the process with a local sink. |
| `pendo: tool call "…" has no visitor configured or resolved; recording visitorId as "anonymous"` | Advisory, once per process: events are attributed to visitor `anonymous`. Set a visitor id or a `visitorIdResolver` to attribute them to real users. |
| `NoSuchMethodError` / `ClassNotFoundException` in `io.modelcontextprotocol` at startup | Wrong generation: re-check `dependency:tree` and swap to the artifact with or without `-0x`. |
| Server build fails on a duplicate tool `get_additional_tools` | Tools were wrapped in two batches; see the "exactly once" rule. |
| The host's tool calls are rejected as invalid arguments | 2.x validation plus required `user_intent`: fix the host, or `validateToolInputs(false)`. |
| Empty conversation ids / a one-time warning `recorded without a conversationId` | No session id (stdio, or the client sends none); use Streamable HTTP with session ids. |
| Last events of a run missing | The process exited before the hook drained; `pendo.close()` before exit, or a larger `shutdownTimeout`. |

---

## .NET (`Pendo.AgentAnalytics.Mcp`)

### Prerequisites

- .NET 8 or .NET 10.
- The official `ModelContextProtocol` NuGet package, 2.3.0 or later.
- The **Application API Key** and the **Agent ID** (Phase 1).

### Find the MCP server

Look for a `.csproj` with `PackageReference Include="ModelContextProtocol"`
(or `ModelContextProtocol.AspNetCore`). Hosted servers register with
`services.AddMcpServer()` and chain `.WithStdioServerTransport()` /
`.WithHttpTransport()`, `.WithTools<T>()` or `.WithToolsFromAssembly()`; tools
are `[McpServerToolType]` classes with `[McpServerTool]` methods. Servers built
by hand call `McpServer.Create(transport, options)` with an `McpServerOptions`.
Check the resolved version (`dotnet list package | grep ModelContextProtocol`);
the SDK needs 2.3.0 or later.

### Install

Add the one package (tell the user; don't run the build yourself unless
asked). It is published to Pendo's JFrog NuGet repository, which allows
anonymous reads, so the install names that source. Without `--version`,
`dotnet add package` takes the newest stable release from that source and
writes the exact version into the project file; leave it pinned there.
`Pendo.AgentAnalytics` arrives transitively, and the customer's own
`ModelContextProtocol` package is the one used. A
pre-release build, when one is offered for trying a fix, needs `--prerelease`
and its exact version; never pin a customer build to one.

```bash
dotnet add package Pendo.AgentAnalytics.Mcp --source https://pendo.jfrog.io/artifactory/api/nuget/v3/pendo-sdk-nuget-release/index.json
```

### Wire it in

**Hosted servers** (`AddMcpServer()`): add one line to the builder chain. It
covers every tool registered on the builder, before or after the call, and
flushes queued events when the host stops.

```csharp
builder.Logging.AddConsole(o => o.LogToStandardErrorThreshold = LogLevel.Trace);

builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly()
    .WithPendoAgentAnalytics();   // reads PENDO_API_KEY / PENDO_AGENT_ID from the environment
```

Settings come from the environment (`PENDO_API_KEY`, `PENDO_AGENT_ID`, and
optionally `PENDO_ENDPOINT`, `PENDO_VISITOR_ID`, `PENDO_ACCOUNT_ID`,
`PENDO_DISABLED`); anything set in code wins:

```csharp
.WithPendoAgentAnalytics(options =>
{
    options.ApiKey = "<app api key>";
    options.AgentId = "<AA agent id>";
    // options.Endpoint = "<non-prod base URL>";
    // options.VisitorId = "static-visitor"; options.AccountId = "static-account";
    // options.VisitorIdResolver = context => context.Services?.GetService<IHttpContextAccessor>()?.HttpContext?.User.Identity?.Name;
    // options.CallerAgentResolver = context => context.Services?.GetService<IHttpContextAccessor>()?.HttpContext?.Request.Headers.UserAgent.ToString();
    // options.Redact = true; options.RequireConversationId = true;
})
```

**Servers built with `McpServer.Create(...)`**: create one `PendoMcpServer` and
apply it to the options before the server is created.

```csharp
using ModelContextProtocol.Server;
using Pendo.AgentAnalytics;
using Pendo.AgentAnalytics.Mcp;

using var pendo = PendoMcpServer.Create(PendoConfig.From(new PendoOptions
{
    ApiKey = "<app api key>",
    AgentId = "<AA agent id>",
}));

var options = new McpServerOptions { /* server info, tools */ };
pendo.Apply(options);
await using var server = McpServer.Create(transport, options);
```

Rules that matter:

- **Filters, not wrapped tools.** `WithPendoAgentAnalytics()` and `Apply` add
  request filters, so tools registered on the options or builder are
  instrumented without wrapping. `Apply` must run before the server is created.
- **`get_additional_tools` is added for you.** A host tool registered under
  that name is replaced, with a warning.
- **`user_intent` is required.** The SDK adds it to every tool's input schema
  and removes it before the handler runs.
- **Identity.** `PENDO_VISITOR_ID` / `PENDO_ACCOUNT_ID` are defaults for the
  hosted one-liner only; `PendoMcpServer.Create` takes the options it is given.
  `VisitorIdResolver` / `AccountIdResolver` / `CallerAgentResolver` receive a
  `RequestContext` (`Services`, `Meta`, `CallerAgent`, `ConversationId`,
  `ToolName`, `Arguments`) once per `tools/call`, before the handler. Null/blank
  → the static value (the handshake client info for `CallerAgentResolver`); a
  throw is logged and never affects the call. A blank visitor goes on the wire
  as `anonymous`, with a one-time warning. Over HTTP, record the request's
  `User-Agent` as the caller agent: call `services.AddHttpContextAccessor()` in
  the host, then set `CallerAgentResolver = context => context.Services?.GetService<IHttpContextAccessor>()?.HttpContext?.Request.Headers.UserAgent.ToString()`.
- **Conversation id = MCP session id.** stdio has none, so one conversation is
  used for the life of the process. An HTTP transport without a session id gets
  a fresh conversation for every call; enable sessions to group calls.
- **Caller agent** is the client name and version from the `initialize`
  handshake unless `CallerAgentResolver` returns a value; over HTTP, set it to
  the `User-Agent` as in the Identity rule above.
- **MCP-only.** Nothing patches LLM client libraries.
- **stdio transport:** stdout is the JSON-RPC channel. The default console
  logger writes to stdout, so anything it logs, including the SDK's own startup
  warning, would corrupt the protocol: keep the `AddConsole(o =>
  o.LogToStandardErrorThreshold = LogLevel.Trace)` line from the hosted snippet.
- **Shutdown:** hosted servers flush when the host stops. With
  `McpServer.Create`, dispose the `PendoMcpServer` (or call `Flush()`) before
  exit. Knobs on `PendoOptions`: `BatchSize` 512, `FlushInterval` 2 s,
  `MaxQueueSize` 2048, `ShutdownTimeout` 10 s.

### Verify data is flowing

1. **Quiet on success, loud on failure.** The .NET SDK logs no success line.
   Warnings go to the host's `ILoggerFactory` (standard error when none is
   supplied). A missing key or agent id logs one warning at startup
   (`disabled: no API key … no events will be sent`) and the SDK stays
   disabled; a failed batch logs a warning naming the endpoint and the number
   of events discarded.
2. **Drive real traffic** over the protocol (MCP Inspector, or Claude Code with
   `--mcp-config`), so `tools/list` shows `user_intent` and
   `get_additional_tools` and `tools/call` carries the intent.
3. **Wire-level assertions:** set `PENDO_ENDPOINT` to a local sink
   (`contract/harness/sink.mjs` in `pendo-io/pendo-sdk`, then
   `curl <base url>/events`). Expect
   the prompt, a `tool_request` / `tool_response` pair and the agent response
   per call, one `conversationId` per session, `callerAgent` set to the
   client's name and version, and `user_intent` absent from the recorded
   arguments.
4. **`OnDeliveryFailure`** on `PendoOptions` for code-level visibility: it
   receives a `DeliveryFailure` when a batch is dropped.
5. **Latency expectations:** as for the other SDKs — a 2xx is acceptance, not
   persistence; budget about an hour before Agent Analytics reflects new data.

### Troubleshooting

| Check | What to look for |
|-------|-------------------|
| Restore can't find `Pendo.AgentAnalytics.Mcp` | the JFrog NuGet source was not passed (`--source` on the install, or add it with `dotnet nuget add source`; check `dotnet nuget list source`), or the pinned version is a pre-release and `--prerelease` was not passed. |
| `disabled: no API key …` warning at startup | `PENDO_API_KEY` / `PENDO_AGENT_ID` unset (or `ApiKey` / `AgentId` in code); the SDK stays disabled. |
| `ArgumentException` at startup | A present value is invalid: whitespace in the key or agent id, or `PENDO_ENDPOINT` not an `http(s)` URL. |
| Nothing recorded and no warning | `PENDO_DISABLED` is `true` or `1`, or `Disabled = true` in code. |
| Delivery warnings naming the endpoint | Wrong key, wrong region endpoint, or no network access from the server. Prove events leave the process with a local sink. |
| Every event is visitor `anonymous` | Set `PENDO_VISITOR_ID`, `VisitorId`, or a `VisitorIdResolver`. |
| A new conversation for every call over HTTP | The transport issues no session id; enable sessions. |
| Last events of a run missing | The process exited before the queue drained; stop the host cleanly, or `Flush()` and dispose the `PendoMcpServer`. |

---

## What the instrumentation does (explain to the user)

Across all five SDKs, tool calls are intercepted and recorded:

- **`tools/list`** — every SDK injects a required `user_intent` string
  parameter into every tool schema, so the AI client explains why it's
  calling. Tool schemas are otherwise unchanged.
- **`tools/call`** — times the call, strips the injected parameter before
  the real handler runs, and emits `prompt` / `tool_request` /
  `tool_response` / `agent_response` events. Handler behavior is untouched.
- Every SDK also advertises a virtual `get_additional_tools` tool that
  records capability gaps as `missing_capability` events.

Where the hook sits differs. Python, TypeScript, Go and .NET wrap the
request-handler layer (.NET through the server's request filters), so tools registered after the wrapper is constructed
are instrumented too. Java wraps each tool *specification* (the MCP Java SDK
has no `tools/list` hook), so a tool must go through `wrap` / `wrapAll` to be
seen — including tools added later with `addTool`. The Spring AI starter does
that wrapping for the tools Spring AI registers.
