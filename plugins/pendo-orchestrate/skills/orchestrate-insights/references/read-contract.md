# Orchestrate read contract

What the Orchestrate read tools return, what each number means, where lists stop, and which methods work.
Read this before the first Orchestrate read call. Parameter rules stay on the tools themselves.

Checked against the Pendo backend and the live MCP connector on 2026-10-19. If a tool's description now says
something different, the description wins. Say so in your answer.

---

## Tool cards

| Tool | Scope | Row cap and truncation signal | Notes |
| --- | --- | --- | --- |
| `listOrchestrateJourneys` | All journeys, optional `status` | `limit` ≤ 500 with `offset`. **No total**: page until a short page | No name filter. Match by name, then by `appId` |
| `getOrchestrateJourney` | One journey's configuration | — | Goal, entry segment, schedule, `ignoreThrottling`, `reachInactiveVisitors`, `retainVisitorsAfterEntry`, `controlGroupSize`. No step graph, no email content, no split rules |
| `getOrchestrateJourneySteps` | Step graph | — | Steps, edges, `messageId`, `messageType`, `durationInDays`. Condition steps return a label only, not the rule |
| `getOrchestrateJourneyMetrics` | Entered, completed, goal achievers | — | With a date range, `visitorsAchievedGoal` and `goalAttainmentRate` are **dropped**. Goal numbers are cumulative only |
| `getOrchestrateJourneyStepMetrics` | Per message step, people who **entered within the date range** | — | Counts are unique people per step, never emails |
| `getOrchestrateJourneyStepVisitors` | Per person, one step, entrants in the range, events in the range | ≤ 5,000 rows. `summary.numVisitors` is the full count; `summary.truncated` | Rows sorted by visitor ID. `eventType` decides who appears |
| `getOrchestrateJourneyVisitors` | Who is in the journey **now** (point in time, `meta.evaluatedAt`) | ≤ 5,000 rows. `summary.numVisitors` is uncapped; `summary.truncated` | Default filter is `active`: use `all` for everyone who ever entered. Each row has `enteredAt` and `stepVisits[].enteredAt`. `meta.enrollmentAvailable=false` means no list yet |
| `getOrchestrateVisitorJourneyState` | One person, all active or paused journeys | ≤ 5,000 | **Omits journeys where the person hit the entry throttle.** Ended journeys may be missing |
| `getOrchestrateVisitorEmails` | One person, emails in the range | ≤ 5,000. `summary.truncated` | One person per call |
| `listOrchestrateEmails` | Standalone **and** journey emails | `limit` ≤ 500 with `offset`. **No total** | Has a `name` filter. Survey emails are excluded |
| `getOrchestrateEmail` | One email's configuration | — | `attributes.journeyId` is set when the email belongs to a journey |
| `getOrchestrateEmailMetrics` | One email, unique people, all its sends in the range | — | `metricsScope` and `segmentPipeline` (see Methods) |
| `getOrchestrateEmailVisitors` | Per person, one email | `limit` ≤ 5,000, but rows have been seen to stop around 500 with no cursor. **No `truncated` field**: compare `summary.numVisitors` with the row count | — |

All date ranges are inclusive, in the subscription's time zone, at most 367 days. All rates are on a 0–100
scale. Bounced means hard bounces only.

---

## What the numbers mean

**Goal.**
- A person counts as reaching the goal when their last use of the goal page, feature or track event is on
  or after the moment they entered the journey.
- There is no end date and no link to any one message.
- It is counted per person, not per account.
- It does not depend on the person still being in the entry segment. People who left the segment still count.
- The journey's Metrics tab uses a different rule (goal events inside the journey's scheduled dates, divided by
  all entrants). Name the rule next to every goal number.

**Goal achievers and later messages.**
- When the goal is **required**, people who reach it are marked completed and removed from later messages.
- When the goal is **optional**, nobody is removed.
- Check which one before you blame the goal for missing sends.

**Toggle goals.** A goal on a toggle feature counts the click whether it turned something on or off. Say
"used the toggle", never "enabled". Read the goal item with `getEntity` to see what it is.

**Completed.** `visitorsCompleted` means the person reached an exit step or aged out. It is not goal attainment.

**Entered.** People with the journey's entry tag. With a date range, it is filtered on when they were tagged,
not on event dates.

**Throttling. There are two different settings.**
1. **Journey throttle** (a limit on how many journeys a person can be active in at once):
   - It **blocks entry**. A held person gets no entry tag, so they are missing from "entered".
   - They stay eligible. They are probably checked again on later runs, but that is inferred from the code and not confirmed.
   - People already in a journey are never held.
   - Nothing records who was held.
   - `ignoreThrottling` on the journey is readable. The subscription switch and cap are **not readable**.
2. **Email throttle** (a limit on emails per person over a period of days):
   - It is applied by the email service.
   - Whether it delays or drops a send is **not documented anywhere we can read**.
   - Never name it as the cause of a missing send. You may list it as a possible cause.
3. **`ignoreThrottling=false` doesn't mean people can't get several messages.** The journey throttle limits
   how many journeys a person is active in at once, only when the subscription setting is on (unreadable), and
   it doesn't apply to standalone emails.

**Opens and clicks.**
- Opens can be recorded automatically by mail privacy features.
- Some security-scanner clicks still get through the filter. Clicks above opens on a step is a scanner
  signature.
- A guide step's "clicked" includes closing the guide.
- Conditional splits route on unfiltered clicks, so a split's counts can differ from the reads.

**Conditional splits.** The rule isn't readable. Never work out what a split does from its name, and never
recommend rewiring it. Say where people actually went, using `stepVisits` from `getOrchestrateJourneyVisitors`.

