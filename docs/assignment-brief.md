# Requirements of record

Restatement of the take-home brief in a form that can be built and tracked against.
The authoritative source is `docs/assignment-brief.pdf` (10 pages, not committed — see
[decisions.md](decisions.md) 0003). Where this file and the PDF disagree, the PDF wins.

| Field | Value |
| --- | --- |
| Title | Take-Home Project: Computer-Use Automation System |
| From | interface.ai Engineering Team (via Ashby, `no-reply@ashbyhq.com`) |
| Received | 2026-08-13 |
| Format | Design + working implementation + short write-up, public GitHub repo |
| Time box | No deadline; focused effort, not a polished product |
| Submission | Push to public GitHub, email the URL to `assignments@interface.ai` |

## What is being built

interface.ai builds AI agents for banks and credit unions. This project is the **backend
integration layer that gives those agents hands** — the system that lets an agent operate
an institution's back-office applications when there is no API to integrate through.

The through-line, in the brief's own words:

> The model discovers. The artifact becomes a reusable capability. Deterministic replay is
> how the AI agent invokes it in production.

API integration is the preferred path in real life and is **explicitly out of scope here**.
The whole point is the long tail of legacy apps — core banking screens, servicing tools,
admin consoles — where the only way in is to drive the UI like a human operator.

## The environment being designed for

Three properties shape every decision. These are the reason the design has to be shaped a
particular way, so they are worth internalizing before choosing anything.

1. **Stable UIs, but real runtime errors.** Enterprise back-office apps change slowly —
   that slowness is exactly what makes record-once / replay-many viable. So the hard
   problem is *not* layout drift. It is that replay must cope with the errors that
   legitimately occur at runtime: validation errors, "record not found", permission
   denials, unexpected confirmation dialogs, session/timeout expiry, transient slowness,
   outright app errors. *A capability that only works on the happy path is not useful in
   production.*
2. **Heterogeneous, often legacy surfaces.** A target may be a modern web app, a legacy
   server-rendered app (framesets, nested tables, non-semantic markup, no test IDs), or a
   native desktop application. No clean DOM, no stable selectors, no API can be assumed.
   Often the only reliable surface is what a human operator sees and does.
3. **Multi-tenant at scale.** Hundreds of tenants × ~20 apps each = thousands of app
   instances, and many tenants run *the same vendor product* configured, branded, and
   versioned differently. Automation should generalize across them — or degrade
   gracefully — rather than being rebuilt per tenant.

The brief is explicit that it is **intentionally under-specified**: "Part of what we're
assessing is the judgment you apply when a problem is open-ended. Where the brief doesn't
dictate an answer, make a decision, and tell us why."

## Core requirements (must-have)

Section 3 of the brief. These are what the submission is evaluated against.

### 3.1 Goal-driven agent loop

- [ ] Accept a goal (natural language) + a target (app / URL / entry point) as input
- [ ] LLM-driven **observe → decide → act** loop against a live surface, until the goal is
      met or a stopping condition hits (max steps, timeout, dead-end)
- [ ] Must genuinely interact with a real UI: click, type, navigate, read state
- [ ] Mechanism is our choice, but **bias toward an approach that still works with no clean
      DOM** — that is the common case in their environment

### 3.2 Structured artifact (an agent-invocable capability)

After a successful run, emit a typed, serializable artifact. Not a step list — a
**capability contract an AI agent can call**. At minimum it must express:

- [ ] the ordered steps / actions
- [ ] how each target element/control is identified, *with reasoning about robustness*
- [ ] typed **input parameters** (what the agent supplies per invocation, e.g. a member ID)
- [ ] typed **outputs** / data to extract, and their shape
- [ ] a **checkpoint** or success condition
- [ ] versioned and reviewable — a human reviewer *and* a calling agent must both be able to
      understand what it does, what it needs, and what it returns

> "Design the schema deliberately; it's a focal point of the evaluation."

