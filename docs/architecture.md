# Architecture (proposed)

Status: **proposed, pre-implementation.** Written after reading the full brief. The layering
and the contracts below are deliberate positions; the stack choices at the bottom are still
open. Every position here should end up defended in `REPORT.md` §1.

Requirement references are to [assignment-brief.md](assignment-brief.md).

## The shape of the problem

The brief's environment section drives almost every decision, so the design starts there
rather than from the phases:

| Environment fact | What it forces |
| --- | --- |
| UIs are stable, but runtime errors are real | Effort goes into an **error taxonomy and result contract**, not into self-healing selectors |
| No clean DOM, no test IDs, sometimes a desktop app | Perception and action must sit behind a **surface driver interface**; the artifact cannot embed CSS selectors as its only targeting story |
| Same vendor product across many tenants | Artifacts need **identity separate from instance** — a base artifact plus per-tenant overrides, not a copy per tenant |
| Regulated financial data | Redaction is a property of the **logging/persistence layer**, not something remembered at each call site |
| A human must be able to take over the live session | The session is a **first-class, long-lived object** with an owner, not a local variable inside a run function |

## Layers

```
        ┌─────────────────────────────────────────────────────────────┐
        │  Capability catalog          (what an AI agent can call)    │
        │  name, version, typed inputs/outputs, approval state        │
        └───────────────┬─────────────────────────┬───────────────────┘
                        │ record                  │ invoke
        ┌───────────────▼──────────┐   ┌──────────▼───────────────────┐
        │  DISCOVERY               │   │  REPLAY                      │
        │  LLM observe→decide→act  │   │  interpreter over artifact    │
        │  + trace → compile       │   │  NO model client, structurally│
        └───────────────┬──────────┘   └──────────┬───────────────────┘
                        │                          │
        ┌───────────────▼──────────────────────────▼───────────────────┐
        │  POLICY   allowlist · action risk class · redaction          │
        │  every action passes through here, both paths                │
        └───────────────────────────┬──────────────────────────────────┘
                                    │
        ┌───────────────────────────▼──────────────────────────────────┐
        │  SESSION      owner: automation | human | none               │
        │  pause / cede / resume · survives handoff                    │
        └───────────────────────────┬──────────────────────────────────┘
                                    │
        ┌───────────────────────────▼──────────────────────────────────┐
        │  SURFACE DRIVER   observe() → state · act(action) → result   │
        │  web (impl) │ legacy web (design) │ desktop (design)         │
        └──────────────────────────────────────────────────────────────┘

        EVIDENCE writes from every layer: structured events + failure snapshots
```

Two things are load-bearing about this picture:

- **Policy sits below both discovery and replay.** Guardrails that only wrap the LLM loop
  are theatre — replay executes the same actions against the same bank. One chokepoint,
  both paths.
- **Session sits below policy.** Handoff (3.6) requires pausing an in-flight run and letting
  a human drive *the same* session. That is impossible if the session is owned by the run;
  it has to be the other way round.

## Surface driver: the seam (3.7)

The single most important abstraction for the heterogeneity story. Everything above it is
surface-independent; everything surface-specific lives below it.

```
observe()            → a normalized view of current state: an element/control list
                       (role, name, value, enabled, bounds) + optional screenshot
act(action)          → apply one action from a small closed vocabulary
resolve(target)      → find the control a Target descriptor refers to, or fail loudly
```

- The **action vocabulary must be small and closed** — click, type, press key, select,
  scroll, navigate, wait-for, read. A small alphabet is precisely what makes a recorded
  flow replayable and portable; every driver-specific escape hatch added here is a step
  the artifact can no longer express on another surface.