**`metricsScope`** on email metrics:
- `allActivityInPeriod` (default) counts any activity in the range, including on emails delivered earlier.
- `emailsDeliveredInPeriod` counts only people who were delivered the email in the range. In this scope sent
  equals delivered and bounces don't appear.
- So use **two calls**: take sent, delivered and bounced from the default scope, and take opened, clicked and
  their rates from `emailsDeliveredInPeriod` (that stops open and click rates going over 100%). Never report sent
  or bounced from `emailsDeliveredInPeriod`.

---

## Methods that work

**Exact engaged vs not engaged** (no sampling, no ID cap):
1. Build the outcome segment with `buildPendoSegment`: people who used the outcome item at least once between
   two fixed dates.
2. Call `getOrchestrateEmailMetrics` for the email, `metricsScope=emailsDeliveredInPeriod`, twice: once without
   `segmentPipeline`, once with the segment's `pipeline` passed **verbatim**.
3. Openers who did the outcome = opened (with segment). Non-openers who did it = delivered (with segment) minus
   opened (with segment). Divide each by its own group size from the call without the segment.
4. Repeat with a segment for the same length of time **before** the send.

Do not sample visitor IDs into `entityUsage` for this. A list cut at 500 or 5,000 and sorted by ID is not a
sample.

**Who entered, and when they reached each step.**
- `getOrchestrateJourneyVisitors` with `enrollmentFilter=all` gives an uncapped total, plus each person's
  `enteredAt` and step entry times (`stepVisits`).
- Step entry time is **not** send time. A step waits before it sends.

**Find the outcome item** (before you say anything about results). In this order:
1. Read the audience rule (`getSegment`). A "never used X" or "has not done X" clause names the outcome item:
   use that exact item.
2. The journey goal, read with `getEntity`.
3. Search every app for the purpose's words: `searchEntities` with no `appId` (Feature, Page, TrackType), or
   `listPagesFeaturesAndEvents` per app. Outcomes often live in another app (a community site, a docs site).
4. Keep the item with recorded events in the window (`entityUsage`). Check events **in the item's own app**:
   pass that app's id, never the journey's app. An item in another app shows 0 events when checked in the
   wrong app.

Never say "there's no usage data for X" until you've searched every app and checked for events.

**People with no email address** (they can't be sent anything):
- Find the email field with `visitorMetadataSchema` (often `visitor.agent.email`).
- Call `listVisitors` on the audience segment with `groupByMetadata` set to that field, `limit` 5. The
  "(none)" group is the count.
- This is today's segment, not the audience at entry. Say so. For a one-time send it's usually close.

**How long until a message is sent** (send lag):
- The configured wait is the sum of the step waits on the path.
- The observed lag compares entries a day with sends a day. Call `getOrchestrateJourneyMetrics` and
  `getOrchestrateJourneyStepMetrics` one day at a time for the launch days and the first busy week only.
- Say what share was sent the same day and what share the next day or later.
- Step entry times (`stepVisits`) are not send times.

**People who can't reach a step before the journey ends:**
- For each message step, add up the waits before it on the path (`durationInDays`).
- Anyone who entered after the schedule end minus that total can never reach it.
- Count them from `enteredAt` in `getOrchestrateJourneyVisitors` (`all`), or from entries a day near the end.
- Also count people whose last `stepVisits` entry is a step before the message, with `completed` false.
- Say the count. Don't just warn.

**After an event or deadline date** (a date in the journey or email name, subject or description):
- Count entries and sends from the day after that date until today: `getOrchestrateJourneyMetrics` and
  `getOrchestrateJourneyStepMetrics` with that range.
- Compare unsubscribe rates before and after it.

**Accounts.**
- `listVisitors` on a segment returns `numAccounts`.
- For a recipient list you have in full (5,000 or fewer), build a segment of those visitors, or group
  `listVisitors` by account. That gives the accounts reached.
- If the lists are cut, say accounts weren't counted.

**When a list stops at 5,000 rows**, split the read so each part fits, then add the parts up:
- `getOrchestrateJourneyVisitors`: call it once with `enrollmentFilter=active` and once with `completed`.
- Step and email visitor tools: split the date range into shorter windows, or use separate `eventType`s.
- Never say the rest "can't be retrieved" before you've tried splitting.

**Journey emails vs standalone emails.**
- Page `listOrchestrateEmails`.
- An email whose `getOrchestrateEmail` has `attributes.journeyId` belongs to that journey. The rest are
  standalone.
- An overview must include both.

**Does the outcome have recorded events?**
- Before measuring an outcome, call `entityUsage` on the item for the whole window, with no visitor filter.
- No events means the tag records nothing: every rate on it is zero by construction.

**Is the goal too common to mean anything?** Measure it on the journey's own entrants before the send, not on
today's segment. If 25% or more already did it, say the goal is weak. At 50% or more, say it tells you nothing.

**Test and operational sends.**
- Treat a send as a likely test when it went to five people or fewer.
- Treat a name with "test", "internal", "incident" or "outage" as a likely test or operational send.
- Both are heuristics: label them and keep them out of campaign totals.

---

## Known read defects (as of 2026-10-19)

Internal reference: never quote these ticket keys in an answer. Tell the user what the number can't show.

| Defect | Ticket |
| --- | --- |
| A goal number is dropped when a date range is given | APP-169232 |
| `visitorsCompleted` is unreliable on some subscriptions | APP-169234 |
| Opens can appear with 0 delivered in post-send windows | APP-169235 |
| List tools have no total or "more" flag | APP-169236 |
| Email visitor rows stop around 500 with no cursor | APP-169237 |
| Audiences that match nobody look valid | APP-169239 |
