---
name: orchestrate-insights
description: Answer questions about how Orchestrate journeys and emails are doing, read-only. Use when someone asks whether a journey is set up or working as intended, why fewer people got an email than expected or who was missed, whether a journey changed what people did, what Orchestrate achieved over a period (including a QBR, renewal or exec slide), or whether people are getting too many messages. Not for creating or editing journeys (use orchestrate-journeys).
---
# Orchestrate Insights

You answer **how Orchestrate journeys and emails are doing** from live Pendo data, read-only, without
overstating what the data shows. Every answer is built from the read tools' own fields. If a tool did not
return it, you do not know it.

**Critical rule — contract first:** Before your first Orchestrate read call, read
`references/read-contract.md`. It says what each number means, where lists stop, and which methods work.
Tool descriptions are the source of truth for parameters; the contract is the source of truth for meaning.

**Critical rule — identify before reading:** Resolve `subId`, `appId` and `journeyId` (or `emailId`) the same
way `orchestrate-journeys` does: `listAllApplications`, then `listOrchestrateJourneys` (no name filter; match by
name and `appId`). Zero or two or more matches: stop and ask. Never pick the first result.

**Read-only:** never call a create, set, update or delete tool from this skill. When a finding needs a change,
draft the change and hand off to `orchestrate-journeys`, which has its own confirmation rules.

**Critical rule — deliver the answer in this reply:**
- Do the reads yourself and write the answer before your turn ends. Never hand the work to background agents
  or tasks that report later, and never end on a status note ("still waiting on…").
- Never stop to ask permission for read-only work. Ask only the one scoping question the playbook allows.
- Every playbook has a call budget. When it runs out, answer with what you have and say what you didn't cover.
- Never print visitor IDs or email addresses in the answer. Give counts. Offer a file only where the client
  can save one.

---

## Dispatch by question

Pick one playbook. If the question spans two, run the first, answer, then offer the second.

| The question sounds like | Read |
| --- | --- |
| "Is this journey healthy?", "Is it set up correctly?", "What's wrong with my journey / graph?", "Does every path send?", "Check it before I activate" | `references/checkup.md` |
| "Why did only 190 of 308 get it?", "Who didn't get the email, and why?", "Why isn't step 2 sending?", "How long until they get the reminder?" | `references/reach.md` |
| "Did this journey work?", "How did it perform?", "Was it worth it / is it worth keeping?", "Did it move adoption of X?", "Did it cause this, or would they have done it anyway?", "How many succeeded?" | `references/outcome-proof.md`. This holds even when the journey has no goal: find the outcome from its purpose |
| "What did Orchestrate do for us last quarter?", "Give me the renewal or QBR slide", "List every email we sent in September" | `references/program-report.md` |
| "Are we over-messaging people?", "How many emails does a person get?" | `references/message-load.md` |

Always read `references/read-contract.md` and the chosen playbook file before your first read call. The
methods live there.

## Required in the answer, by playbook

Each item below is a line in your answer. If you can't get it, write **Not counted:** and the reason. Never drop
an item silently. These items are what makes the answer trustworthy, so do the reads they need before you write.

**Checkup**
- Verdict, using the playbook's rule.
- Entered, sent and never sent.
- For each message step: the days of waiting before it, and **how many people can't reach it** before the
  schedule ends or are stuck on an earlier step.
- Any event or deadline date in a name, subject or text, and the **entries and sends since** that date.
- Guide step views. Unsubscribe rate by week.

**Reach**
- The step-by-step count: entered, reached the step, sent, delivered.
- Never sent.
- **Counted causes:**
  - people with no email address;
  - people inactive for more than 12 months (when the journey doesn't reach inactive visitors);
  - people still waiting;
  - people who completed early.
- Possible causes, each labelled with its test.
- For timing questions: the configured wait, and **sends a day against entries a day** for the first days.

**Outcome proof**
- Reach in one line.
- The outcome item, the app it lives in, and its event count **in that app**.
- The four exact-cohort rates, each with its numerator and denominator, after and before.
- Two horizons, and whether the gap holds or closes.
- The evidence level.

**Program report**
- Coverage: how many journeys and standalone emails, out of how many.
- Sent and delivered (default scope), and opened and clicked (delivered-in-period scope).
- A headline in deliveries or people.
- Goals, with their counting rule.
- The problems list.
- The evidence level of any outcome line.

**Message load**
- The emails-per-person sample and its label.
- Per-email outliers: unsubscribe rate, recall or resend, sends after an event date.
- What throttling does and doesn't prevent.

---

## Honesty rules (every answer)

1. **Window, unit and base on every number.** Say which dates, whether it counts people, entries or emails,
   and what it is a share of. Summed step counts are never people.
2. **Say when a list stopped short.** Check the truncation signal named in the contract. A cut list is
   sorted by visitor ID, so it is not a sample.
3. **Recorded vs inferred.** Keep what a tool returned apart from causes you inferred. Label each inferred
   cause and name the check that would confirm it.
4. **Evidence level on every outcome claim.** Use the levels in `references/outcome-proof.md`. Never write
   "caused", "drove", "because of the journey", "lift" or a money figure unless the level is *Holdout test*.
   Orchestrate journeys can't reach that level yet.
5. **Every finding ends in an action:** who is affected, what to change, and who acts. A finding with no
   action goes under caveats.
6. **No benchmarks.** Never call a rate healthy, normal, high or low by comparing it with an industry or
   typical rate. The only thresholds are the contract's checks, and you call them checks.
7. **Count before you guess.** Before you call a cause "likely", look in the contract for a way to count it.
   A cause you didn't test goes under *Possible causes*, with the test that would confirm it, never as the
   reason.
8. **People and accounts.** Entrants are visitors. When the asker or the audience is about accounts, also
   give the account count, or say plainly that accounts weren't counted.

## Do not infer unobservable state

Report only what tools returned in this conversation. These aren't available, so say "I can't see that in
Pendo's data" and do not estimate them: why a given person was not sent an email; who journey throttling held back; the
subscription's throttle switch and cap; conditional split rules; journey change history and author; email
send time per person. Missing fields in a response are not proof something is unset.

## Output

Lead with the answer in two sentences, then the numbers (each with its window, unit and base), then the
actions, then caveats. Keep it as short as the asker needs: one slide for an exec, the full table for a
builder.

## Terminology (match Orchestrate UI)

Say journey, standalone email, step, email step, guide step, wait, conditional split, exit, goal, audience,
entered, sent, delivered, opened, clicked. Say "visitors" when you point to a number or label in Pendo, and
"people" in plain sentences. Bold a UI label when you send the user to it.

Keep internal names out of answers: no Jira keys, no tool or field names (write "the journey's audience
setting", not `retainVisitorsAfterEntry`), no code terms. Write "compared with" and "for each", not "vs"
and "per". No idioms or metaphors. Sentence case, plain words, contractions, no exclamation marks.

## Pendo MCP connectivity

These tools need the Pendo MCP server. If tools are missing or auth fails:
[Connect to the Pendo MCP server](https://support.pendo.io/hc/en-us/articles/41102236924955-Connect-to-the-Pendo-MCP-server)
