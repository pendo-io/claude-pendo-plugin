# Journey graph patterns

**Authoritative:** `templateJson` JSON Schema on `createOrchestrateJourneyFromJson` (fields, enums, wait
days, conditional split rules).

**This file:** topology only — how steps connect. Adjust names, ids, and wait days to match the user.
Audience, segments, schedule, and email copy are handled in `SKILL.md` and MCP tools — not here.

---

## Use case → pattern

| User intent (plain language) | Pattern | Create tool |
|------------------------------|---------|-------------|
| Single welcome email | Built-in graph | `createOrchestrateJourneyFromTemplate` (`mcpSingleEmailJourneyTemplate`) |
| Welcome or onboarding sequence (2 emails) | Pattern 1 | `createOrchestrateJourneyFromJson` |
| Re-engage inactive users, retention emails, win-back | Pattern 2 | `createOrchestrateJourneyFromJson` |
| Different follow-ups by behavior (after intro email) | Pattern 3 | `createOrchestrateJourneyFromJson` |

---

## Pattern 1 — Two-email journey (welcome / onboarding)

```
Start -> [Welcome email] -> [Getting started tips] -> Exit
         wait: 3 days        wait: 1 day
```

Linear sequence. Wait before the gap goes on the **first** email (see schema for `durationInDays`).

---

## Pattern 2 — Three-email re-engagement / retention journey

Default topology when the user asks for a **retention journey**, **re-engagement**, or **win-back** emails
for inactive visitors.

```
Start -> [We miss you] -> [Here's what's new] -> [Last chance] -> Exit
         wait: 2 days      wait: 5 days         wait: 1 day
```

Linear sequence. Tune wait days and step names to match the user's request.

---

## Pattern 3 — Conditional split

Use when one intro email should branch to different follow-ups (e.g. engaged vs not engaged — condition
set in UI after create).

```
Start -> [Introduction] -> (conditional split) --Yes--> [Power-user tips] -> Exit
         wait: 1 day              |  (immediate)
                                  --No--> [Getting started guide] -> Exit
```

Split has **no wait**. Condition of the split is configured in the Orchestrate UI after create.
