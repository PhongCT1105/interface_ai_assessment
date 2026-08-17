# Evidence

Required deliverable (brief Section 6.3). The brief is explicit that the discovery run
cannot be described — it has to have happened, with the evidence here to show it:

> "At least one genuine LLM-driven run against a live surface, with the evidence in
> `/evidence/` to show it happened. That's the heart of the project and we can't assess a
> description of it."

## Required contents

- [ ] A saved example capability artifact
- [ ] Logs from a **discovery run** (the real LLM-driven run, including the model's decision
      rationale per step)
- [ ] Logs from a **replay run** (successful, showing zero model calls and the verified
      checkpoint)
- [ ] Ideally: a **replay that hits an error or exceptional state** — bad input, a not-found
      result, or an injected/simulated failure — to show how it is detected and reported
- [ ] Optional: a short screen recording

## Rules for what lands here

Everything in this directory is public. It passes through the same redaction as artifacts
and logs: no credentials, no tokens, no real PII, no real institution data. Screenshots get
the same treatment as text.

Currently empty — populated by the first real discovery and replay runs.