### 3.3 Deterministic replay (the production execution path)

- [ ] Given a saved artifact + input parameters, replay **without invoking the LLM for
      decisions**
- [ ] Stable element/control targeting; verify the checkpoint; return declared outputs
- [ ] Handle errors and exceptional states explicitly, distinguishing three classes in the
      **result contract**:
  - **expected business outcomes** the caller needs to know about ("no such member" is a
    legitimate result, not a crash)
  - **recoverable conditions** (dismiss a known interstitial, wait/retry a transient load)
  - **hard failures** that stop and surface a clear, debuggable error
- [ ] Structured result: success (with outputs) | known business outcome | failure with
      enough detail to debug (which step, what was expected, what was observed)

### 3.4 Safety & policy guardrails

- [ ] Explicit, configurable **allowlist** (permitted domains/routes, permitted action
      types). The agent must not act outside it.
- [ ] Distinguish safe/reversible actions from risky/irreversible ones; handle the risky
      class conservatively — block, require confirmation, or flag (our call, must justify)
- [ ] **Never persist secrets or raw sensitive data** (credentials, tokens, full PII) into
      artifacts or logs. Redact appropriately. This is regulated financial data.

### 3.5 Evidence / observability

- [ ] Structured log of what the agent did **and why**
- [ ] At least one richer signal on failure (screenshot, DOM snapshot, trace — our choice)

### 3.6 Human-in-the-loop escalation & handoff

- [ ] **Detect and route**: identify a stuck/blocked state, raise an intervention request
      carrying enough context to act on (which capability/goal, current step, current state
      or screenshot, why it stopped)
- [ ] **Take control of the live session**: the human operates *the same live session* the
      automation was using — not a fresh one — performs manual steps, then hands control
      back so the run can resume or complete
- [ ] Preserve context and evidence across the handoff, and **record what the human did**
- [ ] There must be a way to know who is (or should be) in control

Scope note from the brief: a full real-time co-browsing operator console is **out of
scope**. A minimal but real handoff — pause automation, expose the live session for manual
control (even a bare/mock operator surface), signal resume, capture the human's actions —
plus a clear design for the rest, is what they want. *Mock the operator UI if needed, but
make the handoff mechanism and control-transfer model real and well-reasoned.*

### 3.7 Design for heterogeneity & scale (design, not necessarily build)

Implement against one concrete surface, but the write-up must credibly address:

- [ ] **Surface abstraction**: how the artifact schema and replay engine extend from the
      chosen surface to a legacy web app and/or a desktop app. What is the seam between
      "how we perceive/act on a surface" and "the recorded flow"?
- [ ] **Multi-tenant reuse**: how to represent an artifact so it can be reused (or safely
      specialized/overridden) across tenants running the same app, rather than re-recorded
      per tenant. How to detect and manage per-tenant/version drift.

> "We don't expect you to implement multi-tenant or desktop support. We do expect the core
> abstractions not to paint you into a corner."

## Explicitly our call (Section 4)

Choose and defend: language/runtime/frameworks · LLM provider & model, prompting, agent
loop structure · computer-use technology (Playwright, Puppeteer, Selenium, a CUA/agent SDK,
screenshot control, accessibility APIs, OS automation) · target application · artifact
schema and serialization · how determinism is achieved (locator strategy, fallbacks,
waiting) · architecture and boundaries (single process vs. services, sync vs. queued —
simpler is fine if justified).

**Target application constraints.** No access to a real bank system, and we must not try to
obtain one. Pick a proxy target that exercises the interesting problems: a non-trivial
multi-step flow (search → detail → action, or a multi-field form with a confirmation step).
Good options: a public demo/sandbox site, a local sample app we build or mock, or — to lean
into the legacy reality — an intentionally hostile surface (iframes/framesets, table-based
layouts, no test IDs), or even a simple desktop app. If using a public site: respect terms
and rate limits, never use real credentials or real PII.

