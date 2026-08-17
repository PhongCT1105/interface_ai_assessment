# Research

Background research for the design decisions in [architecture.md](architecture.md) and
[scope.md](scope.md). Sources are linked inline. Conducted 2026-08-17; anything time-sensitive
is dated.

The purpose is twofold: understand the environment well enough that the design reflects it,
and check the proposed approach against what production systems and published research
actually do — rather than against what sounds plausible.

---

## 1. Who interface.ai is, and where this project sits

**The company.** interface.ai builds agentic AI for community banks and credit unions —
[over 100 financial institutions](https://interface.ai/why-interface/), with a platform they
describe as having learned from 500M+ conversations. It was bootstrapped until an
[$30M round led by Avataar Venture Partners](https://techcrunch.com/2024/10/22/interface-ai-raises-30m-to-help-banks-field-customer-requests)
($20M equity / $10M debt) — its first outside capital. CEO and co-founder Srinivas Njay.
Product lines: Voice AI (replacing IVR), Chat AI, **Employee AI / Sphere** (a staff-facing
copilot), [Smart Collections](https://interface.ai/newsroom/interface-ai-launches-smart-collections-an-agentic-multi-channel-collections-ai-agent-built-for-credit-unions-and-community-banks-to-increase-payments/),
all under an [agentic "BankGPT" platform](https://interface.ai/newsroom/interface-ai-unveils-industry-first-agentic-bankgpt-platform-that-moves-cx-from-convenience-to-outcomes/).

**The detail that matters most for this project.** Their public positioning is
*integration-first*: [40+ out-of-the-box integrations](https://interface.ai/solutions/employee-ai/)
spanning core processors (Symitar, Corelation, Fiserv, Jack Henry, CSI, Finxact, COCC,
CU\*Answers), origination (nCino, Blend), CRM (Salesforce), aggregation (Plaid, Yodlee), and
telephony (RingCentral, Twilio). Employee AI is explicitly about *executing actions* — moving
funds, updating account information — not just answering questions, and one credit union is
quoted saying Sphere replaces 14–15 applications for frontline staff.

So the brief is asking for the layer that covers **the long tail those 40+ integrations don't
reach** — and "replaces 14–15 applications" is a fair description of the operator workflow a
capability would automate.

**The industry framing sharpens the gap.** CCG Catalyst's
[sector spotlight on AI agents and connectors](https://www.ccgcatalyst.com/thought-leadership/research-snapshot/sector-spotlight-ai-agents-and-connectors-for-banks-and-credit-unions/)
maps 40+ vendors (interface.ai among the independent platforms) and concludes that MCP "has
become the de facto standard for granting agents access to systems and data" — while focusing
entirely on modern cores and APIs. It does not address systems that have no API at all. That
absence *is* the problem this assignment describes.

Two things to carry into the write-up:

- **Frame the artifact as a connector, not a script.** If the agent-facing boundary looks like
  a typed, discoverable capability, a no-API app becomes indistinguishable from one of the 40+
  real integrations. That makes the "agent-invocable capability catalog" stretch goal the most
  strategically aligned one, not just the flashiest.
- **"Every connector is an access grant."** That line from the CCG Catalyst piece is the exact
  governance argument for requirement 3.4, and it is stronger than a generic safety paragraph:
  the allowlist exists because a capability *is* a grant of authority over a bank system.

## 2. The surfaces this design has to survive

**Credit-union cores are a small set of products, widely shared.** Per
[Aerial's 2026 guide](https://joinaerial.com/blog/credit-union-core-systems) and
[CUCollaborate](https://www.cucollaborate.com/blog/top-core-processors-among-largest-us-credit-unions):
Fiserv leads on share (~25.9% across DNA, Portico and others); Jack Henry's **Symitar Episys**
is the most widely deployed platform (~699 credit unions, counting Jack Henry, Member Driven
Technologies and Synergent as service bureaus) and dominates institutions over $1B; **Corelation
KeyStone** is the fastest grower (~4.7%); FIS Miser trails. DNA, Episys and KeyStone are the
"tier-one" three.

One detail explains the brief's multi-tenant framing better than the brief itself does:
Corelation was founded in 2009 by John Landis, who had previously helped write Episys. The
market is a handful of closely related products, each deployed hundreds of times with different
configuration, branding, and version. "Many tenants running the same vendor product" isn't an
abstraction — it's the shape of the industry.

> Caveat: I verified market structure and vendor names, not UI internals. Claims about specific
> screens, terminal emulation, or markup in these products are **not** verified and should stay
> out of the write-up.

**How the RPA industry handles no-DOM surfaces.** UiPath's Citrix story is instructive: rather
than defaulting to image matching, they install a
[Remote Runtime component plus a client extension](https://docs.uipath.com/studio/standalone/2023.4/user-guide/about-automating-citrix-technologies)
so that "selectors are natively generated … without having to rely on OCR and image recognition."
Element-identity bridging first, pixels last. Their
[Object Repository](https://docs.uipath.com/studio/standalone/latest/user-guide/about-object-repository)
captures UI elements as reusable objects in a DOM-like store, shareable across projects — "build
a UI API for your application and share it with your team." That is the same idea as a base
capability with per-tenant overrides, from a decade of production RPA.

**Desktop, concretely.** [Microsoft UI Automation](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-uiautomationoverview)
exposes the whole desktop as a tree of elements with properties and control patterns, and
deliberately "masks differences across diverse control frameworks, including Win32, WPF, and
HTML." Legacy apps that only implement the older MSAA `IAccessible` are reachable through the
MSAA-to-UIA proxy. Elements carry runtime IDs. This is the concrete answer for the desktop half
of requirement 3.7 — the surface driver's normalized element view maps onto UIA almost directly,
which is the argument for shaping it like an accessibility tree rather than like a DOM.

## 3. Prior art: what already exists

### Tools

| Project | What it does | Relevance |
| --- | --- | --- |
| [**browser-use/workflow-use**](https://github.com/browser-use/workflow-use) | Self-described "RPA 2.0". A browser-extension recorder captures a demonstration, compiles it to a `.workflow.json`, auto-extracts variables from forms, and replays deterministically — with `run_with_no_ai()` for a model-free path and a fallback to the Browser Use agent when a step fails. | The nearest sibling to this assignment, and it validates the core premise commercially. Its own README calls the fallback "currently really bad" and the project not production-ready. |
| [**Stagehand**](https://github.com/browserbase/stagehand) (Browserbase) | `act` / `extract` / `observe` / `agent` primitives over Playwright, with [action caching](https://www.browserbase.com/blog/stagehand-caching): cache the *resolved selector* plus metadata, keyed on a SHA-256 of method + normalized URL + DOM-snapshot hash + project scope; before replaying, compare the current DOM fingerprint against the recorded one and only execute if it clears a safety threshold. Drift or miss → fall back to the LLM path and re-cache. ~80% speedup on repeat runs. | The best-validated design for replay-time validation, and directly improves my draft (see §5). Their stated principle — *"a wrong cached click is worse than a slow click"* — is the same instinct as "fail loudly, never drift," from a team that ships it. |
| [**Playwright**](https://qaskills.sh/blog/playwright-aria-snapshots-accessibility-tree-guide) | No formal self-healing engine, but `getByRole` / `getByLabel` / `getByText` resolve against the **accessibility tree** rather than CSS structure, so they survive class renames and DOM re-nesting. `page.ariaSnapshot()` and `toMatchAriaSnapshot` produce a YAML accessibility tree. | Validates accessibility-first targeting, and `ariaSnapshot()` is a ready-made primitive for both the `Target` vocabulary *and* per-step state assertions — meaningful build-cost reduction. |
| [**Skyvern**](https://github.com/Skyvern-AI/skyvern) | LLM + computer vision parsing the viewport each step, Playwright-compatible SDK, and a visualizer with 1:1 action-to-screenshot mapping. Caching ("memorize past actions and repeat them") is on the roadmap, not shipped. | Confirms the replay half is genuinely unsolved in the open ecosystem. The 1:1 action↔screenshot visualizer is a cheap, high-value evidence pattern worth borrowing. |
| [**Butter.dev's taxonomy**](https://blog.butter.dev/the-messy-world-of-deterministic-agents) | Nine approaches to deterministic agents: workflow builders (n8n) · context engineering (explicit and learned skills — Claude Skills, mem0, Cursor Memory) · **code generation** (ephemeral scripts, meta-tools, *script-agent fallback*, script generators — Browser Use, Browserbase Director, Forge, Sola, Cloudflare Code Mode) · response caching (an HTTP proxy layer) · LLM-layer changes (action models, RL). | Gives this project a name and a neighbourhood: **code generation → script-agent fallback**. Its central tradeoff — approaches needing advance task knowledge are more deterministic, discovery-based ones more flexible — is precisely what discover-once/replay-many resolves. |
| [Cloudflare Browser Run](https://developers.cloudflare.com/browser-run/features/human-in-the-loop/) · [Browserbase Live View](https://www.browserbase.com/templates/agent-with-human-in-loop) | Human-in-the-loop is a *productized* feature, with a consistent pattern: keep the session alive (`auto_close=false`) with a stable session ID over CDP; the script disconnects; a human attaches to a Live View URL and clears the blocker (login, MFA, CAPTCHA, sensitive input); the human disconnects; the script reconnects to the *same* session. **Only one client controls at a time.** noVNC covers the desktop equivalent. | This is requirement 3.6's control-transfer model, already validated in production — and "only one client at a time" is independent confirmation of the single-explicit-owner session design. |

### Research

- [**Agent Workflow Memory**](https://arxiv.org/abs/2409.07429) (ICML 2025) — induces reusable
  *workflows* from experience, either offline from demonstrations or **online from
  self-generated successful trajectories**, then selectively feeds them back to guide later
  generation. Evaluated on Mind2Web and WebArena. This is the academic name for step 2 of the
  assignment, and the online variant is exactly "compile a capability from a successful run."
- [**PreAct**](https://arxiv.org/pdf/2606.17929) — computer-using agents that get faster on
  repeated tasks via workflow induction into **parameterized** workflows carrying conditional
  branches and input/output mappings, with **verification gates** confirming a cached workflow
  still applies and **graceful degradation** to normal execution on mismatch. Reported across
  WebArena, AndroidWorld and OSWorld. Independent validation of three specific choices:
  parameterization at compile time, a verification gate before reuse, and degrade-rather-than-guess.
- *Get Experience from Practice: LLM Agents with Record & Replay*
  ([arXiv 2505.17716](https://arxiv.org/pdf/2505.17716)) — **not read**; PDF text extraction
  failed. Listed so it isn't mistaken for a source. Worth a second attempt before the write-up.
- **Grounding.** Set-of-Marks prompting (numbered bounding boxes drawn on the screenshot)
  measurably improves grounding over plain text; the common hybrid pairs those marks with
  accessibility metadata per element (index, tag, name, text). The accessibility tree is
  effective but "may not be well-supported across all software and can impose an additional
  inference burden … due to its token volume" — per the
  [OS Agents survey](https://arxiv.org/pdf/2508.04482). Benchmarks in this line:
  [OSWorld](https://arxiv.org/pdf/2404.07972), [VisualWebArena](https://arxiv.org/pdf/2401.13649),
  [OmniParser](https://arxiv.org/pdf/2408.00203).

### Prior submissions of this assignment

Searched GitHub and the open web for public submissions of this specific brief
(`assignments@interface.ai`, "Computer-Use Automation System", the assignment title). **Nothing
found.** Either candidates aren't publishing, or the repos aren't discoverable by title. So
there is no house style to match and no bar to read off other submissions — the evaluation
criteria in the brief are the only scoring signal available, and they are unusually explicit.

## 4. Anthropic platform facts relevant to the build

From the bundled `claude-api` reference (current as of this session), not from memory:

| Topic | Fact |
| --- | --- |
| Model | `claude-opus-5` — $5 / $25 per MTok, 1M context (default and max), 128K max output. Adaptive thinking is **on by default**; `effort` runs `low`→`max`. |
| Effort | Start `xhigh` for agentic/coding work, `high` elsewhere — then sweep down, because `low`/`medium` are unusually strong on this model. At `xhigh`/`max`, set `max_tokens` ≥ 64K. |
| Computer use | Client-executed tool, type `computer_20251124`, beta header `computer-use-2025-11-24`. |
| Vision | High-resolution tier: 2576 px on the long edge, up to ~4784 image tokens per image, and **coordinates map 1:1 to pixels** (no scale-factor math). For computer use, **1080p screenshots** balance performance and cost; 720p or 1366×768 are the cheaper options. |
| Closed action vocabulary | Strict tool use (`strict: true` + `additionalProperties: false` + `required`) guarantees the tool input validates exactly — the right mechanism for a small closed action alphabet. |
| Compile step | Structured outputs via `output_config.format` (JSON Schema) for emitting the artifact. |
| Prompt caching | 512-token minimum on Opus 5. Writes cost ~1.25×, reads ~0.1×. Render order is `tools` → `system` → `messages`, so keep the stable prefix first and the volatile screenshot last — material for a multi-step perception loop. |
| Cheap sub-steps | Haiku 4.5 at $1 / $5 per MTok, if any classification step wants a smaller model. |

Cost note: the brief states a single successful discovery run "is not an expensive thing to
produce," and these numbers agree. Cost is not a design constraint here; it is a talking point
about why replay exists.

## 5. Direction check

The user asked specifically to verify the direction rather than trust it. Each row is a decision
from [architecture.md](architecture.md) against independent corroboration.

| Decision | Corroboration | Verdict |
| --- | --- | --- |
| Accessibility-tree-shaped normalization, not DOM-shaped | Playwright's role/label/text locators resolve against the a11y tree and survive class and nesting changes; Windows UIA exposes the same shape for desktop; UiPath bridges to native element identity rather than OCR | **Confirmed** |
| Coordinates as diagnostics, never targeting | Same sources; UiPath treats image matching as the last resort | **Confirmed** |
| Capability as a typed contract with declared inputs/outputs | workflow-use auto-extracts form variables and exposes workflows as callable tools; PreAct's parameterized workflows | **Confirmed** |
| Parameterization decided at compile time | PreAct's input/output mappings are part of induction, not replay | **Confirmed** |
| Replay must not guess — degrade and report | PreAct's graceful degradation; Stagehand's "a wrong cached click is worse than a slow click" | **Confirmed** |
| Per-step verification | PreAct's verification gates; Stagehand's pre-execution fingerprint comparison | **Confirmed, and improvable — see below** |
| Single explicit session owner across handoff | Cloudflare / Browserbase: one controlling client at a time, human attaches to the same live session, script reconnects | **Confirmed** |
| Base artifact + per-tenant overrides | UiPath Object Repository — shared UI taxonomy across projects | **Confirmed** |
| Compile a capability from a successful LLM trajectory | Agent Workflow Memory's online induction (ICML 2025) | **Confirmed** |
| Policy chokepoint beneath both paths | No direct corroboration in the tools surveyed — none of them have a policy layer at all | **Unvalidated, and likely a differentiator** rather than a mistake: none of these tools operate inside a bank |

**Three substantive updates to the plan.**

1. **Add a pre-step precondition check, not just a post-step assertion.** The draft had each step
   carrying an `expect` verified *after* acting. Stagehand validates *before* acting, comparing a
   page fingerprint against the recorded one and refusing to execute if it hasn't cleared a
   threshold. That is strictly better for a system that must not misfire inside a bank: catching
   "the screen isn't what this step was recorded against" before a click beats catching it after.
   Each step should carry both — a `precondition` and an `expect`.
2. **`page.ariaSnapshot()` can serve as both the target vocabulary and the checkpoint mechanism**,
   which lowers the cost of the most load-bearing part of the build.
3. **The target-app decision has a better third option than the two I offered.** See below.

**Revisiting the target application.** [ParaBank](https://hub.docker.com/r/parasoft/parabank)
(Parasoft's demo banking app) is a genuine candidate I'd missed the details of: `docker pull
parasoft/parabank`, JSP-era markup, a real multi-step banking flow, and an
[admin page](https://parabank.parasoft.com/parabank/admin.htm) exposing data-access mode
(SOAP/REST/JDBC), service endpoints, database initialize/clean, and loan-processor settings. But
I checked the admin page specifically for fault injection and **there is none** — no delay,
error, or downtime controls. So it fails the requirement that drove decision 0006.

That suggests a third option that is better than either original:

> **ParaBank behind a small fault-injection reverse proxy.** A real third-party legacy-styled
> banking app, plus faults on demand — the proxy returns 500s, injects latency, drops the session
> cookie, and splices in an unexpected interstitial. A second "tenant" comes nearly free by having
> the proxy rewrite branding and labels.

This is worth serious consideration because it directly answers the stacked-deck objection: the
app under automation was written by someone else, for other purposes, before this project existed.
The cost is less control over the hostility of the markup, and one more moving part in the demo
path. Recorded as an open question rather than a decision — it changes decision 0006 and that is
the user's call.

## 6. Methods, by component

Options considered and the current recommendation. Where a choice is already corroborated above,
it is marked ✓.

### Perception (3.1)

| Option | Trade-off |
| --- | --- |
| Accessibility tree only | Precise, cheap to target from, portable to desktop; blind to anything visual, and token-heavy on large pages |
| Screenshot + coordinates only | Works on any surface including Citrix and images-of-apps; coordinates don't survive replay, so it fights determinism |
| **Hybrid ✓** | A11y tree for targeting and assertions, screenshot for judgment and evidence. Optional Set-of-Marks overlay when the tree is ambiguous — measurably better grounding, at token cost |

Recommendation: hybrid, with the screenshot sent at 1080p and the a11y snapshot trimmed to the
interactive subtree. Keep the stable prompt prefix cached and the screenshot last.

### Targeting and determinism (3.3)

A `Target` resolves through an ordered fallback chain, each rung recorded with its rationale:

1. Role + accessible name (Playwright `getByRole`) — the durable primary
2. Accessible name within a named container (disambiguates repeated controls in tables)
3. Visible text / label association
4. Ordinal within a container — brittle, recorded as a last resort
5. Structural path — diagnostic only; if replay reaches this rung, report rather than act

Plus: condition-based waits only, never sleeps; a pre-step precondition fingerprint; a post-step
assertion; and the model client structurally unreachable from the replay module.

### Error taxonomy (3.3) — the highest-value piece

The brief's glossary names conflating business outcomes with failures as the most common mistake,
and no surveyed tool models this at all. Concretely:

- **Signals** are declarative predicates over observed state (text present, element present, URL
  matched, HTTP status observed) — data in the artifact, not code in the engine.
- Each signal maps to one of: `businessOutcome` (named, returned to the caller as a legitimate
  answer), `recover` (a bounded action — dismiss, wait, retry-once — then continue), or `fail`
  (stop with step index, target, expected, observed, evidence refs).
- Anything unmatched is a hard failure by default, never an assumption. An unmatched exceptional
  state is also a natural escalation trigger.

### Escalation and handoff (3.6)

Follow the productized pattern: long-lived session, single explicit owner, human attaches to the
same live session, human's actions captured, resume on signal. Options for the operator surface,
cheapest first: a CLI prompt against a headed browser (adequate and honest); a tiny local page
with Take control / Resume buttons plus the intervention payload; Playwright's inspector; a
CDP-attached live view. Capturing what the human did: an injected recorder script listening for
click/input events (the technique Playwright's own codegen uses) is the real mechanism; a
before/after state diff is the cheap fallback.

### Safety (3.4)

Allowlist of origins/routes and action types at the single policy chokepoint, covering discovery
*and* replay. Risk classes `safe` / `confirm` / `blocked`, defaulting conservative. Redaction at
the persistence boundary — sensitive-marked inputs plus a small credential/PII pattern set —
enforced where writing happens so no call site can forget. Frame it with "every connector is an
access grant."

### Multi-tenant (3.7, design only)

Base artifact keyed on `app.product` + a per-tenant override layer that may replace targets and
add tenant-specific exceptional states, but not change the step sequence or the contract
(inputs/outputs/checkpoint stay fixed — that's what makes the capability the *same* capability).
Drift detection falls out of the error taxonomy: a precondition or checkpoint failure on a
specific tenant, aggregated, *is* the drift signal — plus canary replays. This mirrors UiPath's
Object Repository, which is worth citing as precedent rather than presenting as novel.

### Testing (code-quality criterion)

Where it counts: schema validation round-trips; the replay interpreter against a fake driver
(fast, no browser, covers the result-contract matrix); error/outcome classification; redaction;
and a boundary test asserting the replay module cannot import a model client — the test that
makes the headline claim mechanical rather than rhetorical.

## 7. Tools, data, and inputs

| Need | Source |
| --- | --- |
| Model access | Anthropic API key, or an `ant auth login` profile (the SDKs resolve it automatically) |
| UI driver | Playwright — accessibility snapshots, tracing, video, headed mode for handoff |
| Target app | `docker pull parasoft/parabank`, optionally behind a fault-injection proxy; or a purpose-built hostile app |
| Fake member data | Faker. **Never real credentials or real PII** — an explicit ground rule |
| Redaction | Regex pattern set; Microsoft Presidio if a heavier approach is wanted (probably overkill) |
| Evidence | JSONL event stream + Playwright trace/video; consider Skyvern's 1:1 action↔screenshot pairing |
| Framing references | WebArena, Mind2Web, OSWorld, VisualWebArena — for citing the problem class, not for running benchmarks |

## 8. Open questions

- **Target app**: keep decision 0006 (purpose-built hostile app) or switch to ParaBank + fault
  proxy? Affects the stacked-deck defense and the demo path.
- **Stack**: still open (decision 0005). Nothing in this research forces the choice, though
  Playwright's accessibility snapshot API is the strongest single argument for the language with
  the most idiomatic Playwright binding.
- **Set-of-Marks overlay**: include in discovery, or keep perception to a11y tree + plain
  screenshot? Better grounding versus token cost and build time.
- Re-attempt the record-and-replay paper (arXiv 2505.17716) before writing `REPORT.md` §1.
