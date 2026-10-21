# Journey checkup

Answers "is this journey set up and behaving the way it was meant to?". Use it when the user asks for a health
check, before activation or for a live journey.

## Reads (budget: about 15 calls)

1. `getOrchestrateJourney`: purpose (name, description), goal, entry segment, schedule start and end, the two
   segment flags, `ignoreThrottling`.
2. `getOrchestrateJourneySteps`: every step, edge, wait and `messageType`.
3. For a live journey: `getOrchestrateJourneyMetrics` (no date range, for the cumulative goal) and
   `getOrchestrateJourneyStepMetrics` over the journey's live dates.
4. `getSegment` on the entry segment, for its rule text.
5. `entityUsage` on the goal item over the live dates, with no visitor filter.
6. `getOrchestrateEmail` for each email step, only when a check below needs its name or settings.

## Checks

Report each check as **Pass**, **Warning** or **Needs a fix**, with the evidence in one line.

| # | Check | Warning or needs a fix when |
| --- | --- | --- |
| 1 | **Purpose and schedule match** | A date in the journey, email or guide name, subject or text (an event, a deadline) has passed while the journey still enters or sends people. **Count** entries and sends since that date, and compare unsubscribe rates before and after it (contract: *After an event or deadline date*) |
| 2 | **Every path sends something** | A path from start reaches an exit without a message step. Name the path |
| 3 | **Waits fit the schedule** | People can't reach a message before the schedule ends, or are stuck on a step before it. **Count** them (contract: *People who can't reach a step before the journey ends*) |
| 4 | **Goal has recorded events and is specific** | No events were recorded for the goal in the window, or most entrants already do it (see the contract) |
| 5 | **Goal required or optional** | Required goal plus a goal that is the email's own call to action: achievers are removed from later messages. Say whether that was intended |
| 6 | **Audience settings** | `retainVisitorsAfterEntry=false` with an entry rule people leave by acting ("has not done X"). Say it as: people leave the journey between steps when they stop matching the audience. `reachInactiveVisitors=false` with a "no recent visit" rule: the two conflict. Name the UI settings, never these field names |
| 7 | **Exclusions work** | The audience rule joins two "is not" conditions with OR, so it excludes nobody |
| 8 | **Audience quality** | Bounce rate 2% or more and no active-status or recent-activity rule |
| 9 | **Guide steps shown** | A live guide step has 0 views. Its cause can't be read: don't guess it from the guide's type or settings. Name what to check in the guide's settings |
| 10 | **Opens and clicks recorded** | Opens under 5% of delivered while clicks are normal (opens not recorded), or clicks above opens (scanner signature) |
| 11 | **Unsubscribes trending** | Unsubscribe rate rising across weekly windows. Give the rates |
| 12 | **Throttling** | Say whether the journey ignores throttling. The subscription setting can't be read |
| 13 | **Everyone entered is sent something** | Entered minus sent on the first email step. If the gap is large, run the reach diagnosis for it, or offer to |

## Always say plainly

- Conditional split rules can't be read; only their labels. Don't infer them from the names, and don't recommend
  rewiring. Say where people actually went.
- There is no change history and no author. "Who changed it" has no answer in the data.
- `visitorsCompleted` is not goal attainment.

## Output

Lead with a one-line verdict:
- *Not doing its job*: any check from 1, 2, 3, 9 or 13 needs a fix. If it is still sending after its event or
  deadline date, say **pause it now**, first.
- *Working, with N warnings*: only warnings.
- *Set up as intended*: everything passes.

Then the counts that matter most in one line each: entered, sent, never sent, and anything since the event date.

Then the check table, items that need a fix first. For each warning or needed fix, draft the fix and offer to hand it to
`orchestrate-journeys`.