## The one thing that is not our call

> "The discovery run has to be real. At least one genuine LLM-driven run against a live
> surface, with the evidence in `/evidence/` to show it happened. That's the heart of the
> project and we can't assess a description of it."

Requires our own model API access. A single successful run is cheap. Everywhere else, a
clean documented seam is acceptable — they would "rather see a well-designed seam than a
stalled project."

## Scope & expectations (Section 5)

AI-assisted development is assumed, and that assumption **sets the bar**: the
scaffolding-heavy parts (agent loop, schemas, replay executor, guardrails, logging) come
together fast, so what they want is a **complete end-to-end vertical slice touching every
core requirement in Section 3** — not one or two of them.

The working thread that must run all the way through:

```
goal → LLM-driven run that completes it → saved capability artifact →
deterministic replay with input params, outputs, and error/outcome handling →
human-escalation path that can take over the live session → evidence for both runs
```

- **Go deep where it matters**: the artifact schema, deterministic replay + error handling,
  and the safety/escalation model are the load-bearing pieces.
- **Cut depth, not whole capabilities.** A thin-but-real version of every core requirement
  beats a polished subset. Minimal / stubbed-at-a-clean-seam / mocked is fine when
  intentional and documented.
- **Say what was cut and why**, and what would come next with more time.

## Deliverables (Section 6) — exact paths and headings required

> "Please use these exact paths and headings — we read a lot of submissions side by side."

1. **`/README.md`** covering:
   - how to set up and run it (including any keys/config needed, and how to run without
     live services if applicable)
   - a **demo path**: the exact command(s) to run the agent on a goal, then replay the
     resulting artifact
2. **`/REPORT.md`** (~1–3 pages) using these seven headings, in this order:
   1. Architecture — architecture, key decisions, trade-offs
   2. Artifact schema — the schema and why it is shaped that way
   3. Determinism & error handling — how replay is made deterministic; how runtime errors
      and exceptional states are detected and handled (and secondarily, UI drift)
   4. Heterogeneity & multi-tenant — extension to legacy web and desktop surfaces, and
      reuse across institutions running the same app (see 3.7)
   5. Escalation & handoff — how "stuck" is detected, how a human takes control of the live
      session, how control is handed back
   6. Safety — the guardrail model and its limits
   7. Cuts — what was deliberately left out, and what would be built next
3. **`/evidence/`** — a saved example artifact plus logs from **both** a discovery run and a
   replay run. Ideally including one replay that hits an error or exceptional state (bad
   input, not-found result, or an injected/simulated failure) to show detection and
   reporting. A short screen recording is welcome but optional.

## Evaluation criteria (Section 7) — roughly in this weighted order

1. **System design** — clear boundaries, sensible data models, good trade-offs, appropriate
   simplicity. *The artifact schema and replay contract are central.*
2. **Correctness of the core loop** — the agent actually completes a real goal; the artifact
   replays deterministically and verifies success.
3. **Robustness & error handling** — how replay detects and responds to runtime errors and
   exceptional states; how cleanly it separates expected business outcomes from recoverable
   conditions and hard failures; sound locator, wait, and checkpoint strategy.
4. **Human-in-the-loop escalation** — a real, well-reasoned mechanism to detect stuck, route
   an intervention request with context, transfer control of the live session, and resume.
   "Not just a TODO."
5. **Generalization to the real environment** — credible design story for heterogeneous
   surfaces and cross-tenant artifact reuse, without brittle assumptions or per-tenant
   rebuilds.
6. **Safety & data handling** — allowlist enforcement, treatment of risky and irreversible
   actions, redaction of regulated financial data.
7. **Code quality** — readable, reasonably typed and tested where it counts, easy to run.
8. **Communication** — the write-up makes reasoning, trade-offs, and cut lines clear.