- The normalized element view is deliberately close to an **accessibility tree**, because
  that is the one representation available across all three target surfaces: browsers
  expose it, and so do desktop OSes. Choosing DOM-shaped normalization would work today
  and paint us into a corner exactly where the brief warns about it.
  *Precedent:* Playwright's `getByRole` / `getByLabel` / `getByText` resolve against the
  accessibility tree and survive class renames and DOM re-nesting; Windows UI Automation
  exposes the same tree shape uniformly across Win32, WPF and HTML, with an MSAA proxy for
  legacy apps; UiPath's Citrix support installs a runtime bridge specifically so selectors
  resolve against native element identity *instead of* OCR. Three independent surfaces, one
  answer. See [research.md](research.md) §2–3.
- Coordinates are captured but treated as **diagnostics, not targeting** — they are the one
  thing guaranteed not to survive a window resize or a differently-branded tenant.
- On a browser surface, Playwright's `page.ariaSnapshot()` supplies this view directly, which
  means the same primitive serves as the `Target` vocabulary *and* as the state assertion
  mechanism. That is a build-cost reduction on the most load-bearing part of the system.

## Capability artifact (3.2)

The focal point of the evaluation, so the schema gets designed before the code. It is a
**contract**, not a recording. Sketch of the intended shape:

```
capability
  id, name, version, schemaVersion
  app        { product, surfaceKind, entryPoint }   ← identity, not instance
  inputs     [ { name, type, required, sensitive } ]
  outputs    [ { name, type, source } ]
  steps      [ step ]
  checkpoint { assertion }                          ← overall success condition
  provenance { discoveredAt, model, runId, humanEdits }
  approval   draft | approved

step
  action        one of the closed vocabulary
  target        { primary: Target, fallbacks: [Target] }
  value         literal | { param: inputName }       ← parameterization lives here
  precondition  assertion                            ← verified BEFORE acting
  expect        assertion                            ← verified AFTER acting
  onCondition   [ { when: signal, then: outcome|recover } ]  ← known exceptional states
  risk          safe | confirm | blocked

Target         role + accessible name (primary), text, ordinal-within-container,
               structural path (last resort); each with the reasoning recorded
```

Design commitments worth arguing for in the write-up:

- **Every step is verified on both sides — `precondition` before, `expect` after.** Without the
  post-step assertion, replay drifts silently: it clicks into the void and only notices at the
  final checkpoint, by which point the debuggable information is gone. Without the pre-step
  check, it acts on a screen it was never recorded against — which inside a bank is the worse
  of the two failures. Together they are what makes a failure report *which step, expected
  what, observed what* (3.3). Browserbase reached the same conclusion for cached Stagehand
  actions, validating a page fingerprint against a safety threshold before executing; see
  [research.md](research.md) §3.
- **Parameterization is explicit, not inferred at replay time.** A value in a step is either
  a literal or a named reference to a declared input. The recorder decides which, at
  compile time, using knowledge of what came from the goal versus what came from the page.
- **Known exceptional states are part of the artifact**, not hardcoded in the engine. "This
  screen can show a validation error", "this flow can hit an interstitial" is knowledge
  about the *app*, so it belongs with the app's capability where a human reviewer can see
  and edit it.
- **`app.product` is identity separate from tenant instance.** This is the hook for
  cross-tenant reuse (3.7): one base artifact per vendor product, with per-tenant override
  layers, rather than N recordings. An override layer may **replace targets** and **add
  tenant-specific exceptional states**; it may *not* change the step sequence or the contract
  — inputs, outputs and checkpoint stay fixed, because that is what makes it the *same*
  capability rather than a different one wearing the same name.
  *Precedent:* UiPath's Object Repository has done shared, reusable UI-element taxonomies
  across projects for a decade — worth citing as prior art rather than presenting as novel.
- **Human-readable and diff-reviewable.** The brief requires a human reviewer *and* a
  calling agent to understand it. That rules out an opaque binary trace and argues for
  plain declarative data with the LLM's transcript kept separately as evidence.

## Discovery (3.1)

Goal + target in; a validated trace out. Per turn: observe, propose one action from the
closed vocabulary, check it against policy, apply, observe the result, record what was
tried and why. Terminates on goal-check pass, step budget, timeout, or a dead-end that
triggers escalation (3.6).

