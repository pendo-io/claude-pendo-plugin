# Message load

Answers "are we over-messaging people?" and "how many messages does a person get?".

## What can and can't be counted today

- **Can count:** per person, the emails they received in a window (`getOrchestrateVisitorEmails`), and the
  journeys they are in (`getOrchestrateVisitorJourneyState`). This works one person per call.
- **Can't count:** an exact messages-per-person distribution across every journey and standalone email.
  There is no tool that rolls this up for everyone.
- **Throttling, can't count:** journeys a person was held out of by the journey throttle. They are **omitted**
  from `getOrchestrateVisitorJourneyState`.
- **Throttling, can't read:** the subscription's throttle switch and caps.

## Method (a labelled sample; budget about 30 calls)

1. List the journeys and standalone emails that sent in the window. Reuse `program-report`'s listing.
2. Pick the people to check:
   - Take recipients from the largest sends with `getOrchestrateJourneyStepVisitors` or
     `getOrchestrateEmailVisitors` (`eventType=delivered`).
   - Draw a random subset of 50 to 100 from the returned rows.
   - Rows are sorted by visitor ID. If the list was cut, say the subset comes from the first rows only.
3. For each person in the subset, call `getOrchestrateVisitorEmails` over the window.
4. Report the distribution in the sample: share of people with 1, 2–3, 4–6 and 7 or more emails.
   - Label it **sample, n = …, drawn from …**.
   - Do not scale it up to the whole audience.
5. List `ignoreThrottling` for each active journey. Journeys that ignore throttling stack freely. Throttled
   ones can still stack with standalone emails and with each other when the subscription setting is off (not
   readable). Never say "nothing can stack".
6. **Find the outliers.** For each email that sent in the window (default scope for sent, `emailsDeliveredInPeriod`
   for opens):
   - unsubscribe rate, against the rest of this subscription;
   - repeat or recall sends (a name or subject with "recall", "resend", "reminder" or "correction");
   - sends after an event date in the name or subject.

   These are usually the real over-messaging problem, not the average.

## Say it

- Lead with the answer: is anyone over-messaged, and which sends cause it (the outliers).
- Then the sample distribution and its label.
- Then name the journeys that ignore throttling and overlap in audience or dates.
- **Action:** which journeys to stagger, or which to stop ignoring throttling. Make the throttle and cap
  settings a human check in Orchestrate settings, because they can't be read.
