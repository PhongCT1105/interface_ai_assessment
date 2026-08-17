# Computer-Use Automation System

Assignment A for the interface.ai Software Engineer application.

## The problem

Give an AI agent hands. Many real systems have a UI but no API. This project builds the
layer in between:

1. **Explore** — an LLM works out how to complete a task inside a real UI, by looking at
   the screen and acting on it.
2. **Record** — the successful run is captured as a reusable *capability*: a concrete,
   parameterized sequence of actions plus the checks that prove each step landed.
3. **Replay** — that capability runs again deterministically, with **no model in the
   loop**. Same inputs, same actions, same result, no tokens spent.

The interesting part is step 3. Anything can drive a browser once with an LLM; the value
is turning one successful exploration into a repeatable, auditable automation that does
not depend on a model being available, cheap, or in a good mood.

## Status

Repository initialized — documentation first, implementation next. See
[docs/assignment-brief.md](docs/assignment-brief.md) for the requirements of record and
[docs/architecture.md](docs/architecture.md) for the working design.

| Piece | State |
| --- | --- |
| Assignment brief (email) | Captured |
| Assignment brief (PDF) | **Pending** — needs to be added to `docs/` |
| Architecture sketch | Draft |
| Stack decision | Open |
| Explore / record / replay implementation | Not started |
| Demo task + evaluation | Not started |

## Repository layout

```
docs/
  assignment-brief.md   requirements of record, submission checklist
  architecture.md       working design: explore -> record -> replay
  decisions.md          decision log (what was chosen and why)
README.md
```

Source directories are added as the implementation lands, so the tree stays honest about
what exists.

## Getting started

Not yet runnable. Setup and run instructions land here with the first working slice,
along with the demo task used to show explore-then-replay end to end.

## Submission

- Public GitHub repo: <https://github.com/PhongCT1105/interface_ai_assessment>
- Emailed to `assignments@interface.ai` when it represents best work (no deadline)