Two guards worth stating up front:

- **The model is not the sole judge of its own success.** A run that cannot be verified
  against an explicit checkpoint must not be compiled into a capability — otherwise the
  catalog fills with plausible automations that never worked.
- **The transcript is evidence, not the artifact.** Compilation prunes dead ends, retries,
  and backtracks; only the path that actually reached the checkpoint survives.

Mechanics settled by the research pass ([research.md](research.md) §4):

| Concern | Setting |
| --- | --- |
| Model | `claude-opus-5` — 1M context, adaptive thinking on by default |
| Effort | Start `xhigh` (agentic work), then sweep down; `low`/`medium` are unusually strong on this model. `max_tokens` ≥ 64K at `xhigh` |
| Action vocabulary | **Strict tool use** (`strict: true`, `additionalProperties: false`, `required`) — the API then guarantees the action JSON validates, which is exactly what a small closed alphabet wants |
| Compile step | Structured outputs (`output_config.format` + JSON Schema) so the artifact is emitted validated rather than parsed hopefully |
| Perception payload | Accessibility snapshot trimmed to the interactive subtree, plus a screenshot at 1080p (Anthropic's recommended performance/cost point for computer use) |
| Prompt caching | Render order is `tools` → `system` → `messages`: keep the stable prefix cached (512-token minimum on Opus 5, reads ~0.1×) and put the volatile screenshot last. Material across a multi-step loop |

## Replay (3.3)

A straight interpreter over the artifact: check `precondition`, resolve target, apply action,
assert `expect`, handle any matched `onCondition`, continue; return declared outputs on
checkpoint pass.

**Target resolution is an ordered chain**, each rung recorded at compile time with its
rationale so a reviewer can see why a step targets what it targets:

1. Role + accessible name — the durable primary
2. Accessible name **within a named container** — disambiguates repeated controls in tables
3. Visible text / label association
4. Ordinal within a container — brittle, recorded explicitly as a last resort
5. Structural path — **diagnostic only**. If resolution reaches this rung, replay reports
   rather than acts; a step that can only be found structurally is a step whose recording has
   gone stale, and clicking anyway is how automation does damage quietly.

- **"No model in the loop" must be structural.** Replay lives in a module that cannot import
  a model client, enforced by build/lint boundary rather than by discipline — so the claim
  is a property of the code and demonstrable to a reviewer, which is what they will press
  on.
- **Three-class result contract**, mirroring the brief exactly, because conflating these is
  named in the glossary as the most common mistake:
  - `success` + typed outputs
  - `businessOutcome` — a legitimate answer the caller needs (`no such member`), carrying
    which condition matched
  - `failure` — hard stop, with step index, target, expected, observed, and evidence refs
  - plus **recovered** conditions logged but not surfaced as failure (dismissed
    interstitial, retried transient load)
- **Waits are conditions, never sleeps.** `wait-for` an assertion; a fixed sleep is both
  slower and less deterministic than the thing it replaces.
- **The model-free claim gets a test, not a paragraph.** A boundary test asserts that the
  replay module cannot import a model client. That turns the headline property from a promise
  into something mechanically checked — the claim a reviewer is most likely to press on is
  then the one with a test next to it.

## Escalation & handoff (3.6)

Sequence to implement, thin but real:

```
detect stuck  →  raise intervention request (capability, step, state+screenshot, why)
              →  session owner := none, automation pauses
              →  human takes control of the SAME live session, acts
              →  human signals resume; their actions are captured
              →  session owner := automation, run continues or completes
```

The operator console is explicitly mockable; the **control-transfer model** is not. The
things to get right are: a single explicit owner at all times, the session surviving the
transfer, and the human's actions landing in evidence alongside the automation's.

Concretely, in cheapest-first order:

- **Live session**: a headed browser kept alive independently of the run. Automation pauses;
  the human drives the same window.
- **Owner flag**: `automation | human | none`, checked at the policy chokepoint. One owner at
  all times, no implicit concurrency.
- **Capturing the human's actions**: an injected recorder script listening for click/input
  events — the same technique Playwright's own codegen uses. Cheap fallback if that proves
  fiddly: a before/after state diff plus an operator note.
- **Operator surface**: a CLI prompt is adequate and honest. A tiny local page with
  *Take control* / *Resume* buttons and the intervention payload rendered is nicer if time allows.

*Precedent:* Cloudflare Browser Run and Browserbase Live View both productize exactly this
sequence — session kept alive over CDP with a stable ID, script disconnects, human attaches to
a live view and clears the blocker (login, MFA, CAPTCHA), human disconnects, script reconnects
to the *same* session — and both enforce **one controlling client at a time**. That is
independent confirmation of the single-owner model, not a coincidence.

## Safety (3.4)

The framing worth using, borrowed from the industry analysis in [research.md](research.md) §1:
**every connector is an access grant.** A capability is not a convenience script — it is
delegated authority over a bank system, and the allowlist is what bounds that grant. That is a
better argument than a generic safety paragraph, and it is the reviewers' own industry's phrasing.

- Allowlist of permitted origins/routes and permitted action types, checked at the policy
  chokepoint for **both** discovery and replay.
- Every action classified `safe` / `confirm` / `blocked`. Reads and navigation within the
  allowlist are safe; anything that writes, submits, or moves money is not. Default for
  unclassified is conservative, not permissive.
- Redaction at the persistence boundary: values from inputs marked `sensitive`, and
  anything matching credential/PII patterns, never reach artifacts or logs. Enforced where
  writing happens so no call site can forget.

## Evidence (3.5)

Structured event stream per run (`runId`, step, action, target, decision rationale, result)
plus a richer snapshot on failure. Discovery additionally keeps the model transcript. Both
runs land under `/evidence/` as required by the deliverables, including at least one replay
that hits an exceptional state.

## Open decisions

To be closed before implementation and recorded in [decisions.md](decisions.md).

| Decision | Options | Status |
| --- | --- | --- |
| Perception | Accessibility tree, screenshot, hybrid | **Decided:** hybrid — a11y tree for targeting and assertions, screenshot for judgment and evidence (corroborated, 0008) |
| Surface driver (web impl) | Playwright, CDP, OS-level | **Effectively decided:** Playwright, for accessibility-tree access and `ariaSnapshot()` |
| Model | Claude | **Decided:** `claude-opus-5` |
| Language / runtime | TypeScript, Python | **Open** (0005) — the research did not force it; the tiebreak is whichever has the more idiomatic Playwright binding for you |
| Proxy target app | Locally built hostile app, ParaBank + fault proxy | **Reopened** (0006 amended) — leaning ParaBank behind a fault-injection proxy |
| Artifact serialization | JSON, YAML, generated code | Open; leaning JSON with a schema, YAML only if review ergonomics win |
| Set-of-Marks overlay in discovery | Yes, no | Open — better grounding versus token cost and build time |
| Process architecture | Single process + CLI, services | Leaning single process; the brief rewards justified simplicity |

Scope, depth budget, and build order are in [scope.md](scope.md). The stack deferral is safe
because everything above is defined in terms of contracts and seams rather than libraries.

## Principles

- **Replay is provably model-free.** Structural separation over good intentions.
- **A capability is a contract**: declared inputs, declared outputs, verifiable checkpoint.
  If it cannot be verified, it does not get recorded.
- **Business outcomes are results, not errors.** The distinction is designed in, not patched
  on.
- **Fail loudly, never drift.** A capability that stops with a precise reason is useful; one
  that improvises inside a bank is dangerous.
- **Thin-but-real everywhere beats polished-and-partial.** The brief says so explicitly, and
  it is also the honest reading of a system whose value is end-to-end.
