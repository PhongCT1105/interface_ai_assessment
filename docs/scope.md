# Submission scope

What is actually being built, what is deliberately thin, and what is mocked at a clean seam.
Derived from the brief's own scoping guidance ([assignment-brief.md](assignment-brief.md)
Section 5), which is unusually explicit and worth taking literally:

> "What we're looking for is a complete end-to-end vertical slice that touches every core
> requirement in Section 3 — not just one or two of them." … "Cut depth, not whole
> capabilities."

Two consequences that shape everything below:

- **Breadth is mandatory, depth is discretionary.** All seven core requirements must be
  present and real. Depth is spent selectively, on the three the brief names as load-bearing:
  artifact schema (3.2), deterministic replay + error handling (3.3), and safety/escalation
  (3.4 + 3.6).
- **A documented mock beats a stalled feature.** "We'd rather see a well-designed seam than a
  stalled project." So the cut lines are chosen deliberately and recorded here, and they
  become `REPORT.md` §7 verbatim.

## The one thread that must run end to end

```
goal (natural language) + target
  → real LLM-driven discovery run against a live UI
  → compiled capability artifact (typed, versioned, parameterized)
  → deterministic replay with input params → typed outputs, checkpoint verified, zero model calls
  → a replay that hits an exceptional state and reports it as a business outcome, not a crash
  → an escalation that pauses a run, hands the live session to a human, and resumes
  → evidence on disk for every one of the above
```

If any link in that chain is missing, the submission is incomplete regardless of how good the
rest is. Everything in the build order below is sequenced to close this thread first and
deepen afterward.

## Depth budget per requirement

"Real" means it genuinely executes. "Thin" means the narrowest honest version. "Mocked" means
a deliberate seam, documented as such.

| Req | Real | Thin | Mocked / cut |
| --- | --- | --- | --- |
| **3.1** Agent loop | Real LLM, real clicks/typing on a live UI, observe→decide→act, stopping conditions | One goal shape, modest step budget, no reflection/backtracking beyond simple retry | No multi-goal planning, no parallel exploration |
| **3.2** Artifact | **Deep.** Complete schema: identity, versioning, typed inputs/outputs, steps with target + fallbacks + per-step expect + declared exceptional conditions + risk class, checkpoint, provenance, approval state. Validated on load | One recorded capability, one app | No visual schema editor, no migration tooling beyond a `schemaVersion` field |
| **3.3** Replay | **Deep.** Interpreter with model client structurally unreachable; locator fallback chain; condition-based waits; three-class result contract | Handles the conditions the artifact declares; no generic recovery engine | No LLM-assisted recovery (that is a stretch goal), no drift auto-repair |
| **3.4** Safety | **Deep enough.** Allowlist (origins/routes + action types) at one chokepoint covering discovery *and* replay; safe/confirm/blocked classification; redaction at the persistence boundary | Static config file; small pattern set for redaction plus `sensitive` input marking | No policy DSL, no per-tenant policy, no approval workflow service |
| **3.5** Evidence | Structured per-run event log including the model's rationale; screenshot + state snapshot on failure | Files on disk, JSONL | No log viewer, no trace UI, no retention policy |
| **3.6** Escalation | **Real mechanism.** Stuck detection; intervention request carrying context + screenshot; single explicit session owner; automation pauses; human drives the *same* live session; resume; human's actions captured | One operator, no routing, no queue, no auth | **Operator console is mocked** — explicitly permitted by the brief. A minimal local control surface, not a co-browsing product |
| **3.7** Heterogeneity | The surface driver interface genuinely exists with one implementation behind it, so the seam is demonstrable rather than asserted | One surface implemented | Legacy-web and desktop drivers are **design only**, in `REPORT.md` §4. Multi-tenant reuse is design only |

Tests, per "reasonably typed and tested where it counts":

- Artifact schema validation round-trips
- The replay interpreter against a **fake driver** — fast, no browser, and it covers the whole
  result-contract matrix (success / each business outcome / each recoverable condition / hard
  failure), which a real browser makes tedious to exercise
- Error and outcome classification
- Redaction at the persistence boundary
- **A boundary test asserting the replay module cannot import a model client** — the highest
  value per unit of effort, because it converts the submission's headline claim from an
  assertion into a mechanically checked property

Not broad e2e coverage — the evidence directory is the end-to-end proof.

## Deliverables scope

| Path | Scope |
| --- | --- |
| `/README.md` | Setup, required config/keys, **and how to run replay without live model access** (the brief asks for a no-live-services path where applicable — replay is model-free by design, so this falls out naturally and is worth stating) |
| `/REPORT.md` | Seven mandated headings, ~1–3 pages. Judgment document, not a manual |
| `/evidence/` | One artifact, one discovery run log, one successful replay log, one replay that hits an exceptional state. Optional screen recording |

