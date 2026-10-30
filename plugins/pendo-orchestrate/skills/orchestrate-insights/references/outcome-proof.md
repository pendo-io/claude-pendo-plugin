# Outcome proof

Answers "did this journey change what people did?". It reports the strongest claim the data supports and no
stronger.

## Evidence levels

Name one on every outcome claim.

| Level | When | Words allowed |
| --- | --- | --- |
| **Sent only** | No outcome measured | "N people were sent it or saw it" |
| **Before and after** | The outcome trend around the send, with no comparison group | "Use of X rose from A to B in the week after the send." |
| **Engaged and not engaged compared** | People who engaged compared with people who didn't, after the send | "People who opened did X more often (a compared with b)." |
| **Compared, with a before comparison** | The same comparison for an equal period **before** the send, shown next to it | "The gap was c before the send and d after." |
| **Holdout test** (a held-back group) | A random group was held back from the journey | "The journey raised X by …" Orchestrate journeys can't do this yet: the held-back group setting is stored but not used |

Never write "caused", "drove", "because of the journey", "lift", "ROI" or a money figure below *Holdout test*.
If the gap existed before the send, say so first.

## 1. Define the outcome

Follow the contract's *Find the outcome item* method. In this order:
1. **The audience rule.** "Never used X" or "has not done X" means X is the outcome.
2. **The journey's goal**, if it passes the checks below.
3. **The stated purpose**, searched across **every app**.
4. **Ask** (one question), only if nothing is found.

Start the answer with reach in one line (entered, sent, never sent), then the outcome.

**State the success rule in words**, for example "used report builder at least once within 14 days of the
send". If it says "within N days of **each person's** email", note that per-person send times can't be read.
Use the send dates, labelled as an approximation.

## 2. Check before measuring

- **Outcome has recorded events.** `entityUsage` on the outcome item for the window, with no visitor filter. No
  events: stop and propose an item that has events.
- **Goal too common.** Measure it on the journey's own entrants before the send. See the contract.
- **Toggle goals.** A toggle's click counts turning it on **and** off. Say "used the toggle", never "enabled".
- **Feature lives elsewhere.** If the outcome lives in another app, measure it there.

## 3. Measure with exact cohorts

Use the contract's method, for each main email step:

1. `buildPendoSegment`: did the outcome in the **after** window. Build the same for an equal **before** window
   ending the day before the first send.
2. `getOrchestrateEmailMetrics` with `metricsScope=emailsDeliveredInPeriod` over the send window, called three
   times:
   - with no segment;
   - with the after segment;
   - with the before segment.
3. From those counts, give four rates, each with its numerator and denominator:
   - openers who did it, after;
   - non-openers who did it, after;
   - openers who did it, before;
   - non-openers who did it, before.

**Guide steps:** use step visitors with `eventType=viewed`.
- Up to 5,000 people: measure them with `entityUsage` against the same outcome.
- Over 5,000: say the comparison is not available for this step.

**Two horizons, always.** Repeat the after window at a second length (for example 7 or 14 days, and 28 days or
to date). Say whether the gap holds, narrows or closes. Non-engagers often catch up later, which means the
journey changed *timing*, not whether people acted. One horizon is not enough to call a result.

**Where the gap comes from.** If clickers explain most of it, say so: a click on the email's call to action
leads straight to the outcome.

## 4. Compute

**Where the impact engine is available** (bundled script, or Leo's engine tool):
- Pass the counts to it for the gap, the before-vs-after difference and its interval.
- Quote only the engine's numbers.

**Where it is not:**
- Report the four rates and the two gaps (after minus before), each with its numerator and denominator.
- Call the result *Compared, with a before comparison*.
- Say no interval was computed.

Do not compute an interval yourself.

## 5. Say it

- **Verdict:** one sentence at the level reached. "Not enough data to say" is a valid verdict.
- **Numbers:** the four rates, then the two gaps.
- **Caveats:**
  - opens can be automatic;
  - the engaged group chose to engage;
  - the before-check shows how much of the gap is self-selection.
- **Action:** the next step that would raise the level. For example, a guide experiment as a holdout, or a
  second journey with a held-back segment built by the customer.

Across several journeys on one outcome, run this per journey. Never add their entrants together: journeys on
one segment overlap.
