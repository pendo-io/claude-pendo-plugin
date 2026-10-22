# Program report

Answers "what did Orchestrate do for us over this period?" and "give me the slide". It also handles "list
every email we sent in a month, with totals".

## Scope (state it at the top)

- **Default period:** the last full quarter.
- **Includes:** every journey active in the period and every **standalone** email sent in it.
- **Excludes:** likely tests and operational sends. Show them on their own line, labelled as heuristic.

## Reads (budgeted)

1. Page `listOrchestrateJourneys` until a short page.
2. Page `listOrchestrateEmails` until a short page.
3. Split journey emails from standalone emails with `getOrchestrateEmail.attributes.journeyId`.
4. For **every email that sent in the period**, journey and standalone alike, call `getOrchestrateEmailMetrics`
   **twice**: once in the default scope for sent, delivered and bounced, and once with
   `metricsScope=emailsDeliveredInPeriod` for opened, clicked and their rates. Both count by event date. Journey step metrics count people by when they *entered*, so they miss sends in
   the period to people who entered earlier. Don't use them for period totals.
5. Guide steps: `getOrchestrateJourneyStepMetrics`. Label those counts as "people who entered in the period".
6. Per active journey: `getOrchestrateJourneyMetrics` with no range, for the cumulative goal. Name its rule.

**Call budget: at most 40 Pendo calls** (two for each email). Say it at the start.
- When it runs out, report the coverage you reached: "covers 18 of 31 journeys, ranked by sends".
- Never present partial coverage as the whole.
- Cover the largest objects first: rank by sends from the listings and the cheapest metric call.
- Use the contract's two-call scope rule: sent, delivered and bounced from the default scope; opened and
  clicked from `emailsDeliveredInPeriod`.

## Numbers

**People first.**
- A person in two journeys counts once in "people reached".
- Where you can't deduplicate, say "entries" or "sends" and label the headline that way.

**Rates:** open and click rates come from `emailsDeliveredInPeriod`, as a percent of delivered. Sent, delivered
and bounced come from the default scope.

**Trust label** on each headline number:
- *Exact*: read directly.
- *At least*: a lower bound from lists that stopped short.
- *Estimated*: test or operational classification.

Outcome numbers carry their evidence level from `outcome-proof` instead of a trust label.

**Problems to disclose** (one line each, with the count):
- journeys still sending after their event date;
- bounce or unsubscribe rates that stand out against the rest of *this* subscription;
- guide steps with 0 views;
- people entered but never sent anything;
- operational, test or recall sends, kept out of campaign totals.

**One set of numbers.** The overview, the slide and any reconciliation table use the same figures from the same
reads.

## Outcomes

Give an outcome only for journeys that went through `outcome-proof`, at the level it reached. For the rest,
show reach and engagement, and say no outcome was measured. Offer `outcome-proof` for the top journeys by
people reached.

## Output by audience

| Asker | Give |
| --- | --- |
| Exec or finance (renewal, budget) | One slide. Three numbers, each with a trust label (the headline is deliveries or people, never entries), one outcome line at its evidence level, the problems in one line, and one "what's next" (goals plus a held-back group as the path to proving impact). No ROI, no "speed to market" claims |
| CS leader | One page: the slide plus top journeys and any drop-offs, with an offer to run `reach` on them |
| Builder or owner | The full table for each journey and email, then drop-offs and goals with no recorded events |

Drop-offs found along the way (sent well below entered, goals with no recorded events, guide steps never shown)
go under "Needs attention", each with an offer to run `checkup` or `reach`.
