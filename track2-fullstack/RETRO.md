# Retrospective

Author: Farbod Naji
Date: 2026-05-16

---

## Trade-offs

The most significant trade-off was writing an architectural proposal rather than
implementing the refactor. The 3-layer Service + Repository structure is the
right fix, but implementing it cleanly within the assessment window risked
introducing new bugs while reorganising working code. A proposal that is
specific enough to execute felt like the more responsible choice given the scope.

On the frontend, I considered placing both data tables at the top of the animal
detail page followed by both forms side by side below — which gives a cleaner
read-before-write layout and keeps the page compact. I held back because capping
table height with a scrollbar (so a long weight history doesn't push the forms
off-screen) and laying out two forms side by side are presentational decisions
that should go through whoever owns UI/UX before being implemented. I noted the
proposal rather than shipping it unilaterally.

All bugs found in the audit were fixed — the codebase was small enough that
prioritisation didn't require leaving any of them unresolved.

---

## What I'd Do Differently With More Time

I would implement the architectural refactor rather than propose it. Beyond that,
the frontend is missing basic CRUD access for most of the features the backend
already supports — there is no UI for adding animals, editing them, or moving
them between paddocks. Those would be the next additions. I would also implement
the table UX improvements described above once design direction was confirmed.

---

## What I Deliberately Left Alone

The `paddocks.js` route has no PUT or DELETE endpoints — paddocks cannot be
edited or removed through the API. This felt out of scope for the assessment and
carries enough product implications (what happens to animals in a deleted paddock?)
that it warrants a separate decision rather than a quick addition.

The frontend has no error handling UI — if an API call fails, the user sees
nothing. This is a meaningful gap but fixing it properly means designing a
consistent error state across all pages, which is a larger UX task than a
one-line addition.

Finally, I did not add any frontend interface for the existing health events or
weight logging beyond what was required by the TODO — the backend endpoints are
complete and tested, but fuller frontend integration was left for a follow-up.
