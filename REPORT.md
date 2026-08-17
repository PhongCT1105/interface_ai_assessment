# Design write-up

> **Skeleton.** The seven headings below are mandated by the brief (Section 6.2) and must
> appear in this order. Content is written as the implementation lands, drawing on
> [docs/architecture.md](docs/architecture.md) and [docs/decisions.md](docs/decisions.md).
> Target length ~1–3 pages total — this is a judgment document, not a manual. Delete this
> note before submission.

## 1. Architecture

The architecture and the key decisions, plus trade-offs.

_To write. Source: docs/architecture.md (layering, and why policy and session sit beneath
both discovery and replay)._

## 2. Artifact schema

The schema and why it is shaped that way.

_To write. This is the focal point of the evaluation — cover per-step `expect`, explicit
compile-time parameterization, exceptional states living in the artifact rather than the
engine, and `app.product` as identity separate from tenant instance._

## 3. Determinism & error handling

How replay is made deterministic, and how runtime errors and exceptional states are detected
and handled (and, secondarily, any UI drift).

_To write. Cover the structural exclusion of the model from replay, the locator strategy and
fallback order, condition-based waiting, and the three-class result contract: business
outcome vs. recoverable condition vs. hard failure._

## 4. Heterogeneity & multi-tenant

How the design extends to legacy web and desktop surfaces, and to reuse across institutions
running the same app.

_To write. Cover the surface driver seam, why normalization is accessibility-tree-shaped, and
base artifact + per-tenant override layers with drift detection._

## 5. Escalation & handoff

How "stuck" is detected, how a human takes control of the live session, and how control is
handed back.

_To write. Cover stuck detection, the intervention request payload, the single-owner control
model, session survival across the transfer, and capture of the human's actions._

## 6. Safety

The guardrail model and its limits.

_To write. Cover allowlist enforcement at a single chokepoint across both paths, action risk
classification, redaction at the persistence boundary — and be honest about what this model
does not protect against._

## 7. Cuts

What was deliberately left out, and what would be built next.

_To write. Every mock and stub, why it was chosen as the cut line, and what the next
increment would be._
