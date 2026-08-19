# Open questions

Eight decisions that would change my design. Every one has a recommended default — if the defaults
are all fine, reply "defaults are fine" and Chapter 1 gets built.

Questions 1, 2, 4, and 5 change chapter *content*. The rest change logistics.

---

### 1. API budget, and whether the eval harness samples

**Recommended default: stratified sample of 150 PRs for iteration, full 600 only at gates.**

This is the question with the largest design consequence. A full-corpus run is 600 reviews. At
Chapter 13 you run the full suite across roughly a dozen frontier configurations, plus the improve
loop's inner iterations. Naively that is tens of thousands of reviews.

The fix is architectural, not incidental: the harness takes a `--slice` argument backed by a
stratified sampler that preserves class balance, and the sample is the default everywhere except
CI gates and the final held-out read. It has to be designed in from Chapter 9 rather than added
later, because sample-vs-full changes how confidence intervals are computed and reported.

**What I need:** a rough ceiling on API spend for the whole course, and confirmation you want the
sampling design. If your budget is generous I would still recommend sampling — it makes the
iteration loop fast enough to actually use, which matters more than the money.

---

### 2. Which framework in Chapter 7

**Recommended default: evaluate `tool_runner` and LangGraph, implement with LangGraph.**

The Anthropic SDK now ships `client.beta.messages.tool_runner`, which handles the agentic loop and
automatic compaction. So the ladder has three rungs, not two, and Chapter 7 measures all three.
But you only *port* to one:

| Option | For | Against |
|---|---|---|
| **LangGraph** | The one you are most likely to meet at work. Durable execution, checkpointing, human-in-the-loop interrupts, replay — all of which Chapters 11 and 12 want anyway. | Heaviest abstraction. Most to learn that is framework-specific rather than transferable. |
| **Pydantic AI** | Matches "Pydantic for every schema, everywhere". Typed, light, fast to ship. Least new vocabulary. | Mostly-linear agents are its sweet spot; you would feel less of what a framework buys, which weakens the chapter. |
| **SDK `tool_runner` only** | Zero new dependencies. Honest answer to "do I need a framework?" for many real systems. | You never feel the framework tradeoff, and one of your stated goals is understanding that decision space. |

LangGraph is my recommendation specifically because the checkpointing and interrupt machinery gets
reused in Chapter 12's human-approval flow, so the abstraction pays for itself inside the course
rather than only in principle. Say the word if you would rather use Pydantic AI.

---

### 3. Annotation interface in Chapter 8

**Recommended default: small local web app.**

Terminal TUI is faster to build and keyboard-driven, but you will be reading diffs beside findings
beside traces for 60+ traces, and side-by-side rendering matters more than launch speed. A minimal
FastAPI + HTMX app, no build step, keyboard shortcuts for the common path.

Tell me if you would rather stay in the terminal.

---

### 4. Does Ratchet ship fixes, or only proofs?

**Recommended default: generate fixes internally to validate the test, never ship them.**

Chapter 4 must generate a candidate fix, because "passes on the fix" is half of the proof. The
question is whether that fix appears in the review report.

I recommend no. The moment Ratchet proposes fixes it is a different product: you are now evaluating
patch quality, which needs its own failure taxonomy, its own judges, and its own corpus. Your two
deliverables stay measurable if the output is *evidence*, not *patches*.

This is worth a deliberate answer because it is genuinely arguable — a fix is often the clearest
possible explanation of a defect, and reviewers like receiving them.

---

### 5. Acceptance tests: recorded cassettes

**Recommended default: yes, cassettes.**

Chapters 1, 4, 7, 9, 12, and 13 involve live model calls. If `make test-chNN` hits the API it is
slow, costs money, and is non-deterministic — so it is a bad specification.

I would record model responses once and replay them in tests, with a `--live` flag for when you
want the real thing. Consequence: acceptance tests verify your *harness* handles a given model
response correctly, not that the model behaves well. Agent quality is measured by `make eval`
against the corpus, which is the right place for it anyway.

The thing to confirm: this means a chapter can go green while your agent is still bad. That is
intentional and I want you to expect it rather than discover it.

---

### 6. Sandbox runtime on macOS

**Recommended default: one long-lived container, per-run tmpfs overlay.**

Chapter 2's test runner needs real isolation. A fresh container per test run is the clean design
but costs 2–4 seconds of startup, which is significant across 600 PRs with multiple test runs each.
A single long-lived container with a per-run overlay and a wall-clock timeout gets startup to
roughly 200ms with isolation that is weaker between runs but identical against the host.

**What I need:** which Docker runtime you have — Docker Desktop, OrbStack, or Colima. OrbStack in
particular is fast enough that per-run containers might just work, which would let me pick the
cleaner design. Your daemon was not running when I checked, so I could not detect it.

---

### 7. Target repository for Appendix B

**Recommended default: `httpx`, plus one repo of your own.**

Appendix B points Ratchet at real code. A good target has a real test suite, type annotations, a
non-trivial dependency graph, and a build that works without a database — `httpx` fits, is
well-known enough to be a meaningful demo, and is small enough to review.

If you have a repository in mind, especially one of your own, tell me and I will pre-build the
harness adapter for it.

---

### 8. Repository visibility

**Recommended default: public.**

`aiagentcourse` is currently empty and public. The corpus is entirely synthetic so there is nothing
sensitive in ground truth, and a public repo is more useful as a portfolio piece — Deliverable 2 is
the thing you want people to be able to look at.

Flagging it only because the repo will eventually contain your annotations from Chapter 8, which
are your own judgements about failure modes and slightly more personal than code. Easy to keep
those local if you would rather.

---

## Two things I decided rather than asked

**The "reusable synthetic-data generation skill" in your §3 checklist** gets built as both a Python
module (`world/generate.py`) and a thin Claude Code skill wrapping it. The module is what the
course uses; the skill is what ports to your compliance work, where you will want to generate
seeded control violations the same way. No reason to choose.

**Chapter 5 does not resolve fail-open vs. fail-closed.** You asked for the tension surfaced rather
than settled, so the chapter gives you the concrete case, the data from both configurations, and
both arguments — and the acceptance tests pass either way, checking only that your choice is
explicit and that the failure is visible in the report rather than silent.