Explicitly **not** rewarded: feature breadth, framework name-dropping, building scaling
infrastructure (queues, clusters, multi-tenant plumbing). Designing core abstractions so
they *could* scale is valuable; prematurely building that infrastructure is not. "A small,
correct, well-argued system is the goal."

## Optional stretch goals (Section 8) — at most one or two, only with a solid core

- Agent-facing capability interface: expose saved artifacts as a catalog of callable
  capabilities (tool/function-calling surface or an API endpoint) an agent could discover
  and invoke by name with typed args — and show one being invoked
- Code generation: emit a runnable test or automation snippet (page object, test file) from
  an artifact
- Confidence & approval: score artifacts by replay reliability; gate unattended replay on an
  approval state (draft → approved)
- Assisted fallback: on replay failure, a bounded, policy-checked LLM recovery for a
  *single* step (never open-ended), recorded as evidence
- Canonicalization / cross-tenant reuse: normalize concrete routes and values into
  parameterized patterns (`/item/12345` → `/item/:id`), and/or apply one artifact recorded
  on a "base" app to a second, slightly different variant, with per-variant overrides
- Multi-run stability: replay N times and report a stability/flakiness signal

## Ground rules (Section 9)

- AI-assisted development assumed and encouraged. **The flip side: we own everything
  submitted and must be able to explain and defend any part of it in detail.**
- Do not automate against sites where doing so would violate terms, harm the service, or
  require real credentials. Prefer sandboxes, demo sites, or a local app.
- Keep secrets out of the repo.
- Time-box it. "We're evaluating judgment, not endurance. If you stop early, document the
  rest as next steps."

## Submission (Section 11)

- Public GitHub repo: <https://github.com/PhongCT1105/interface_ai_assessment>
- Email the link to `assignments@interface.ai`
- Repo URL **on its own line**, from the address applied with (`phongct1105@gmail.com`),
  **do not send a zip**

## Glossary (Section 10)

> "Not knowing these is fine. Not looking them up is the problem."

| Term | Meaning |
| --- | --- |
| Computer use | An LLM operating an interface the way a person would — reading the screen or page, then clicking and typing, rather than calling an API |
| DOM | The browser's structured page representation. A "clean DOM" has meaningful elements and stable identifiers; legacy apps often don't |
| Accessibility tree | The parallel representation browsers and OSes expose for screen readers. Often more stable than raw markup, and available on desktop apps too |
| Locator / selector | How you tell automation which control to act on. The choice determines whether replay still works next month |
| Test ID | An attribute developers add so automation can find an element reliably. Legacy enterprise apps essentially never have them |
| Deterministic replay | Re-running a recorded flow the same way every time, with no model deciding anything. Same inputs, same steps, same outputs |
| Checkpoint | A condition asserted to confirm you actually reached the expected state, rather than assuming the click worked |
| Business outcome vs. failure | "No such member" is a legitimate answer the caller needs, not a crash. **Conflating the two is the most common design mistake here** |
| Tenant | One customer institution. Hundreds of them, many running the same vendor software configured differently |

## Traceability

Updated as implementation lands. Every row must be green (or an explicitly documented cut)
before submission.

| Req | Summary | Where | Status |
| --- | --- | --- | --- |
| 3.1 | Goal-driven LLM agent loop on a live UI | — | Not started |
| 3.2 | Typed, versioned capability artifact | — | Not started |
| 3.3 | Deterministic replay + 3-class result contract | — | Not started |
| 3.4 | Allowlist, risky-action policy, redaction | — | Not started |
| 3.5 | Structured logs + richer failure signal | — | Not started |
| 3.6 | Escalation, live-session takeover, resume | — | Not started |
| 3.7 | Heterogeneity + multi-tenant design story | `REPORT.md` §4 | Not started |
| 6.1 | `/README.md` setup + demo path | `README.md` | Skeleton |
| 6.2 | `/REPORT.md` seven headings | `REPORT.md` | Skeleton |
| 6.3 | `/evidence/` artifact + both run logs + error case | `evidence/` | Not started |
