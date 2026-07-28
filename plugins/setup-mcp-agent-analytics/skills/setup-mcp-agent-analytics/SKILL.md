---
name: setup-mcp-agent-analytics
description: >-
  Instrument an MCP server with Pendo MCP analytics in Python, TypeScript, or
  Go. Detects which language/SDK the server is built with and wires up the
  matching Pendo integration — PendoMCPServer (Python), initMcp() (TypeScript),
  or gosdk.Instrument() (Go) — then verifies data is flowing to Pendo Agent
  Analytics. Use this whenever the user wants Pendo analytics on an MCP
  server, mentions "MCP analytics", "track MCP tool calls", "PendoMCPServer",
  "initMcp", "instrument my MCP server", or has an MCP server and asks for
  Pendo/agent analytics — even if they don't say "MCP analytics" verbatim.
  NOT for instrumenting regular (non-MCP) agents — use setup-agent-analytics
  for that.
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

# Set up Pendo MCP Analytics

Instrument an MCP server so tool calls, user intent, and client info flow to
Pendo Agent Analytics. The guiding contract, set by product: **a customer
installing for MCP analytics gets ONLY MCP analytics** — never silently
instrument their other LLM/agent code. This applies identically across all
three SDKs below.

There are three SDKs, one per language — your first job is to detect which
one applies.

---

## Phase 0: Identify the SDK (Python, TypeScript, or Go)

Look at the project for the MCP framework in use:

| Signal | SDK |
|--------|-----|
| `pyproject.toml` / `requirements.txt` with `mcp`; `Server(` from `mcp.server.lowlevel` or `FastMCP(` from `mcp.server.fastmcp` | **Python** — `pendo-server-sdk` (pip, `[mcp]` extra) |
| `package.json` with `@modelcontextprotocol/sdk`; `new McpServer(` or `new Server(` | **TypeScript** — `pendo-server-sdk` (npm) |
| `go.mod` with `github.com/modelcontextprotocol/go-sdk` (or another Go MCP framework, e.g. `mark3labs/mcp-go`) | **Go** — `github.com/pendo-io/go-sdk` |

