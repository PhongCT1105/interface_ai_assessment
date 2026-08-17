# Computer-Use Automation System

Take-home project for interface.ai — the backend integration layer that gives an AI agent
hands inside applications that have no API.

**An LLM discovers** how to complete a goal by driving a real UI. **The successful run is
recorded** as a typed, versioned, parameterized capability. **That capability replays
deterministically** — with no model in the decision loop — so an agent can invoke it in
production reliably and cheaply.

The target environment is back-office software at banks and credit unions: stable UIs, but
real runtime errors; often legacy surfaces with no clean DOM or test IDs; hundreds of
tenants running the same vendor product configured differently.

## Status

Design phase. Requirements captured, architecture proposed, no implementation yet.

| Piece | State |
| --- | --- |
| Requirements of record | Complete — [docs/assignment-brief.md](docs/assignment-brief.md) |
| Architecture | Proposed — [docs/architecture.md](docs/architecture.md) |
| Scope, depth budget, build order | Defined — [docs/scope.md](docs/scope.md) |
| Field + domain research | Complete — [docs/research.md](docs/research.md) |
| Target application | Reopened — hostile app, or ParaBank behind a fault proxy |
| Stack / computer-use technology | Open (perception strategy settled) |
| Agent loop (3.1) | Not started |
| Capability artifact (3.2) | Not started |
| Deterministic replay (3.3) | Not started |
| Safety guardrails (3.4) | Not started |
| Evidence / observability (3.5) | Not started |
| Escalation & handoff (3.6) | Not started |
| `REPORT.md` | Skeleton |
| `evidence/` | Empty |

Requirement numbers refer to Section 3 of the brief; the traceability table at the bottom of
[docs/assignment-brief.md](docs/assignment-brief.md) is the authoritative progress view.

## Setup

Not yet runnable. Prerequisites, install steps, and required configuration (including model
API credentials, and how to run against a local target without live services) land here with
the first working slice.

## Demo path

The end-to-end thread this project is built to demonstrate:

```
goal → LLM-driven discovery run → saved capability artifact →
deterministic replay with input params → escalation to a human on the live session →
evidence for both runs
```

Exact commands — run the agent on a goal, then replay the resulting artifact — go here once
they exist.

## Repository layout

```
README.md               setup + demo path (required deliverable)
REPORT.md               design write-up, seven required headings (required deliverable)
evidence/               example artifact + discovery and replay run logs (required)
docs/
  assignment-brief.md   requirements of record + traceability
  scope.md              depth budget, cut list, build order
  research.md           prior art, domain research, direction check
  architecture.md       proposed design
  decisions.md          decision log — every choice and its justification
```

Source directories are added as implementation lands, so the tree stays honest about what
exists.

## Submission

- Public repo: <https://github.com/PhongCT1105/interface_ai_assessment>
- Email the URL on its own line to `assignments@interface.ai` from `phongct1105@gmail.com`
- No deadline; no zip
