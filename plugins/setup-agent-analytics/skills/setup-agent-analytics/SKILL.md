---
name: setup-agent-analytics
description: Detect AI agents in a codebase and instrument them with Pendo trackAgent() calls or server-side Conversations API
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---
You are setting up Pendo agent analytics for this codebase. Your job is to (1) detect the project framework and environment, (2) detect conversational AI agents, then (3) instrument them with either client-side `window.pendo.trackAgent()` calls or server-side Conversations API calls, depending on the environment.
Arguments: `$ARGUMENTS` may contain one or more Pendo agent IDs (space-separated). If provided, use them when instrumenting (first ID → first agent, second ID → second agent, etc.). If there are fewer IDs than detected agents, use `"YOUR_AGENT_ID"` as a placeholder for the extras. If no IDs are provided, use `"YOUR_AGENT_ID"` for all agents and tell the user to replace them after creating agent IDs in the Pendo UI.
---
## Phase 1: Detect Project Framework and Environment
Before scanning for agents, identify the project's framework, structure, and **runtime environment**. This determines both where to search for agents and which instrumentation method to use.
### Step 1: Identify the Framework
Check for:
- **Next.js**: `next.config.*`, `app/` or `pages/` directories
- **Nuxt/Vue**: `nuxt.config.*`, `.vue` files
- **Angular**: `angular.json`, `.component.ts` files
- **SvelteKit**: `svelte.config.*`
- **Plain React**: `react` in `package.json` without a meta-framework
- **Express/Fastify/Nest**: Server-only backends with API routes
- **Monorepo**: `packages/`, `apps/`, or workspace config in `package.json`
- **React Native**: `react-native` in `package.json`, `ios/` and `android/` directories, `app.json` with React Native config
- **Flutter**: `pubspec.yaml` with `flutter` dependency, `lib/main.dart`
- **iOS Native (Swift/ObjC)**: `.xcodeproj` or `.xcworkspace`, `AppDelegate.swift`, `Info.plist`
- **Android Native (Kotlin/Java)**: `build.gradle` with `com.android.application`, `AndroidManifest.xml`
- **Expo**: `app.json` with `expo` config, `expo` in `package.json`
- **Browser Extension**: `manifest.json` with `manifest_version`, `background` scripts, `content_scripts`
Use the framework to guide where you search:
| Framework | Likely agent locations |
|-----------|----------------------|
| Next.js (App Router) | `app/api/**/route.ts`, client components in `app/` or `components/` |
| Next.js (Pages Router) | `pages/api/**`, components in `components/` or `src/` |
| Nuxt | `server/api/**`, components in `components/` |
| Plain React / Vue / Angular | `src/components/`, `src/services/`, `src/api/` |
| Express / Fastify / Nest | `src/routes/`, `src/controllers/`, `src/services/` |
| React Native / Expo | `src/`, `app/`, `screens/`, `services/`, `api/` |
| Flutter | `lib/services/`, `lib/providers/`, `lib/api/` |
| iOS Native | `Services/`, `Networking/`, `ViewModels/` |
| Android Native | `data/`, `repository/`, `viewmodel/`, `network/` |
| Browser Extension | `background/`, `src/`, `content/` |
### Step 2: Classify the Instrumentation Method
Based on the detected framework, determine which tracking method to use:
| Environment | Method | Reason |
|-------------|--------|--------|
| Web apps (React, Vue, Angular, Next.js, Nuxt, SvelteKit, plain HTML/JS) | **Client-side** (`window.pendo.trackAgent`) | Pendo snippet runs in the browser |
| Mobile apps (React Native, Flutter, iOS native, Android native, Expo) | **Server-side** (Conversations API) | No client-side snippet support |
| Browser extensions | **Server-side** (Conversations API) | Extension contexts don't reliably support the Pendo snippet |
| Server-only backends (Express, Fastify, Nest) with no frontend | **Server-side** (Conversations API) | No browser environment |
| Hybrid (web frontend + API backend) | **Client-side** for the frontend | Instrument the client-side caller, not the backend |
Record the instrumentation method — you will use it in Phase 3 to select the correct helper and implementation pattern.
Note the framework — you will need it in Phase 3 to handle SSR/client boundaries correctly (for client-side) or to locate the right backend service layer (for server-side).
---
## Phase 2: Detect AI Agents
Scan the codebase for conversational AI/LLM agent functionality. Your goal is to identify AI agents that handle user conversations — not just any code that uses LLM APIs.
### Step 1: Find LLM SDK Usage
Search for imports from known AI/LLM packages using these patterns:
```
\b(openai|@openai)\b
\b(anthropic|@anthropic-ai)\b
\b(@google\/generative-ai|@google-ai)\b
\b(@mistralai)\b
\b(cohere-ai)\b
\b(@aws-sdk\/client-bedrock-runtime)\b
\b(langchain|@langchain)\b
\b(@ai-sdk|ai)\b
\b(ollama)\b
\b(llamaindex)\b
\b(@huggingface)\b
```
Also look for direct API calls without SDKs:
```
api\.openai\.com|api\.anthropic\.com|generativelanguage\.googleapis\.com
```
### Step 2: Trace to Conversation Orchestrators
For each LLM usage found, trace the import chain to find the **orchestration layer** — the service or component that:
- Manages conversation sessions and message history
- Receives user input and returns AI responses
- Coordinates the overall conversation flow
**Primary indicators (strong signals):**
- **Message handling**: `sendMessage`, `submitPrompt`, `handleUserInput`, `processResponse`, `onStreamComplete`, `generateResponse`, `chatCompletion`, `streamResponse`, `streamChat`, `generateText`
- **Conversation state**: Arrays of messages with roles (`messages.*role.*content`), `conversationHistory`, `chatHistory`, `messageList`, `chatMessages`
- **Chat UI components**: `<Chat*`, `<Conversation*`, `<Assistant*`, `<AIResponse*`, `<MessageThread*`, `<Copilot*`
**Secondary indicators (supporting signals):**
- **Streaming UI**: `isTyping`, `isStreaming`, `isGenerating`, `onToken`, `streamText`
- **Feedback**: `handleThumbsUp`, `handleThumbsDown`, `handleRetry`, `handleRegenerate`
- **RAG/context**: Vector search, embeddings, knowledge base queries
- **Token/usage tracking**: Token counting, rate limiting
### Step 3: Disambiguate
**Do NOT count as agents** — these are utilities, not agents:
- LLM client wrappers (`LLMClient`, `AIProvider`, `LangChainService`)
- Unused/legacy LLM code not imported by the conversation flow
- Prompt loaders, token counters, response parsers
- HTTP `userAgent` strings, browser agent detection, service worker agents
If `ChatService` imports `LangChainService` which wraps OpenAI, there is **one** agent (`ChatService`), not three.
**File name matching caveat:** Files named `agent.*` or containing `agent` in the path are only candidates if they also contain a primary indicator above. The word "agent" appears in many non-AI contexts (user agents, HTTP agents, service agents).
### Step 4: Find CSS Selectors (Client-Side Only)
**Skip this step if the instrumentation method is server-side.**
For each agent, identify TWO CSS selectors that Pendo needs:
1. **Input selector**: The `<textarea>`, `<input type="text">`, or `contenteditable` where users type prompts
2. **Submit selector**: The `<button>` that sends the message
Find these by locating the main chat UI component and inspecting the rendered elements.
**Selector priority (best → worst):**
1. Analytics/tracking attributes — `[data-analytics="chat-input"]`, `[data-testid="send-button"]`
2. Unique ID (if not framework-generated) — `#chat-input`
3. Semantic class (BEM-style) — `.chat-widget__input`, `.chat-widget__submit`
4. ARIA roles — `[role="textbox"][aria-label*="chat"]`
5. Stable component classes — `.ChatInput`, `.SendButton`
When a single selector isn't unique, add parent context: `.app-main .chat-container textarea`.
**Avoid unstable selectors:**
| Pattern | Source |
|---------|--------|
| `css-1a2b3c4` | Emotion |
| `Component_class__hash` | CSS Modules |
| `sc-element-6gY8Tkk` | Styled Components |
| Random hashes | CSS-in-JS |
| `:nth-child(n)` | Positional (fragile) |
### Step 5: Report and Confirm
Present your findings to the user. For each agent:
| Field | Value |
|-------|-------|
| **Agent name** | Descriptive name (e.g., "Customer Support Assistant") |
| **Main file** | Relative path to the orchestration file |
| **LLM provider** | Claude, OpenAI, Gemini, LangChain, etc. |
| **Description** | What the agent does |
| **Instrumentation method** | Client-side or Server-side |
| **Input selector** | CSS selector for the text input (client-side only) |
| **Submit selector** | CSS selector for the submit button (client-side only) |
| **Entry points** | Where prompts are submitted (file + function) |
| **Response handlers** | Where responses complete (file + function) |
| **Feedback handlers** | Where reactions are collected (file + function), if any |
If no agents are found, tell the user and stop.
**Ask the user to confirm before proceeding to Phase 3.**
---
## Phase 3: Instrument with trackAgent()
After confirmation, add tracking calls to capture user interactions. The implementation differs based on the instrumentation method determined in Phase 1.
### Idempotency Check
Before adding any code, grep the codebase for existing tracking calls:
```
trackAgentEvent
trackAgent
/data/agentic
```
If instrumentation already exists, report what's already in place and ask the user whether to skip, update, or replace it. Never double-instrument.
### trackAgent API Reference
Both client-side and server-side share the same event types and core metadata fields.
**Event types:**
| eventType | When to fire | content value |
|-----------|-------------|---------------|
| `"prompt"` | User submits a message | The user's message text |
| `"agent_response"` | Agent finishes responding | The agent's response text |
| `"user_reaction"` | User provides feedback | `"positive"`, `"negative"`, `"mixed"`, `"undo"`, or `"retry"` |
**Core metadata (required for both methods):**
| Property | Type | Required | Description |
|----------|------|----------|-------------|
| agentId | string | Yes | Unique identifier for the AI agent |
| conversationId | string | Yes | Session/conversation identifier |
| messageId | string | Yes | Unique message identifier |
| content | string | Yes | Message content or reaction type (**max 500 chars — truncate if longer**) |
**Optional metadata (both methods):**
| Property | Type | Required | Description |
|----------|------|----------|-------------|
| modelUsed | string | No | LLM model name (e.g., "gpt-4", "claude-3") |
| suggestedPrompt | boolean | No | Was this a pre-defined/suggested prompt? |
| toolsUsed | string[] | No | Tools/functions called by the agent |
| fileUploaded | boolean | No | Was a file attached to this message? |
**Additional fields for server-side only:**
| Property | Type | Required | Description |
|----------|------|----------|-------------|
| visitor_id | string | Yes | Pendo visitor ID for the user |
| account_id | string | No (recommended) | Pendo account ID |
| timestamp | number | Yes | Epoch timestamp in milliseconds |
| context.ip | string | No | Client IP address |
| context.userAgent | string | No | Client user agent string |
| context.url | string | No | Page or screen URL/path |
---
## Client-Side Instrumentation
**Use this path when the instrumentation method is client-side (web apps).**
### Client-Side Helper Function
Add this helper to the project to handle null-safety, content truncation, and error isolation. Place it in a shared utilities file (e.g., `src/utils/pendo.ts` or `src/lib/pendo.ts`):
```typescript
// --- Pendo Agent Analytics Helper (Client-Side) ---
declare global {
  interface Window {
    pendo?: {
      trackAgent: (eventType: string, metadata: Record<string, unknown>) => void;
    };
  }
}
const CONTENT_MAX_LENGTH = 500;
export function trackAgentEvent(
  eventType: "prompt" | "agent_response" | "user_reaction",
  metadata: {
    agentId: string;
    conversationId: string;
    messageId: string;
    content: string;
    modelUsed?: string;
    suggestedPrompt?: boolean;
    toolsUsed?: string[];
    fileUploaded?: boolean;
  }
): void {
  try {
    if (typeof window === "undefined" || !window.pendo?.trackAgent) return;
    window.pendo.trackAgent(eventType, {
      ...metadata,
      content:
        metadata.content.length > CONTENT_MAX_LENGTH
          ? metadata.content.slice(0, CONTENT_MAX_LENGTH)
          : metadata.content,
    });
  } catch {
    // Analytics failure must never break the product
  }
}
```
This helper:
- Guards against `window.pendo` not being loaded (async snippet) or absent (SSR)
- Handles SSR environments (`typeof window === "undefined"`)
- Truncates `content` to 500 characters to avoid payload bloat
- Catches errors silently so analytics never breaks the product
- Provides TypeScript types without requiring a separate `global.d.ts`
**Import this helper** in every file you instrument instead of calling `window.pendo.trackAgent` directly.
### SSR / Server Component Considerations
- `window` does not exist in server-side environments. The helper function handles this with the `typeof window === "undefined"` guard.
- In **Next.js App Router**: Only instrument client components (files with `"use client"`). If the agent logic lives in a server action or API route, instrument the **client-side caller** that invokes it, not the server code itself.
- In **Nuxt**: Only instrument code that runs in the browser. Use `process.client` checks or `onMounted` hooks if the helper isn't used.
- In **SSR frameworks generally**: The `trackAgentEvent` helper is safe to import anywhere — it no-ops on the server. But only **call** it from client-side code paths.
---
## Server-Side Instrumentation
**Use this path when the instrumentation method is server-side (mobile apps, browser extensions, non-web environments).**
### Server-Side Configuration
Before adding the helper, the project needs two configuration values. Search the codebase for an existing configuration/environment pattern (e.g., `.env`, `config.ts`, `constants.ts`) and add:
1. **Track Event Shared Secret** — The `x-pendo-integration-key` value. Store it as an environment variable (e.g., `PENDO_SHARED_SECRET`). Tell the user to retrieve it from **Settings > Subscription Settings > Applications > [their app] > Track Event shared secret**.
2. **Regional API Endpoint** — The base URL for the Conversations API. Store it as an environment variable or config constant (e.g., `PENDO_DATA_ENDPOINT`). The correct endpoint depends on the Pendo data region:
| Region | Endpoint |
|--------|----------|
| US | `https://data.pendo.io/data/agentic` |
| EU | `https://data.eu.pendo.io/data/agentic` |
| US1 | `https://us1.data.pendo.io/data/agentic` |
| JPN | `https://data.jpn.pendo.io/data/agentic` |
| AU | `https://data.au.pendo.io/data/agentic` |
If the user's region isn't clear, default to US and add a comment telling the user to update it.
### Server-Side Helper Function
Add this helper to a shared backend utilities file (e.g., `src/utils/pendo.ts`, `src/services/pendo.ts`, `lib/pendo.dart`, etc.). Adapt the language to match the project:
**TypeScript/JavaScript (Node.js, React Native backend, etc.):**
```typescript
// --- Pendo Agent Analytics Helper (Server-Side) ---
const CONTENT_MAX_LENGTH = 500;
const PENDO_ENDPOINT = process.env.PENDO_DATA_ENDPOINT || "https://data.pendo.io/data/agentic";
const PENDO_SHARED_SECRET = process.env.PENDO_SHARED_SECRET || "";
interface TrackAgentEventOptions {
  agentId: string;
  conversationId: string;
  messageId: string;
  content: string;
  visitorId: string;
  accountId?: string;
  modelUsed?: string;
  suggestedPrompt?: boolean;
  toolsUsed?: string[];
  fileUploaded?: boolean;
  context?: {
    ip?: string;
    userAgent?: string;
    url?: string;
  };
}
export async function trackAgentEvent(
  eventType: "prompt" | "agent_response" | "user_reaction",
  metadata: TrackAgentEventOptions
): Promise<void> {
  try {
    if (!PENDO_SHARED_SECRET) return;
    const body = {
      type: eventType,
      visitor_id: metadata.visitorId,
      ...(metadata.accountId && { account_id: metadata.accountId }),
      timestamp: Date.now(),
      props: {
        agentId: metadata.agentId,
        conversationId: metadata.conversationId,
        messageId: metadata.messageId,
        content:
          metadata.content.length > CONTENT_MAX_LENGTH
            ? metadata.content.slice(0, CONTENT_MAX_LENGTH)
            : metadata.content,
        ...(metadata.modelUsed && { modelUsed: metadata.modelUsed }),
        ...(metadata.suggestedPrompt !== undefined && { suggestedPrompt: metadata.suggestedPrompt }),
        ...(metadata.toolsUsed && { toolsUsed: metadata.toolsUsed }),
        ...(metadata.fileUploaded !== undefined && { fileUploaded: metadata.fileUploaded }),
      },
      ...(metadata.context && { context: metadata.context }),
    };
    await fetch(PENDO_ENDPOINT, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-pendo-integration-key": PENDO_SHARED_SECRET,
      },
      body: JSON.stringify(body),
    });
  } catch {
    // Analytics failure must never break the product
  }
}
```
**Python (Django, Flask, FastAPI):**
```python
# --- Pendo Agent Analytics Helper (Server-Side) ---
import os
import time
import requests
from typing import Optional
CONTENT_MAX_LENGTH = 500
PENDO_ENDPOINT = os.getenv("PENDO_DATA_ENDPOINT", "https://data.pendo.io/data/agentic")
PENDO_SHARED_SECRET = os.getenv("PENDO_SHARED_SECRET", "")
def track_agent_event(
    event_type: str,  # "prompt" | "agent_response" | "user_reaction"
    *,
    agent_id: str,
    conversation_id: str,
    message_id: str,
    content: str,
    visitor_id: str,
    account_id: Optional[str] = None,
    model_used: Optional[str] = None,
    suggested_prompt: Optional[bool] = None,
    tools_used: Optional[list[str]] = None,
    file_uploaded: Optional[bool] = None,
    context: Optional[dict] = None,
) -> None:
    try:
        if not PENDO_SHARED_SECRET:
            return
        props = {
            "agentId": agent_id,
            "conversationId": conversation_id,
            "messageId": message_id,
            "content": content[:CONTENT_MAX_LENGTH],
        }
        if model_used is not None:
            props["modelUsed"] = model_used
        if suggested_prompt is not None:
            props["suggestedPrompt"] = suggested_prompt
        if tools_used is not None:
            props["toolsUsed"] = tools_used
        if file_uploaded is not None:
            props["fileUploaded"] = file_uploaded
        body = {
            "type": event_type,
            "visitor_id": visitor_id,
            "timestamp": int(time.time() * 1000),
            "props": props,
        }
        if account_id:
            body["account_id"] = account_id
        if context:
            body["context"] = context
        requests.post(
            PENDO_ENDPOINT,
            json=body,
            headers={
                "Content-Type": "application/json",
                "x-pendo-integration-key": PENDO_SHARED_SECRET,
            },
            timeout=5,
        )
    except Exception:
        # Analytics failure must never break the product
        pass
```
**For other languages** (Swift, Kotlin, Dart, Go, etc.), adapt the same pattern: fire-and-forget HTTP POST with error isolation, content truncation, and the same JSON body structure.
The server-side helper:
- Reads endpoint and shared secret from environment/config
- Sends a `POST` to the regional `/data/agentic` endpoint with the correct headers
- Builds the JSON payload with top-level `type`, `visitor_id`, `timestamp`, and nested `props`
- Only includes optional fields when they have a value (no `null` or `undefined` in the payload)
- Truncates `content` to 500 characters
- Catches errors silently so analytics never breaks the product
- Is fire-and-forget — does not await the response in the critical path (if the language/framework supports background tasks, use them)
**Import this helper** in every file you instrument instead of constructing the HTTP request directly.
### Resolving `visitor_id` and `account_id`
The server-side API requires `visitor_id` — a string that matches the Pendo visitor ID for the current user. Search the codebase for how the app identifies users:
1. **Look for existing Pendo initialization** — if the app already uses `pendo.initialize()` or a Pendo SDK elsewhere, find the `visitor.id` value being passed. Use the same identifier.
2. **Look for auth/session context** — find where the app stores the current user's ID (`req.user.id`, `currentUser.id`, `session.userId`, auth tokens, etc.). This is typically what maps to `visitor_id`.
3. **For `account_id`** — look for organization/team/company identifiers in the same auth context (`user.orgId`, `user.accountId`, `tenant.id`). This is optional but recommended.
Tell the user which identifier you mapped to `visitor_id` and `account_id`, and ask them to confirm it matches what they use in Pendo.
### Resolving `context` Fields (Server-Side Only)
For server-side instrumentation, the `context` object provides Pendo with client environment information it can't derive from the request itself (since the request comes from your backend, not the user's device). Search for how the app captures client context:
1. **`context.ip`** — Look for request IP extraction (`req.ip`, `request.remote_addr`, `X-Forwarded-For` header parsing). If the agent's backend handler receives the original client request, extract the IP from there.
2. **`context.userAgent`** — Look for user agent headers (`req.headers["user-agent"]`). For mobile apps, this is often a custom string identifying the app and device.
3. **`context.url`** — For mobile apps, use the screen/route name (e.g., `myapp://chat`, `/screens/assistant`). For web-like contexts, use the page URL.
If these values aren't readily available in the code path where tracking happens, **omit the `context` object entirely** rather than passing empty strings. Pendo will attempt to derive what it can from the server request.
---
## Shared: Resolve Optional Metadata from the Codebase
**This section applies to both client-side and server-side instrumentation.**
Before writing any `trackAgentEvent` calls, trace each optional metadata field to its actual value in the codebase. Do **not** hardcode these — detect them dynamically for each agent.
### `modelUsed` — Trace the model string
The model name was already identified in Phase 2 (LLM provider). Now find the **exact string** the code passes to the SDK:
1. Search the agent's LLM call site for the `model` parameter (e.g., `model: "gpt-4o-mini"`, `model: "claude-3-5-sonnet..."`)
2. If the model is set via environment variable or config (e.g., `process.env.OPENAI_MODEL`, `config.model`), reference **that variable** in the tracking call — not a hardcoded string. Example: `modelUsed: config.model ?? "unknown"`
3. If the SDK response object includes the model (e.g., `response.model`, `completion.model`), prefer that — it reflects what was actually used, including fallbacks
4. If the model string is truly not determinable at runtime, use the provider name as a fallback (e.g., `"openai"`, `"anthropic"`) rather than guessing a specific model
### `suggestedPrompt` — Detect prompt chip/button UIs
Search the chat UI component and its children for suggested prompt patterns:
1. Look for arrays of predefined prompts: `suggestions`, `promptChips`, `starterPrompts`, `quickActions`, `exampleQuestions`, `samplePrompts`
2. Look for UI elements that render clickable prompt options: chips, pills, buttons with prompt text, "Try asking..." sections
3. If found, trace the click handler — it typically calls the same submit function as manual input. Add a `suggestedPrompt: true` flag to the tracking call at that call site, and `suggestedPrompt: false` at the manual submit path
4. If the codebase has no suggested prompt UI, **omit the field entirely** rather than hardcoding `false` — absence is cleaner than a meaningless default
### `fileUploaded` — Detect file/attachment inputs
Search the chat UI for file upload capability:
1. Look for `<input type="file">`, drag-and-drop zones (`onDrop`, `onDragOver`), or file upload libraries (`react-dropzone`, `multer` references on the client). For mobile apps, look for image pickers, document pickers, or camera capture (`launchImageLibrary`, `DocumentPicker`, `UIImagePickerController`, `Intent.ACTION_GET_CONTENT`)
2. Look for state variables tracking attachments: `file`, `attachment`, `uploadedFile`, `files`, `attachments`, `selectedFile`
3. If found, reference the file state in the tracking call: `fileUploaded: attachments.length > 0` or `fileUploaded: file !== null`
4. If the chat UI has no file upload capability, **omit the field entirely** — do not hardcode `false`
### `toolsUsed` — Extract from the response object
Search for tool/function calling patterns in the agent's response handling:
1. Check if the SDK response exposes tool calls: `response.tool_calls`, `result.functionCalls`, `completion.choices[0].message.tool_calls`, `response.content.filter(b => b.type === "tool_use")`
2. If found, extract the tool names: `toolsUsed: toolCalls.map(t => t.function?.name ?? t.name)`
3. If the agent uses LangChain or similar frameworks, look for agent executor step outputs that list tool invocations
4. If the agent does not use tool/function calling, **omit the field entirely**
**Summary: For every optional field, either wire it to a real value from the codebase or omit it. Never hardcode a placeholder value.**
---
## Shared: conversationId Lifecycle
The `conversationId` must be **generated once per conversation session** and **reused for every message in that session**. Do not generate a new ID per message.
**Where to store it depends on the codebase:**
- If the app already has a conversation/session ID in state or URL params, use that
- If not, generate one with `crypto.randomUUID()` (or the language equivalent) when the conversation starts (component mount, first message, or new chat action) and store it in component state, context, or a module-level variable
- For server-side: the conversation ID may come from the client request or be managed in a database. Look for existing session/thread IDs in the API layer.
**When to reset it:**
- User starts a "New Chat" / "New Conversation"
- User navigates to a different conversation
- Page/session is refreshed (if conversations aren't persistent)
## Shared: messageId Generation
Each message needs a unique ID:
- **Preferred**: Use an existing message ID if the app already assigns one
- **Fallback**: `crypto.randomUUID()` (or `uuid.uuid4()`, `UUID.randomUUID()`, etc.)
- **Last resort**: `` `${eventType}_${Date.now()}` ``
Important:
- Generate the `messageId` **before** sending the message
- For `"user_reaction"` events, use the **original message's ID**, not a new one
---
## Implementation Steps
For each detected agent, apply the appropriate pattern based on the instrumentation method.
### Client-Side Implementation
**1. Add prompt tracking** — Find the submit/send handler and add tracking right before the API call:
```typescript
import { trackAgentEvent } from "@/utils/pendo";
// Inside the submit handler:
const messageId = crypto.randomUUID();
trackAgentEvent("prompt", {
  agentId: "AGENT_ID",
  conversationId,
  messageId,
  content: userInput,
  // Only include optional fields if detected in the codebase:
  // suggestedPrompt: isSuggested,    ← from prompt chip click handler
  // fileUploaded: files.length > 0,  ← from file upload state
});
```
**2. Add response tracking** — Find where the response completes (not starts) and add tracking:
```typescript
// For streaming: inside onEnd / onComplete / onFinish
// For non-streaming: after the await resolves
trackAgentEvent("agent_response", {
  agentId: "AGENT_ID",
  conversationId,
  messageId: response.id || crypto.randomUUID(),
  content: fullResponseText,
  // Only include optional fields if detected in the codebase:
  // modelUsed: response.model ?? config.model,  ← from SDK response or config
  // toolsUsed: toolCalls.map(t => t.name),       ← from response tool_calls
});
```
**3. Add reaction tracking** (if feedback UI exists):
```typescript
// Inside thumbs up/down, retry, regenerate handlers
trackAgentEvent("user_reaction", {
  agentId: "AGENT_ID",
  conversationId,
  messageId: originalMessage.id, // the message being reacted to
  content: "positive", // or "negative", "retry", "undo", "mixed"
});
```
### Server-Side Implementation
**1. Add prompt tracking** — Find the backend handler that receives user messages (API route, controller method, or service function) and add tracking:
```typescript
import { trackAgentEvent } from "@/utils/pendo";
// Inside the message handler:
await trackAgentEvent("prompt", {
  agentId: "AGENT_ID",
  conversationId: req.body.conversationId, // or however the app tracks sessions
  messageId: req.body.messageId || crypto.randomUUID(),
  content: req.body.message,
  visitorId: req.user.id,        // ← from auth/session context
  accountId: req.user.orgId,     // ← from auth/session context, if available
  // Only include optional fields if detected in the codebase:
  // suggestedPrompt: req.body.isSuggested,
  // fileUploaded: req.body.attachments?.length > 0,
  context: {
    ip: req.ip,
    userAgent: req.headers["user-agent"],
    // url: req.body.screenName or req.originalUrl
  },
});
```
**2. Add response tracking** — Find where the LLM response completes and add tracking:
```typescript
// After the LLM call resolves (streaming: onEnd; non-streaming: after await)
await trackAgentEvent("agent_response", {
  agentId: "AGENT_ID",
  conversationId,
  messageId: response.id || crypto.randomUUID(),
  content: fullResponseText,
  visitorId: userId,
  accountId: accountId,
  // Only include optional fields if detected in the codebase:
  // modelUsed: response.model ?? config.model,
  // toolsUsed: toolCalls.map(t => t.name),
  context: {
    ip: clientIp,
    userAgent: clientUserAgent,
  },
});
```
**3. Add reaction tracking** (if feedback API exists):
```typescript
// Inside the feedback/reaction endpoint handler
await trackAgentEvent("user_reaction", {
  agentId: "AGENT_ID",
  conversationId: req.body.conversationId,
  messageId: req.body.messageId, // the message being reacted to
  content: req.body.reaction,    // "positive", "negative", "retry", "undo", "mixed"
  visitorId: req.user.id,
  accountId: req.user.orgId,
});
```
**Important for server-side:** The tracking calls are `async` but should be fire-and-forget in the critical path. Do not let a tracking failure block the user's response. Use `void trackAgentEvent(...)` or a background task queue if the framework supports it.
---
## Verification
After instrumentation, verify your work:
1. **Grep for all tracking calls** (including via the helper):
   ```
   trackAgentEvent
   ```
2. **Confirm coverage** — check that you have:
   - At least one `"prompt"` call per agent
   - At least one `"agent_response"` call per agent
   - `"user_reaction"` calls if feedback UI exists (not required if there's no feedback mechanism)
3. **Confirm no duplicates** — each event should be tracked in exactly one place
4. **Confirm conversationId reuse** — the same `conversationId` variable is used across prompt, response, and reaction calls within a conversation
5. **Confirm optional metadata is dynamic** — check that:
   - `modelUsed` references a variable, config value, or response field — not a hardcoded model name
   - `suggestedPrompt` is only present where a suggested prompt UI exists, and is wired to the actual click path
   - `fileUploaded` is only present where file upload exists, and references actual file state
   - `toolsUsed` extracts from the SDK response, not a hardcoded array
   - Any optional field not applicable to this agent is **omitted**, not set to a default
6. **Confirm server-side specifics** (if applicable):
   - `visitorId` is wired to the app's user identifier, matching what Pendo uses
   - `accountId` is wired to the org/team identifier if available
   - Environment variables for `PENDO_SHARED_SECRET` and `PENDO_DATA_ENDPOINT` are documented
   - The regional endpoint matches the user's Pendo data region
   - `context` fields are populated from the original client request where available
Present a summary table:
| Agent | Event Type | File | Function/Handler | Agent ID | Method |
|-------|-----------|------|-------------------|----------|--------|
| Customer Support | prompt | `src/components/Chat.tsx` | `handleSubmit` | `abc123` | client |
| Customer Support | agent_response | `src/components/Chat.tsx` | `onStreamEnd` | `abc123` | client |
| Customer Support | user_reaction | `src/components/Chat.tsx` | `handleThumbsUp` | `abc123` | client |
| Mobile Assistant | prompt | `src/api/chat.ts` | `handleMessage` | `def456` | server |
| Mobile Assistant | agent_response | `src/api/chat.ts` | `onComplete` | `def456` | server |
If any agent IDs are placeholders (`"YOUR_AGENT_ID"`), remind the user to replace them with real IDs from the Pendo UI.
For server-side agents, also remind the user to:
1. Set the `PENDO_SHARED_SECRET` environment variable with the track event shared secret from **Settings > Subscription Settings > Applications**
2. Set the `PENDO_DATA_ENDPOINT` to the correct regional endpoint if not using the US default
3. Confirm the `visitor_id` mapping matches their Pendo visitor configuration
