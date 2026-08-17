# Decision log

The brief requires being able to explain and defend any part of the submission in detail.
This file is where that defense is written down as the work happens, rather than
reconstructed afterward. `REPORT.md` is the distilled version for the reviewer; this is the
full log.

One entry per meaningful decision: what was chosen, what was rejected, and why. Keep them
short. Amend rather than delete when a decision is reversed — a reversal with its reason is
more convincing than a clean history.

---

## 0001 — Documentation before implementation

**Date:** 2026-08-17
**Status:** Accepted

The repository was initialized with the requirements of record, an architecture proposal, and
this log before any code. The brief states that implementation throughput is not the
bottleneck and that "the real test is judgment and integration" — the artifact schema,
locator strategy, error taxonomy, and control-transfer model. Those are design problems, and
writing them down first makes the reasoning reviewable and gives later decisions something
to be consistent with.

Rejected: starting with a browser-driving spike and documenting after. Faster to a demo, but
the recorded reasoning would have been written to justify whatever the spike happened to do.

---

## 0002 — Assignment brief PDF obtained and requirements restated

**Date:** 2026-08-17
**Status:** Closed (superseded by the full requirements capture)

The initial commit derived requirements from the recruiting email alone, because the detailed
brief existed only as a PDF attachment. The PDF is now available and
[assignment-brief.md](assignment-brief.md) has been rewritten against it.

The email had understated the scope considerably. Three requirement areas were absent from it
entirely — safety and policy guardrails (3.4), human-in-the-loop escalation with live-session
takeover (3.6), and the heterogeneity/multi-tenant design story (3.7) — as were the mandated
deliverable paths and the seven required `REPORT.md` headings. Holding stack and scope
decisions open until the PDF was read was the right call: an architecture designed against
the email's three-phase framing would have had no place to put the session-ownership model
that 3.6 requires.

---

## 0003 — The brief PDF is not committed to the public repo

**Date:** 2026-08-17
**Status:** Accepted

`docs/assignment-brief.pdf` is gitignored. The requirements live in
[assignment-brief.md](assignment-brief.md) as our own restatement instead.

The repo is public, and the PDF is interface.ai's evaluation instrument — the same assignment
goes to every applicant. Republishing it verbatim would leak their assessment material to
future candidates, which they never asked for and which costs them something real. They asked
for the *work* to be public, not the brief. A restatement is also more useful to build
against: checkboxes, traceability, and requirement numbers that other documents can
reference.

Rejected: committing the PDF for reviewer convenience. The reviewers wrote it; they do not
need a copy.

Reversing this is one line in `.gitignore` if a different call is preferred.

---

## 0004 — Repository layout follows the mandated deliverable paths exactly

**Date:** 2026-08-17
**Status:** Accepted

`/README.md`, `/REPORT.md`, and `/evidence/` sit at the repository root with those exact
names, and `REPORT.md` carries the seven required headings in the required order. Working
documents (requirements, architecture, this log) live under `docs/` so they do not compete
with the deliverables at the root.

The brief asks for these exact paths and headings because submissions are read side by side.
Making a reviewer hunt for a document is a self-inflicted wound in a review whose stated
final criterion is communication. `REPORT.md` exists as a skeleton now so the required
structure cannot be forgotten late.

---

## 0005 — Core technical decisions (open)

**Date:** 2026-08-17
**Status:** Open

Language/runtime, computer-use technology, perception strategy, artifact serialization, and
proxy target application are all explicitly our call (brief Section 4) and all remain open.
Current leanings and their reasoning are in the "Open decisions" table in
[architecture.md](architecture.md); each becomes its own entry here once settled.

The two that most constrain everything else, and so should be settled first:

- **Perception strategy**, because the brief says to bias toward an approach that still works
  with no clean DOM, and that choice determines what a `Target` in the artifact schema can
  even refer to.
- **Proxy target application**, because the brief's interesting failures are runtime
  conditions — validation errors, not-found results, permission denials, session timeouts —
  and a target that cannot produce them on demand makes requirement 3.3 undemonstrable.
