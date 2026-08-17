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

## 0005 — Stack and computer-use technology deferred pending a research pass

**Date:** 2026-08-17
**Status:** Open — deliberately deferred

Language/runtime, computer-use technology, perception strategy, and artifact serialization are
explicitly our call (brief Section 4) and remain open pending a dedicated research pass across
the options. Current leanings and their reasoning are in the "Open decisions" table in
[architecture.md](architecture.md); each becomes its own entry here once settled.

Deferring these is safe because [scope.md](scope.md) defines the submission in terms of
contracts and seams — surface driver, artifact schema, policy chokepoint, session ownership —
none of which depend on the library underneath. Committing to a stack before researching it
would mean defending a choice made for convenience, and the brief requires defending every
choice in detail.

The perception strategy is the one to settle most carefully, because the brief says to bias
toward an approach that still works with no clean DOM, and that choice determines what a
`Target` in the artifact schema can even refer to.

---

## 0006 — Proxy target: a locally built, deliberately hostile back-office app

**Date:** 2026-08-17
**Status:** Accepted

A mock member-servicing console, built locally: legacy-shaped markup (nested tables,
frames/iframes, no test IDs, ambiguous labels), seeded fake data, a search → detail → action
flow with a confirmation screen, and a fault-injection switch for record-not-found, validation
error, permission denial, unexpected interstitial, session timeout, slow load, and app error.

Derived from scope rather than preference — the reasoning is in [scope.md](scope.md) under
"What the target application must therefore be". In short, the scope imposes seven
requirements on the target and a public demo site fails four of them: it cannot emit a
permission denial or session timeout on command, it can change between the discovery run and
the replay run, it carries terms and rate-limit exposure, and it is not offline. Requirement
3.3 is the third-heaviest evaluation criterion and its `/evidence/` error case is explicitly
requested, so a target that cannot produce runtime errors on demand makes the highest-value
part of the submission undemonstrable.

The target is therefore not a distraction from the project — it *is* the fault-injection
harness that requirements 3.3 and 6.3 need.

Rejected: a public demo/sandbox site (fails the above, though it would have been cheaper and
unambiguously "real"); a desktop app (leans furthest into their environment, but coordinate
targeting undermines the deterministic-replay story that carries more evaluation weight).

Known weakness, to be stated in the write-up rather than buried: authoring the target
alongside the agent that drives it risks stacking the deck. Mitigations — make it genuinely
hostile rather than politely legacy-themed, keep its source out of the agent's context so
discovery must actually perceive the UI, and disclose the concern in `REPORT.md`.

A second, differently-branded variant of the same app — a stand-in for two tenants running one
vendor product — is deferred to the stretch goals, per the brief's instruction to attempt those
only on top of a solid core.
