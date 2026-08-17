# Decision log

The brief requires being able to defend everything submitted. This file is where that
defense is written down as the work happens, rather than reconstructed afterward.

One entry per meaningful decision: what was chosen, what was rejected, and why. Keep them
short. Add entries when a decision is made, and amend them (rather than deleting) when a
decision is reversed — a reversal with its reason is more convincing than a clean history.

---

## 0001 — Documentation before implementation

**Date:** 2026-08-17
**Status:** Accepted

The repository was initialized with the requirements of record, an architecture sketch, and
this log before any code. The assignment is graded on judgment as much as output, and the
hard problems here are design problems: what a "capability" is, how targets stay stable
across runs, and how replay is kept structurally free of model calls. Writing those down
first makes the reasoning reviewable and gives later decisions something to be consistent
with.

Rejected: starting with a working browser-driving spike and documenting after. Faster to a
demo, but the recorded reasoning would have been written to justify whatever the spike
happened to do.

---

## 0002 — Assignment brief PDF not yet in the repo

**Date:** 2026-08-17
**Status:** Open

The detailed brief exists only as a PDF attachment on the recruiting email, and the
requirements captured in [assignment-brief.md](assignment-brief.md) are derived from the
email body alone. Stack and scope decisions are deliberately left open until the PDF is
read, since it may specify interfaces, a target UI, or evaluation criteria that would
override them.

Action: save the attachment as `docs/assignment-brief.pdf` and reconcile
`assignment-brief.md` against it.