If several MCP servers exist (possibly in different languages, e.g. a
monorepo), confirm which to instrument — each gets its own agent ID. State
the detected language in one line ("I'll use the **TypeScript SDK** — this
is an `@modelcontextprotocol/sdk` server") and confirm before proceeding,
unless the user already named the language or only one server is present.

Then jump to the matching section below: **[Python](#python-pendo-server-sdk)**,
**[TypeScript](#typescript-pendo-server-sdk)**, or **[Go](#go-github-compendo-iogo-sdk)**.

---

## Phase 1: Collect the required values (all languages)

Ask, don't guess:

- **API key** — the Pendo application API key (the key in
  `POST /data/agenticsdk/<api_key>`). Called `api_key` / `apiKey` / `APIKey`
  / the app's **Public App ID** depending on SDK.
- **Agent ID** — the Agent Analytics agent this data routes to; the user
  creates it in the Pendo UI first (Product → Agent Analytics → settings
  icon next to the agent name).
- **Endpoint** — only for non-US-prod (EU / dev stacks). All three SDKs
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
from `mcp.server.lowlevel` (low-level) or `FastMCP(` from `mcp.server.fastmcp`
(high-level). Both are accepted — `PendoMCPServer` unwraps `FastMCP._mcp_server`
itself.

### Install

Requires the `mcp` extra (`pip install "pendo-server-sdk[mcp]"`, Python >= 3.10).

### Wire it in

```python
from mcp.server.fastmcp import FastMCP
from pendo_sdk.mcp import PendoMCPServer

server = FastMCP("my-mcp")

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
   instead
3. **Wire-level assertions**: there's no packaged e2e harness in this repo
   (unlike `typescript-be-sdk`'s `e2e/`) — `tests/test_mcp.py` shows the
   pattern instead: swap `PendoSpanExporter._send_jzb` for a list-capturing
   stub via `SimpleSpanProcessor` to inspect exact wire events
   (`agentTraceType`, `prompt`/`agent_response` content) without a network call.
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
- **stdio transport**: stdout is the JSON-RPC channel. At the very top of the
  entry file: `console.log = console.error; console.debug = console.error;`
  — also how you see the SDK's export logs.
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

1. **Export log** (stderr after the console redirect):
   `[pendo_sdk] Exported N event(s) to <endpoint>/data/agenticsdk/<key> [200, attempt 1]`.
   No line → check apiKey, endpoint region, and that the wrapper was
   constructed before traffic.
2. **Drive real traffic** with Claude Code as the MCP client:
   `claude -p "<use the tools>" --mcp-config <cfg> --allowedTools "mcp__<name>__*"`
   — exercises tools/list + tools/call including the injected `user_intent`
   parameter, exactly like a customer's AI client.
3. **Wire-level assertions**: this repo's `e2e/` directory has the full
   harness — a local capture sink (`sink.cjs`), a dogfood server, and
   `check.mjs` (agentId purity, zero-LLM-leakage). Point the endpoint at the
   sink to inspect exact wire events. See `e2e/README.md`.
4. **Latency expectations**: export-200 ≠ visible in the UI. Raw storage in
   minutes; hourly expansion ~8 min past the hour; AA aggregations after
   that. Budget **about an hour** end to end. Redaction is applied
   server-side per the AA agent's config — stored content may show
   `[city_1]`-style tokens; that's correct behavior, not data loss.

---

## Go (`github.com/pendo-io/go-sdk`)

### Prerequisites

- Go 1.23 or later.
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
go get github.com/pendo-io/go-sdk/adapters/gosdk

# Core only (custom integrations and other MCP frameworks)
go get github.com/pendo-io/go-sdk
```

### Wire it in

Add one line after constructing the server. Every tool call is then captured
and sent to Pendo automatically — prompts, tool requests, tool responses,
and agent responses.

```go
import (
    "github.com/modelcontextprotocol/go-sdk/mcp"
    "github.com/pendo-io/go-sdk/adapters/gosdk"
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

**Capture user intent.** The SDK reports the end user's question as the
conversation prompt by reading a `userQuery` string argument from each tool
call. Add a `userQuery` parameter to the tool schemas (described so the
calling model fills it in); the SDK strips it from the recorded tool
arguments and surfaces it as the prompt. Tool calls without `userQuery` still
produce trace and response events, just no prompt event.

**Conversation grouping.** Tool calls group into conversations by the MCP
session ID (Streamable HTTP/SSE). For stdio or in-process transports, which
have no session ID, the SDK falls back to one conversation per server run.

### Custom integrations (no MCP framework)

The core client works with any Go service — build the tool call and emit it:

```go
import pendo "github.com/pendo-io/go-sdk"

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
| No prompt events | Prompts come from the `userQuery` tool argument — confirm tool schemas include it and the calling model populates it. |
| Everything in one conversation (stdio) | stdio/in-process transports have no MCP session ID; the SDK groups by one fallback conversation per server run. Use Streamable HTTP for real per-session grouping. |
| Last events of a run missing | Emission is asynchronous; a process that exits right after a tool call can drop the in-flight send. Call `client.Flush(ctx)` (on the client returned by `Instrument`) before exit. |

**Note:** `user_reaction` events (thumbs up/down) are not yet supported in
the Go SDK. Use the Conversations API or the Python/TypeScript SDKs if
reactions need to be recorded.

---

## What the instrumentation does (explain to the user)

Across all three SDKs, the wrapper intercepts the MCP request handlers:

- **`tools/list`** — injects a required `user_intent` (Python/TS) or
  `userQuery` (Go) string parameter into every tool schema, so the AI client
  explains why it's calling. Tool schemas are otherwise unchanged.
- **`tools/call`** — times the call, strips the injected parameter before
  the real handler runs, and emits `prompt` / `tool_request` /
  `tool_response` / `agent_response` events. Handler behavior is untouched.
- Python/TS also advertise a virtual `get_additional_tools` tool that
  records capability gaps as `missing_capability` events (not currently
  present in the Go SDK).

Because instrumentation wraps the request-handler layer, tools registered
after the wrapper is constructed are instrumented too, in all three SDKs.