## What the target application must therefore be

The target is not a free choice once the scope above is fixed — it is a test harness, and the
scope dictates its requirements:

| Scope requirement | What it demands of the target |
| --- | --- |
| 3.1 bias toward "no clean DOM" | Legacy-shaped markup: nested tables, framesets/iframes, non-semantic elements, **no test IDs**, duplicate/ambiguous labels |
| 3.3 three-class result contract, and an error case in `/evidence/` | Must produce **on demand**: record-not-found, validation error, permission denial, unexpected interstitial, session timeout, slow load, app error |
| 3.2 typed inputs/outputs, meaningful checkpoint | A non-trivial multi-step flow: search → detail → action with a confirmation screen |
| 3.6 human takes over the same live session | Must be drivable by a person in a headed browser mid-run |
| Deterministic replay, reproducible evidence | Must not change underneath the artifact between the discovery run and the replay run |
| Ground rules: no terms violations, no real credentials, no real PII | Must carry no third-party terms or rate-limit exposure |
| README "run without live services" | Must run locally and offline |

**A public demo site fails four of those seven** — it cannot be made to emit a permission
denial or a session timeout on command, it can change under you between runs, it carries terms
and rate-limit exposure, and it is not offline. Those are not incidental: requirement 3.3 is
the third-heaviest evaluation criterion, and a target that cannot produce runtime errors makes
it undemonstrable.

That leaves two live options, both of which satisfy all seven:

| Option | For | Against |
| --- | --- | --- |
| **A. Purpose-built hostile app** — mock member-servicing console, legacy-shaped markup, seeded fake data, fault switch | Total control over how hostile the markup is; fault injection is native; a second tenant variant is trivial | Authoring the target alongside the agent that drives it looks like stacking the deck |
| **B. ParaBank + fault-injection reverse proxy** — Parasoft's demo banking app (`docker pull parasoft/parabank`, JSP-era markup, real multi-step flow) with a small proxy in front returning 500s, injecting latency, dropping session cookies, and splicing in interstitials | The app is third-party code written before this project existed, which largely dissolves the stacked-deck objection; a second "tenant" comes nearly free via proxy-rewritten branding | Less control over markup hostility; one more moving part in the demo path; ParaBank's own admin page has **no** fault controls, so the proxy is load-bearing rather than optional |

Currently leaning **B**. Recorded as [decisions.md](decisions.md) 0006 (amended and reopened).

Whichever is chosen, the honest counter-argument belongs in the write-up rather than buried
here — with option A especially, the mitigations are to make the app genuinely hostile rather
than politely legacy-themed, to keep its source out of the agent's context so discovery has to
actually perceive the UI, and to say all of this plainly in `REPORT.md`.

## Build order

Sequenced so the end-to-end thread closes as early as possible, and so nothing downstream has
to be redesigned.

1. **Target app + surface driver interface** (one implementation). Nothing can be exercised
   before there is something to drive and a seam to drive it through.
2. **Artifact schema.** Before the discovery compiler, because the compiler's job is defined
   by what the schema demands. Designing this after the recorder would let accidents of the
   recorder leak into the contract — and the schema is the highest-weighted thing in the
   evaluation.
3. **Policy chokepoint + session object.** Before any live run: discovery must already be
   allowlisted, and retrofitting session ownership after the run loop exists is the mistake
   requirement 3.6 punishes.
4. **Discovery loop + compiler → artifact.** The first real LLM run. Evidence starts here.
5. **Replay interpreter + result contract.** Closes the core thread.
6. **Escalation + handoff** on top of the session object.
7. **Write-up, demo path, evidence curation.** Including the deliberate error-case replay.

## Cut list (feeds `REPORT.md` §7)

Committed cuts, to be justified rather than hidden:

- No queues, services, clusters, or multi-tenant plumbing — the brief explicitly does not
  reward building scaling infrastructure, only designing abstractions that could scale
- No desktop or legacy-web driver implementation — design only
- No real operator console; a minimal local control surface stands in
- No authentication, multi-operator routing, or intervention queue
- No LLM-assisted replay recovery, no artifact confidence scoring, no flakiness signal
  (stretch goals, only if the core is solid)
- One target app, one capability, one surface
- No cross-tenant variant unless the core lands with time to spare (see stretch goals)

## Open

- **Language, runtime, and computer-use technology are deliberately still open**, pending a
  full research pass on the options ([decisions.md](decisions.md) 0005). Nothing in this scope
  document depends on that choice, which is the point — the scope is defined in terms of
  contracts and seams, not libraries.
