# Ratchet — a course in building and evaluating agentic systems

Eight to ten weeks of building one system, end to end, with the evaluation harness treated as the
primary deliverable rather than an afterthought.

You build **Ratchet**: an agentic code review system that finds real defects in a pull request and
proves each one with a failing test. You then build the thing that matters more — an eval harness
that runs Ratchet across a corpus where ground truth is known and reports true positive rate,
false positive rate, bootstrap confidence intervals, regression against a committed baseline,
adversarial results, and a cost-vs-accuracy Pareto frontier.

Anyone can demo an agent. Very few people can show the numbers proving one works on data where the
right answer is known.

---

## The principle the whole course turns on

**The agent orchestrates. It does not adjudicate.**

An agent that says "I think line 44 has an off-by-one" has produced an opinion. An agent that
writes a test that fails on the current code and passes on the fix has produced *evidence* —
runnable, reproducible, and checkable by someone who does not trust the agent at all.

So the agent's job is the messy work at the edges: deciding what is worth investigating, locating
relevant code across files, forming a hypothesis, and constructing the artifact that would prove or
disprove it. The verdict itself belongs to a deterministic function — a linter exit code, a test
result, a type error.

This has a security twin that arrives in Chapter 12 but is built in Chapter 2:
**authorization must hold even when the model is compromised.** Prompt injection has no reliable
detector. Therefore permissions live in code, never in the prompt.

Every chapter asks the same question in a new setting: *which part of this belongs to the model,
and which part belongs to code?*

---

## How a chapter works

Four sections, always in this order, in every `chapters/chNN/README.md`:

1. **What we're building and why** — 200 words. No throat-clearing.
2. **The brief** — requirements written as a ticket, with explicit non-goals.
3. **Acceptance criteria** — `make test-chNN` goes green. The tests are the specification.
4. **Hints** — collapsed, at the bottom, progressive. Never the answer.

Then you stop reading and go write code.

When `make test-chNN` passes, and *only* then, you read `solutions/chNN/`:

- `SOLUTION.md` — an annotated walkthrough of the reference implementation, including what was
  tried and abandoned.
- `ALTERNATIVES.md` — the part that matters most. For every significant decision: what was chosen,
  what else was on the table, why this one, and the specific conditions under which a different
  choice would be correct.

**Solutions live in a separate tree on purpose.** If `SOLUTION.md` sat next to the brief you would
read it before trying. Do not go looking. The acceptance tests check behaviour and contracts, not
implementation details, so your solution can differ from the reference and still pass — and if it
does differ, `ALTERNATIVES.md` is where you find out whether you picked a good one.

### The rule about hints

Use hint 1 after you have been stuck for 30 minutes. Use hint 2 after another 30. If you burn
through all the hints and are still stuck, that is a defect in the chapter — open an issue against
this repo describing where you got lost. The brief was supposed to be sufficient.

---

## Prerequisites

**Skills you need:** competent Python, comfort with APIs, Docker, and the command line.

**Skills you do not need:** any prior experience with agentic systems or LLM evals. That is what
this is for.

**Machine:** macOS or Linux. 16 GB RAM minimum, 24 GB comfortable — the Chapter 3 observability
stack runs six containers. 20 GB free disk.

**Accounts and keys:**

| What | Needed by | Notes |
|---|---|---|
| Anthropic API key | Ch 1 | Budget guidance in `COURSE_PLAN.md`. Acceptance tests replay recorded responses and cost nothing. |
| Docker daemon running | Ch 2 | Docker Desktop, OrbStack, or Colima. The sandboxed test runner needs it. |
| GitHub account | Ch 11 | For the Actions gate. A fork of this repo is enough. |

**Deliberate domain choice.** The domain is automated code review, not compliance, even though
compliance is where this work is heading. Learning agent engineering and a new professional
vocabulary at the same time splits attention, and the vocabulary is the part the job teaches for
free. Code review has the property that makes evals teachable: you can manufacture the defects, so
ground truth is free. `PORTING.md`, written at the end, maps every structural lesson onto
compliance control automation one-to-one.

---

## Setup

Not yet written — Phase 0 is planning only. When Chapter 1 ships, setup will be:

```bash
git clone git@github.com:ardalanbenam/aiagentcourse.git ratchet-course
cd ratchet-course
make setup
```

`make setup` installs dependencies with `uv`, materialises `world/`, and runs a self-check that
tells you exactly which prerequisites are missing. `docker compose up` brings up the observability
stack from Chapter 3 onward. Every heavy service has a lightweight fallback so you are never
blocked on infrastructure — the trace layer in particular writes JSONL by default and treats
Langfuse as an optional second sink.

---

## Repository map

Files marked *planned* do not exist yet.

```
README.md               you are here
COURSE_PLAN.md          chapter map, objectives, time budget, syllabus coverage table
WORLD_SPEC.md           spec for the synthetic service and PR corpus
DESIGN_DECISIONS.md     running log of every major choice, its rationale, and its alternatives
OPEN_QUESTIONS.md       decisions that need your input before Chapter 1

SPEC.md                 planned — you write it in Ch1, revise it throughout
PROGRESS.md             planned — the checklist you tick off
GLOSSARY.md             planned — eval terminology in plain language
PORTING.md              planned — how all of this maps onto compliance automation
Makefile                planned
pyproject.toml          planned
docker-compose.yml      planned

world/                  planned — the synthetic service under review and the PR corpus
src/ratchet/            planned — the system you build, growing chapter by chapter
chapters/chNN-name/     planned — README, acceptance tests, starter skeleton
solutions/chNN/         planned — reference implementation, kept deliberately far away
```

---

## Status

**Phase 0 — planning. Awaiting review.**

The five documents above are the complete Phase 0 deliverable. Nothing is built yet. Read
`COURSE_PLAN.md` for the chapter plan and what changed from the original brief, `WORLD_SPEC.md`
for the synthetic corpus design, and `OPEN_QUESTIONS.md` for the eight decisions that need your
answer before Chapter 1 can be built.

Chapters are built one at a time, on explicit instruction, in this order: world pieces the chapter
needs → acceptance tests → starter skeleton → reference solution → README → ALTERNATIVES.md. Tests
are verified to fail on the starter and pass on the solution before a chapter is called ready.
