# Reach diagnosis

Answers "why didn't more people get it, who are they, and how long until they get the next message?".

## Premise first

Check the claim before diagnosing it. If someone says an email "isn't sending", read step metrics first. When
sends exist, say how many and since when, then answer the real question.

## Reads (budget: about 15 calls)

1. `getOrchestrateJourney` and `getOrchestrateJourneySteps`: schedule, waits, goal (required or optional), the
   two segment flags, `ignoreThrottling`.
2. `getOrchestrateJourneyVisitors` with `enrollmentFilter=all` and `limit` up to 5,000:
   - the uncapped `summary.numVisitors`;
   - each person's `enteredAt`;
   - each person's step entry times;
   - where each in-progress person is now.

   Note `meta.evaluatedAt` and `summary.truncated`.
3. `getOrchestrateJourneyStepMetrics` over the journey's live dates.
4. `getOrchestrateJourneyStepVisitors` with `eventType=sent`, then `delivered`, for the email step in question.
5. `getSegment` on the entry segment.

## Build a step-by-step count of people

Use one column for each stage, and keep the same unit (people) all the way down:

1. **Entered**: `getOrchestrateJourneyVisitors` with `all`, `summary.numVisitors`.
2. **Reached the step**: people whose `stepVisits` include the step.
3. **Sent**: step metrics.
4. **Delivered**: step metrics.
5. **Opened or viewed**: step metrics.

Give each drop its count and its share of the stage before it.

## Explain each drop

Use two lists, never mixed.

**Counted** (from tool fields, or with a counting method from the contract):
- People still waiting before the step, as of `meta.evaluatedAt`: they are in progress at an earlier step.
- People whose wait ends after the schedule end: they will never reach the step.
- People already marked completed (`completed=true`) before the step, whether by an exit step or a required goal.
- Bounces, from step metrics. A bounce happens after a send, so it never explains a missing send.
- **People with no email address.** Count them (contract: *People with no email address*).
- **Inactive for over 12 months** when `reachInactiveVisitors` is false. Count them on the audience segment
  with `buildPendoSegment` (last visit more than 365 days before the journey started) and `listVisitors`.
  Label the count as matched from today's data.

**Possible causes** (not counted; label each one and name the check that would confirm it):
- **Left the entry segment.** Possible when `retainVisitorsAfterEntry=false` and the rule is one people leave by
  acting.
- **Completed through the goal.** Which completed people were removed for reaching a required goal. Optional goals remove nobody. Confirm by checking the goal item's events after each person's `enteredAt`.
- **Unsubscribed or suppressed** before the send.
- **Throttled at entry.** This explains people **missing from "entered"**, never a missing send. Who was held
  is not recorded.
- **Email throttle.** A possible cause of a missing send, effect unknown. Never call it the cause.
- **Third-party email steps** (HubSpot, Marketo, Eloqua). Pendo sees only the hand-off, and its metrics can lag
  by hours.

Never say "today's segment" is the audience the journey had. Segments are evaluated on today's data.

**Also say:**
- *Completed* is not a drop-off. It means people who reached an exit step or the journey's end.
- *Entered* is visitors. If the audience is defined by account, or the asker talks about accounts, give the
  accounts reached too (contract: *Accounts*).

## Timing questions

- **Configured time to each message** is the sum of the waits on the path, from `getOrchestrateJourneySteps`.
  Explain what each wait means: a wait after a message is how long people stay before the next step.
- **Observed lag:** use the contract's *How long until a message is sent* method: daily entries against daily
  sends. Give the share sent the same day and the next day or later.
- Check the schedule's day rules (weekend skips) and say how they move the dates.
- Step entry time is **not** send time. Never call a step-entry time "sent".

## The list

When asked "who" or "can I export them":
- **Where you can write files:** save the IDs to a CSV and give the path. Never paste visitor IDs into a
  shared answer.
- **Where you can't:** give the counts, and say the per-person list is not available here.
- **If a list was cut at 5,000:** say so, and say the rows are the first 5,000 by visitor ID, not a sample.

## Cannot answer

- The recorded reason a specific person was not sent an email. It is not stored.
- How many people the journey throttle held. It is not stored.

Say both plainly when asked.
