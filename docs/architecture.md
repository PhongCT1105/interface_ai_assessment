# Architecture (draft)

Status: **draft.** Written before the PDF brief was available and before any code exists.
The three-phase split comes straight from the assignment description; the specifics below
are a starting position, not a commitment. Revise freely — and record the revisions in
[decisions.md](decisions.md).

## The three phases

```
                  ┌──────────────┐
   task in        │   EXPLORE    │   LLM sees the screen, decides the next action,
   natural   ───► │  (model in   │   acts, observes the result, repeats until the
   language       │   the loop)  │   task goal is satisfied.
                  └──────┬───────┘
                         │ successful trace
                         ▼
                  ┌──────────────┐
                  │    RECORD    │   Compress the trace into a capability: ordered
                  │  (compile)   │   steps, stable targets, parameters, assertions.
                  └──────┬───────┘
                         │ capability artifact (versioned, on disk)
                         ▼
                  ┌──────────────┐
                  │    REPLAY    │   Execute the capability directly. No model call.
                  │ (no model)   │   Assertions decide pass/fail.
                  └──────────────┘
```

## Explore

The agent gets a task description and a handle on the UI. Each turn it observes state,
picks one action, and applies it. The loop ends when a goal check passes or a step budget
is exhausted.

Open questions:

- **Perception.** Screenshots (pure vision, works anywhere, imprecise) versus an
  accessibility/DOM tree (precise, structured, only for instrumented surfaces) versus
  both. Both is likely right: the tree for targeting, the screenshot for judgment.
- **Action vocabulary.** Keep it small and total — click, type, key, scroll, wait,
  navigate, extract. A small alphabet is what makes replay feasible.
- **Goal checking.** The agent should not be the only judge of its own success; a run that
  cannot be verified should not be recorded as a capability.

## Record

The compile step is where the real design work is. A raw trace is not a capability — it is
one path through one session, full of accidents. Turning it into something reusable means:

- **Stable targeting.** Coordinates from the exploration run are worthless on replay.
  Targets need to resolve by something durable (role + accessible name, test ids, text),
  with the recorded coordinates kept only as a diagnostic.
- **Parameterization.** Values the task varies on (search terms, form fields, IDs) become
  inputs. Values that are incidental to the UI stay fixed. Deciding which is which is the
  crux — the recorder needs to know what came from the task versus what came from the
  page.
- **Assertions.** Each step records what "it worked" looked like, so replay can fail loudly
  instead of drifting silently.
- **Pruning.** Dead ends, retries, and backtracking in the exploration trace get dropped;
  only the path that actually led to success survives.

The artifact should be plain, versioned, human-readable data — reviewable in a diff and
editable by hand.

## Replay

A straight interpreter over the capability artifact. Reads steps, resolves targets,
performs actions, checks assertions, stops on first failure. It must have no dependency on
any model client — ideally enforced structurally, so that "no model in the loop" is a
property of the code rather than a claim in a README.

Failure is a first-class outcome: when the UI has changed enough that a capability no
longer applies, replay should say so precisely (which step, which target, what was
expected) rather than guessing. Whether a failed replay can escalate back to exploration
is a design question worth answering deliberately — it is a genuinely useful feature and
also the easiest way to accidentally put a model back in the replay path.

## Open decisions

| Decision | Options | Status |
| --- | --- | --- |
| Language / runtime | TypeScript, Python | Open |
| UI driver | Playwright, CDP, OS-level computer use | Open |
| Perception | Screenshot, accessibility tree, hybrid | Leaning hybrid |
| Capability format | JSON, YAML, generated code | Open |
| Target UI for the demo | — | Open, must genuinely have no API |
| Model | Claude (see `claude-api` guidance before wiring) | Open |

## Design principles

- **Replay must be provably model-free.** Structural separation over good intentions.
- **A capability is a contract.** It states its inputs, its steps, and how each step is
  verified. If it cannot be verified, it cannot be recorded.
- **Fail loudly, never drift.** A broken capability that stops is useful; one that
  improvises is dangerous.
- **Every choice defensible.** The brief asks for it explicitly.
