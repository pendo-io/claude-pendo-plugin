---
name: guide-management
description: Run a guide-review pass over the Guide Signals Pendo has already produced — list the archive, deactivate, and anomaly signals for an app, show each with its evidence and how old it is, then let the user mute a signal or deactivate/archive the guide. Use this skill whenever someone wants to review their guides, clean up stale or declining guides, see which guides Pendo recommends archiving or deactivating, triage guide signals or anomalies, or snooze a signal — even if they don't say "signals" explicitly. Nothing is computed here; the skill reads what the weekly job produced and never changes a guide without an explicit choice.
---

# Guide Management

Turn Guide Signals into a repeatable guide-review pass. A weekly Pendo job decides which guides deserve attention; this skill reads those signals, shows the evidence so the reader can judge for themselves, and offers the only two actions available — **mute** a signal, or **change the guide's state**.

## Tools

Pendo MCP tools only (prefixed `Pendo:` in the tool list). Three do the work:

- `Pendo:listSignals` — the only source of signals. One call per signal type for one app, paged with `offset`.
- `Pendo:muteSignals` — snoozes signals by id so they stop appearing.
- `Pendo:setGuideState` — changes a guide's state (`disabled` to deactivate, `_archived_` to archive).

Helpers: `Pendo:listAllApplications` to resolve the subscription and app, and `Pendo:searchEntities` or `Pendo:listGuides` only to turn a signal's guide id into a guide name.

Read each tool's input schema before calling it and pass only the parameters it advertises. If a parameter this skill expects is not offered, say so and carry on without it.

## This skill does not compute

The job owns detection, thresholds, trend fitting, and signal definitions. This skill must not re-derive any of them:

- Do not decide a guide "should" be flagged, or un-flag one the job flagged.
- Do not fit, smooth, or recompute a trend from the weekly counts.
- Do not hold your own thresholds for what counts as a decline or an anomaly.
- Show a figure the row does not contain as "not provided" — never estimate it.

## Parameters

Take these from the user's request; otherwise use the default. State the values in use at the start of the run.

- **app**: the application to review (resolved in Step 0).
- **signal_types**: any of `archiveRecommendation`, `deactivateRecommendation`, `anomaly`. Default: all three.
- **max_signals**: how many signals to surface per signal type. Default: 10.

## Step 0: Resolve the App

Call `Pendo:listAllApplications`. Use the app the user named; if there is exactly one app, use it and say which. If there are several and none was named, list them and ask. Do not silently pick.

## Step 1: Fetch the Signals

Call `Pendo:listSignals` once per selected signal type for the app, in parallel. Signal type is a filter you pass through, not a routing decision. Request more pages with `offset` only until `max_signals` is reached for that type.

Resolve each signal's guide id to a guide name before presenting it.

## Step 2: Say What the Run Found, Honestly

Check this before presenting anything.

- **As-of date.** Every signal came from a job run. State the as-of date of the run that produced each one, using the row's `lastUpdatedAt`. If it is more than a week old, say so plainly and give its age ("this signal is 19 days old — the weekly job may have failed since"), and say it may no longer describe the guide today. If `lastUpdatedAt` is missing, say "as-of date not provided" — never imply the signals are current.
- **No signals is not the same as the job not having run.** If the tool reports that the job has never run for this subscription, or reports no run information at all, say that: "Pendo has not produced guide signals for this subscription yet." If it returns an empty list and you cannot confirm a run, say "no signals returned, and I can't confirm the job has run" — never "no problems found" or anything that reads as a clean bill of health. Only report a clean result when the tool confirms a completed run that found nothing.

## Step 3: Present Each Signal

Group by signal type, and within each group give the guide name, id, and as-of date. Then the evidence, exactly as returned.

**deactivateRecommendation** — the guide's views have been falling:
- Weekly view counts, oldest to newest, and the number of weeks observed
- Weekly loss percentage and half-life (the weeks for weekly views to halve)
- R² — how closely a steady decline fits the weekly counts
- When the guide was published

If the row carries only a raw decay rate, show it labelled as returned instead of converting it yourself.

**anomaly** — views departed from the guide's usual pattern:
- The weekly view counts, oldest to newest, with their week dates. Show the weeks as returned and let the departures stand out on their own; do not recompute, rank, or pick out anomalous weeks, and report `maxVariance` as returned if present.

**archiveRecommendation** — the guide has not been live for a long time:
- Its current state and when it was last saved

**Reading R².** R² says how well a straight-line decline describes the past weekly counts. It is not a probability or confidence that the guide should be retired — a guide can fit a decline perfectly and still be worth keeping, and a poor fit does not make it healthy. Say this in your own words when you present it, and never write "X% confident" or equivalent.

**Seasonal series.** When the weekly counts show a repeating rise and fall — for example a guide that peaks around an annual event — point it out and suggest **deactivating** until the season returns rather than archiving. A series shorter than the cycle cannot show seasonality; if the observed weeks are too few to tell, say that instead of guessing.

## Step 4: Take a Decision on Each Signal

Walk the signals one at a time. For each, ask the user what to do. The choices are:

1. **Keep the guide and mute the signal**
2. **Deactivate the guide** (`disabled`) — reversible, it can be re-published
3. **Archive the guide** (`_archived_`) — heavier: it leaves the guide list and needs delete-level permission
4. **Skip** — do nothing and move on

Suggest an option when the evidence points to one, but never choose for the user and never act without an explicit selection. Silence, "looks fine", or "sure" about the evidence is not a selection.

**Changing state.** Before calling `Pendo:setGuideState`, state the guide's name, the current state, and the target state (current → target), and get explicit confirmation. If the user lacks permission (archiving requires delete rights) or the tool rejects the change, report the reason as returned and offer the alternatives.

**Muting.** Muting is per guide-and-signal and exists so the same finding does not come back on the next pass. Ask how many days to mute for; the tool accepts a bounded range, which its schema states, and the user chooses the number — never pick one. Confirm the guides, signal types, and duration, then call `Pendo:muteSignals` with the signal ids (batch the kept ones into one call, within the tool's per-call limit). Report each result.

Describe a mute accurately: it hides the signal until the date returned, and it rides on the signal row, so it ends when that row is replaced or expires. Do not call it a fixed 30-day silence.

## Step 5: Close the Pass

Summarize in a few lines: how many signals were reviewed, which guides were muted (until when), deactivated, archived, or skipped, and any signals that were stale or could not be acted on. Remind the user that muted signals will not come back in the next pass, and skipped ones will.

## Rules

- **Never change a guide's state, or mute a signal, without the user's explicit choice** for that specific guide and signal.
- Read only through `Pendo:listSignals`; the skill holds no thresholds, fit logic, or signal definitions.
- Always show each signal's as-of date, and state its age plainly when it is over a week old.
- Never report an unrun job, or an unconfirmed run, as "no problems found".
- Never present R² as confidence that a guide should be retired.
- Suggest deactivate over archive for seasonal series.
- Show a missing figure as "not provided"; do not derive it.
- Keep responses outside the signal review brief — let the evidence speak.
