---
name: create-orchestrate-journey
description: >
  Creates and configures draft Orchestrate email journeys in Pendo via MCP. Trigger when the user asks to
  create an Orchestrate journey, build a multi-email journey, set up a welcome email campaign, design an
  onboarding email sequence, or add a conditional split after an email. Also trigger when the user wants
  email HTML or copy saved on journey email steps. Also trigger for phrases like
  "multi-email journey", "retention journey", "re-engage inactive users", "win back users",
  "re-engagement emails", or "lifecycle email flow" — even if the request is casual
  or incomplete. Always use this skill proactively when Orchestrate journey creation is requested.
---

# Create Orchestrate Journey

You help customers create **draft Orchestrate email journeys** in Pendo using MCP tools. A journey is a
sequence of email steps (and optional conditional splits) that visitors move through over time.

**Email steps only.** MCP can create journeys with **email** message steps. **In-app guide** steps in
journeys are not supported via MCP yet — coming soon. If the user asks for a guide step, explain that
limitation and offer an email-only journey for now.

**Critical rule: Never call a create tool on the first message.** Even if the user gives a detailed
one-shot prompt, run through intake first. Skipping intake leads to wrong apps, guessed segment ids, and
invalid journey graphs.

---

## Where rules live (read tools first)

| Need | Authoritative source |
|------|---------------------|
| Graph shape, node fields, wait days, conditional splits | **`templateJson` JSON Schema** on `createOrchestrateJourneyFromJson` |
| Segment, schedule, goal updates | Each tool's **description + `WithWorkflow`** (`setOrchestrateJourneySegment`, `setOrchestrateJourneySchedule`, `setOrchestrateJourneyGoal`) |
| Email HTML validation | **`htmlContent` JSON Schema** on `updateOrchestrateEmailContent` + tool error messages |
| Topology sketches (non-obvious layouts) | `references/journey-graph-patterns.md` |
| Email copy craft (tone, iteration) | `references/email-content.md` |

This skill covers **customer intake and step order**. Do not duplicate MCP parameter rules here — read the
tool schema when you call it.

---

## Terminology (match Orchestrate UI)

| Say | MCP / wire |
|-----|------------|
| **conditional split** | `nodeType` `"Condition"` — no wait; evaluated immediately when the visitor arrives |
| **Yes branch** / **No branch** | `edgeType` `"Yes"` / `"No"` from the split |
| **condition of the conditional split** | UI only after create — not in `templateJson` |
| **built-in template id** | `mcpSingleEmailJourneyTemplate` → `createOrchestrateJourneyFromTemplate` |
| **email step id** | `messageId` from `getOrchestrateJourneySteps` → `emailId` on `updateOrchestrateEmailContent` |
| **reach inactive visitors** | `reachInactiveVisitors` on `setOrchestrateJourneySegment` |
| **retain visitors after entry** | `retainVisitorsAfterEntry` on `setOrchestrateJourneySegment` |

---

## Step 0 — Pendo MCP connectivity

Orchestrate journey tools require the Pendo MCP server.

If MCP tools are unavailable or authentication fails, tell the user how to connect Pendo MCP:
https://support.pendo.io/hc/en-us/articles/41102236924955-Connect-to-the-Pendo-MCP-server

Resolve **subscription** and **application** before creating:
- If the user names them, use those values.
- Otherwise call `list_all_applications` and help them pick the right subscription and app.

---

## Step 1 — Progressive intake

**Retention vs analytics:** If the user asks about retention **metrics** (churn rate, retention curve,
"how many users come back") — that is **not** this skill. Use analytics MCP tools (e.g.
`cohortRetentionCurve`) or the pendo-analytics plugin. This skill is for **building** a draft Orchestrate
**email journey** (e.g. re-engage inactive visitors).

Work through **two rounds**. Ask each round together, wait for answers, then continue. Do not collapse
rounds into one message.

Acknowledge what the user already shared, then ask what is still missing.

### Round 1 — Required

1. **Journey name** — Shown in the Orchestrate UI (e.g. "Q1 onboarding").
2. **Application** — Which Pendo application this journey belongs to.
3. **What to build** — In plain language: how many emails, wait days between them, and any branching.
   You infer the create tool from this — do not ask the user to pick a technical template category.
   For common shapes (welcome, re-engagement / retention emails, branched follow-ups), see the use-case
   table in `references/journey-graph-patterns.md`.

### Round 2 — Optional (ask every time; user may skip)

4. **Audience** — Segment name or rules. Can find, create, or attach after the journey exists (Step 4).
   For **re-engagement / retention** journeys, ask how they define inactive visitors (e.g. no login in
   30 days) — then `segmentList`, `buildPendoSegment`, or `createSegment`.
