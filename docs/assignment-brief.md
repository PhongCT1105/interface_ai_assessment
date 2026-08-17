# Assignment brief — requirements of record

## Source

| Field | Value |
| --- | --- |
| Subject | Interface AI - Next step for your application - take-home assignment |
| From | `no-reply@ashbyhq.com` (interface.ai Recruiting Team, via Ashby) |
| Received | 2026-08-13 |
| Attachment | `Assignment A — Computer-Use Automation System.pdf` |
| Role | Software Engineer, interface.ai |

## Email body (verbatim)

> Hi Phong,
>
> Thank you for applying to the Software Engineer role at http://interface.ai. The next
> step is a build assignment - everyone who applies gets the same one, and we evaluate
> the work.
>
> The assignment brief is attached as a PDF.
>
> You'll build the layer that gives an AI agent hands: an LLM works out how to complete a
> task inside a real UI that has no API, the successful run is recorded as a reusable
> capability, and that capability replays deterministically afterward with no model in the
> loop.
>
> Push it to a public GitHub repo and email it to mailto:assignments@interface.ai.
>
> No deadline - send it when it represents your best work. AI-assisted development is
> assumed and encouraged; you'll need to be able to defend everything you submit.
>
> We know this is a real investment of your time, and we don't ask for it lightly. Every
> submission is reviewed in detail by our engineering team, this isn't a filter that goes
> into a void.
>
> If the work is strong, the next step is a conversation with our leadership team about
> what you built.
>
> http://interface.ai Recruiting Team

## Outstanding: the PDF

The detailed brief lives only in the PDF attachment and is **not yet in this repo**. The
requirements below are derived from the email body alone, so treat them as the shape of
the problem rather than the full specification. Anything the PDF adds — required
interfaces, a specific target UI, evaluation criteria, constraints on tooling — takes
precedence and should be folded into this file.

To close the gap: download the attachment from the email and save it as
`docs/assignment-brief.pdf`, then this document gets reconciled against it.

## Requirements derived from the email

1. **LLM-driven exploration.** An LLM must work out how to complete a task inside a real
   UI that exposes no API. The agent perceives the interface and acts on it — this is
   computer use, not scripted selectors handed to it up front.
2. **Capability recording.** A successful run is recorded as a reusable capability. The
   record must be concrete enough to re-execute and general enough to be worth reusing.
3. **Deterministic replay, no model in the loop.** Replaying a capability must involve no
   LLM call at all. Same inputs must produce the same actions.
4. **Real UI, no API.** The target must genuinely lack a programmatic path, otherwise the
   whole premise is sidestepped.

## Submission requirements

- [ ] Public GitHub repo — <https://github.com/PhongCT1105/interface_ai_assessment>
- [ ] Email the repo to `assignments@interface.ai`
- [ ] Every decision defensible in conversation with the leadership team
- [ ] No deadline — ship when it represents best work

## Judgment notes

- AI-assisted development is explicitly encouraged, and defensibility is explicitly
  required. The decision log in [decisions.md](decisions.md) exists to serve that second
  half: for each meaningful choice, what was picked, what was rejected, and why.
- "Deterministic" is the load-bearing word in this assignment. It should be demonstrable,
  not asserted — a replay that provably makes zero model calls is the thing to show.