5. **Schedule** — Start date/time if they care. End date and send windows are UI-only — say so if asked.
6. **Conversion goal** — Page, feature, or track event (if any).
7. **Email copy** — Subject and/or body (optional now; Step 5 after create).

After Round 2 (or skips), proceed to Step 2.

---

## Step 2 — Choose the create tool

| User need | Tool |
|-----------|------|
| Exactly one welcome email (built-in graph) | `createOrchestrateJourneyFromTemplate` with `templateId` `mcpSingleEmailJourneyTemplate` |
| Multiple emails, wait days, or a conditional split | `createOrchestrateJourneyFromJson` with `templateJson` |

**Do not** use ids from `listOrchestrateJourneyTemplates` as create inputs.

For custom graphs: read the **`templateJson` JSON Schema**, then `references/journey-graph-patterns.md` if
the layout is not obvious. Field names, enums, and validation live in the schema — not in this skill.

---

## Step 3 — Create and verify

1. Call the chosen create tool with `subId`, `appId`, and `name`.
2. Share `journeyUrl` from the response.
3. Call `getOrchestrateJourneySteps` and confirm step names and wait days match what the user asked for.
4. On validation failure, read the tool error and JSON Schema; retry once before asking the user for help.

---

## Step 4 — Configure audience, schedule, and goal (optional)

Only run steps the user asked for in intake (or explicitly confirms now).

**Follow each tool's workflow string** in its MCP description. This skill adds only what MCP does not cover.

### Audience — `setOrchestrateJourneySegment`

- Follow the tool workflow (`segmentList` → disambiguate → set).
- If no segment exists, offer `buildPendoSegment` or `createSegment`, then set with the new id.

**Phrase → flag** (omit flags unless the user asked; tool preserves existing values when omitted):

| User might say | Parameter |
|----------------|-----------|
| "include inactive visitors", "new signups without activity yet" | `reachInactiveVisitors: true` |
| "entry segment only", "don't kick people out if they leave the segment" | `retainVisitorsAfterEntry: true` |
| "must stay in the segment", "drop if they leave the segment" | `retainVisitorsAfterEntry: false` |

### Schedule — `setOrchestrateJourneySchedule`

- Follow the tool description for `startDate` formats and constraints.
- **Plugin-only tip:** when starting soon, prefer a time on the hour (`:00`) and at least two hours out —
  reduces friction when they activate in the UI (not enforced by MCP).

### Goal — `setOrchestrateJourneyGoal`

- Follow the tool workflow (`listCountables` / `searchEntities` → disambiguate → set).

---

## Step 5 — Email HTML content (optional)

When the user wants to **write, draft, add, or update** email body HTML (including copy from intake).

1. `getOrchestrateJourneySteps` → each `messageType` `Email` step (`name` + `messageId`).
2. Read **`htmlContent` JSON Schema** on `updateOrchestrateEmailContent` (validation rules).
3. Read `references/email-content.md` for **copy craft** and the **minimal marketing HTML worked example**.
4. Call `updateOrchestrateEmailContent` per approved email (`emailId` = `messageId`).
5. Remind the user: **subject**, **from**, and **reply-to** are UI-only.

---

## Step 6 — What the user finishes in Orchestrate UI

Tell the user clearly what MCP did **not** set:

- **Email subject line**, **from / reply-to**, and **sending identity** (even when HTML body was saved via MCP).
- **Condition of the conditional split** (when the graph includes a conditional split).
- **End date**, delivery time windows, and advanced schedule constraints.
- **Email HTML body** — only if the user skipped Step 5.
- **Activate** the journey last — after audience, schedule, content, and split conditions are ready.

Point them to `journeyUrl` for all of the above.

---

## Step 7 — Assistant message guidelines

- Be friendly and concise — explain choices briefly.
- Share the journey link prominently after create.
- After configuration, summarize what was set vs what remains in the UI.
- Do not say "Here's the journey:" — describe what you built and why.
- Invite changes: segment, timing, extra emails, conditional split structure, or email copy.

---

## Common mistakes — agent behavior (not API rules)

- Creating on the first message without intake.
- Using `listOrchestrateJourneyTemplates` ids as create inputs.
- Ignoring a tool's `WithWorkflow` (guessing ids, auto-picking first segment/goal match).
- Using `journeyId` or `stepId` as `emailId` (use `messageId`).
- Expecting to edit the graph via MCP after create.
- Promising activation, subject line, or sender setup via MCP.
- Trying to add in-app guide steps via MCP (not supported yet).
- Expecting `getOrchestrateEmail` to return saved HTML.

API validation errors (unsubscribe, personalization, graph shape) come from the tool — read the error and schema.

---

## Reference files

- `references/journey-graph-patterns.md` — Topology sketches only; pair with `templateJson` JSON Schema.
- `references/email-content.md` — Copy craft and workflow; pair with `htmlContent` JSON Schema.
