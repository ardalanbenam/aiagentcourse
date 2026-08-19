# Design decisions

A running log. Every major choice, why it was made, what else was on the table, and the conditions
under which the alternative would be correct. Rationale lives here, not in code comments.

Format: `D-NNN`, newest last. Entries are amended rather than rewritten when a decision changes, so
the reasoning stays readable in sequence.

---

## Phase 0 — planning

### D-001 · Domain is code review, not compliance

**Decided.** Build the entire course against automated code review. Map to compliance at the end in
`PORTING.md`.

**Why.** Two reasons, one stated by you and one structural. Yours: learning agent engineering and
compliance vocabulary simultaneously splits attention, and the vocabulary is what the job teaches
for free. Structural: code review has the property that makes evals teachable — defects can be
manufactured, so ground truth is free and no corpus needs hand-labeling before measurement can
start.

**Alternative.** Build directly in compliance. Correct if the goal were domain expertise rather than
method, or if ground truth were equally cheap there — it is not. Seeding a control violation in a
real cloud environment is far more work than flipping a comparison operator in an AST.

---

### D-002 · Raw Anthropic SDK first; framework deferred to Chapter 7

**Decided.** Chapters 1–6 hand-roll the loop against the raw SDK. Framework introduced in Chapter 7,
after six chapters of accumulated friction.

**Why.** Your stated divergence from the Hamel/Shankar course, and I agree with it. An abstraction
adopted before you have felt the problem it solves is an abstraction you cannot evaluate. You will
be making framework calls at work in six months on problems this course does not cover, and the
transferable skill is knowing what the abstraction is *for*.

**What their choice buys that yours does not.** They build on the OpenAI Agents SDK, and it buys
three real things. First, students reach a working agent in one lecture instead of two — for a
cohort course with fixed contact hours that matters enormously. Second, it removes a whole class of
distracting bugs — message accumulation, tool-result formatting, stop-reason handling — that teach
you the SDK's shape rather than agent design. Third, their handoff and guardrail primitives give a
shared vocabulary for multi-agent patterns, so the lecture can discuss architecture without every
student having implemented it differently.

**What you give up by hand-rolling.** Roughly 4–6 hours across Part I, and a genuine risk of
spending an evening on a message-history bug that teaches nothing.

**What you get.** When Chapter 7 asks whether the framework is worth it, you will have a real answer
instead of a preference. That is the thing you said you wanted from this course.

**Conditions under which theirs is right.** Time-boxed under ~40 hours; a team standardising on one
framework where local idiom beats first principles; or any setting where shipping matters more than
understanding the decision space.

---

### D-003 · The framework debate is three-way

**Decided.** Chapter 7 compares hand-rolled loop, the SDK's `tool_runner`, and one full framework.

**Why.** Checked against current releases rather than memory. The Anthropic Python SDK is at
**v0.121.0** (August 2026) and ships `client.beta.messages.tool_runner`, which handles the agentic
loop, error wrapping, type safety, and automatic compaction when token usage crosses a threshold.

This did not exist in the form I remembered, and it materially changes the answer. A significant
fraction of what people adopt LangGraph for — loop management, tool dispatch, context overflow —
the official SDK now provides with zero additional dependencies. A two-way "raw SDK vs. framework"
debate would teach an obsolete decision.

**Stale-API note.** This is the kind of drift your §9 warned about. Recorded here rather than
silently absorbed.

**Alternative.** Keep the binary framing for simplicity. Rejected — it produces a wrong answer, and
the whole point of the chapter is the decision space.

---

### D-004 · Trace schema in Chapter 2, trace backend in Chapter 3

**Decided.** The Pydantic trace model and a JSONL sink land in Chapter 2. Langfuse becomes a second
sink in Chapter 3.

**Why.** Chapter 2 must emit permission denials as first-class trace events, which requires the
trace model to already exist. Beyond ordering, this makes the sink swappable by construction, so
the six-container Langfuse stack is never load-bearing and never blocks a chapter.

**Verified.** Langfuse v3 self-hosting requires web, worker, ClickHouse, Postgres, Redis, and
MinIO, and the full stack is the only officially supported configuration. All infrastructure must
run in UTC or queries silently return wrong results. Your 24 GB machine handles it comfortably; a
16 GB machine would struggle.

**Alternative.** Instrument with the Langfuse SDK directly and skip the intermediate model. Cheaper
by an hour or two, and correct if you were certain of the backend and never intended to change it.
Rejected because it couples your trace semantics to a vendor's schema, and because Chapter 13's
frontier needs prompt-hash provenance that is cleaner to own yourself.

---

### D-005 · Vertical slice closes at Chapter 4, not Chapter 7

**Decided.** Hypothesis-and-proof moves from your Chapter 7 to Chapter 4 and closes Part I. The
framework component that was bundled with it moves to Chapter 7.

**Why.** Your brief requires an end-to-end run producing a real failing test by the end of Part I.
As proposed, Part I ended with an agent that produced opinions — which the architectural principle
explicitly rejects. The two statements could not both hold.

**Alternative.** Redefine the Part I milestone as "structured findings, no proof yet." Rejected: the
proof step is the entire architectural thesis, and deferring it to Chapter 7 means six chapters
built on an unvalidated premise.

---

### D-006 · The world ships in two versions

**Decided.** `world v0` — hand-written, 3 modules, 12 PRs — ships with Chapter 1. `world v1` — the
generator, full service, ~600 PRs — is Chapter 6.

**Why.** Two reasons. Mechanically, Chapter 1's acceptance tests need a fixture to run against.
Pedagogically, and more importantly: building the generator before watching the agent fail produces
a corpus the agent passes, because you unconsciously inject the defects you already know it
catches. Building it after Chapters 4 and 5 have shown you the real failure shape produces a corpus
that discriminates.

**Alternative.** Generate everything up front, as your §6 proposed. Correct if corpus construction
were purely mechanical — but it is a modeling exercise, and modeling before observation is how you
get a benchmark that flatters you.

---

### D-007 · Corpus is 50% negatives

**Decided.** 300 of 600 PRs produce no findings — 150 decoys, 150 clean.

**Why.** False positive rate is the headline metric and can only be attributed cleanly on a PR that
should produce zero findings. Resolving a ~10% FP rate to ±3.4pp at 95% confidence requires 300
negatives. Sizing from the measurement you need, rather than from defect coverage, is the whole
point.

**Honest limitation.** Per-decoy-class rates land at `n ≈ 21`. Two false positives out of 21 gives
a point estimate of 9.5% with an exact Clopper-Pearson interval of [1.2%, 30.4%] — asymmetric and
very wide. Those rates are reported with their intervals and described as directional. The harness
refuses to print a per-class rate without its interval, because a naked point estimate at that
sample size is worse than no number at all.

**Alternative.** More positives for richer defect coverage. Correct if recall were the product. It
is not — precision is, and a corpus that cannot measure precision to a useful tolerance cannot
support the claim the course is built to make.

---

### D-008 · Fourteen chapters with a declared critical path

**Decided.** Ten core chapters (~71 h) form a complete course. Four depth chapters (~25 h) are
deferrable to weeks 9–10, with a stated deferral order.

**Why.** You said you would rather have eight excellent chapters than fourteen mediocre ones, and
the §3 checklist has 43 items covering the full Hamel/Shankar curriculum. Eight chapters cannot hold
it without gutting error analysis and CI/CD, which are the parts you said matter more. Declaring a
critical path preserves both: the core path is genuinely complete, and depth is deferred rather
than silently dropped.

**Alternative.** Cut to eight by dropping Modules 4 and 5. Rejected — adversarial evaluation and the
cost/accuracy frontier are both explicit deliverables, and the frontier is half of Deliverable 2.

---

### D-009 · Ground truth isolated structurally, then attacked

**Decided.** `world/prs/*/proof/` sits outside the sandbox mount, *also* carries an explicit deny
rule, and *also* gets attacked by a prompt injection in Chapter 12.

**Why.** Three layers where one would do, because the redundancy is the lesson. Chapter 12 argues
that sanitization and structural isolation both fail and only privilege separation holds. That
argument is far more convincing demonstrated on this repository — an injection that fully succeeds
at the model layer and achieves nothing — than asserted.

**Alternative.** Keep ground truth in a separate repository. Cleaner isolation, and it would remove
the risk entirely. Rejected precisely because removing the risk removes the demonstration, and
because a single-repo layout is what you will actually face at work.

---

### D-010 · Acceptance tests replay recorded model responses

**Decided.** Model calls in acceptance tests are served from committed cassettes, with a `--live`
flag for real calls.

**Why.** An acceptance test is a specification. A specification that is slow, costs money, and
returns different answers on each run is a bad specification. Cassettes make `make test-chNN` free,
fast, and deterministic.

**Consequence, stated plainly.** A chapter can go green while the agent is still bad. Acceptance
tests verify that your harness handles a given model response correctly; they do not verify that the
model behaves well. Agent quality is measured by `make eval` against the corpus, which is the
correct place for it — but the split needs to be expected rather than discovered.

**Alternative.** Live calls in acceptance tests. Correct only if the chapter's subject genuinely is
model behaviour, which is Chapter 13's territory, where the eval harness already handles it.

---

### D-011 · Checkers and policy merged into one chapter

**Decided.** Your Chapter 5 (deterministic checkers) and Chapter 6 (review policy as data) become a
single Chapter 5.

**Why.** Both produce and consume the same normalized `Finding` schema. Split across two chapters,
that schema gets written twice or Chapter 5 ships without an output contract. The noise problem and
the policy that resolves it are one narrative arc.

**Alternative.** Keep them separate for a lighter per-chapter load. Rejected — this is the only
merge in the plan, and it makes a coherent 7-hour chapter out of two thin ones.

---

### D-012 · New chapter for cross-file reasoning

**Decided.** Chapter 7 owns contextual defects, the retrieval bake-off, context management, and the
framework decision.

**Why.** Your §5 defines a `contextual` defect class and your §8 lists two retrieval and context
debates, but no chapter in §6 owned any of them. A whole defect class with no chapter is the
largest gap in the proposed plan.

Placing the framework decision here rather than at hypothesis-and-proof is deliberate: multi-hop
navigation with partial failures and context pressure is where hand-rolled orchestration actually
starts to hurt. One chapter of raw SDK is not enough friction to justify an abstraction. Six is.

**Alternative.** Fold contextual defects into Chapter 4. Rejected — it would make Chapter 4 a
12-hour chapter and would bury the retrieval bake-off, which deserves real measurement rather than
a paragraph.

---

## Phase 0 — resolved after review

### D-013 · Eval harness samples a fixed, stratified, negative-heavy slice

**Decided.** A 150-PR development slice is the default; the full 600 runs only at gates. Three
properties enforced in code.

**Proportionally stratified, never uniform.** The slice preserves the corpus's negative-heavy
composition exactly: 75 negatives (37 decoy, 38 clean) and 75 positives (15 MECH, 33 SEM, 27 CTX).
Uniform sampling would let class balance drift between runs and throw away the false-positive
resolution D-007 sized the corpus to buy.

**Fixed and seeded, not redrawn.** `--slice dev150` resolves to a committed manifest of PR ids with
its own hash, not to a runtime sampler. Comparability between runs becomes a property of the
artifact rather than a discipline you have to maintain.

**Aggregate FP only on the dev slice.** Per-class rates print at full-corpus gates and nowhere else.
This one is the subtlest and was raised in review: at 75 negatives the per-class cells are single
digits, D-007's refuse-to-print rule would fire on every single run, and a guardrail that fires
constantly is one you learn to disable. Restricting per-class reporting to full-corpus gates is what
keeps the rule credible.

**Honest cost.** 75 negatives resolves aggregate FP rate to roughly ±6.8pp, against ±3.4pp at full
corpus. Enough to see a real move; not enough to call a two-point difference.

**Alternative considered and rejected.** Oversampling negatives in the dev slice — say 100/50 — to
buy FP resolution, then reweighting to recover unbiased estimates. Gets you to ±5.9pp, which is a
marginal gain, and reweighting introduces a class of subtle bugs that is exactly wrong for a
teaching harness. Proportional stratification stays.

---

### D-014 · LangGraph, justified by asynchronous approval specifically

**Decided.** Chapter 7 ports to LangGraph. The justification is Chapter 12's approval flow being
**asynchronous**, and the decision is stated so it can be falsified.

**Why the distinction is load-bearing.** A synchronous approval gate — pause, prompt, human answers
in seconds, continue in-process — works with `tool_runner` directly and requires nothing of the
framework. If that were Chapter 12's flow, checkpointing would not be load-bearing and the
framework argument would collapse to "it is what you will meet at work," which is true but weak.

An asynchronous flow needs durable checkpoints, a resume entry point, and idempotency on replay.
That is LangGraph's interrupt-and-checkpoint model doing real work.

**Which one Chapter 12 builds: asynchronous.** A review bot runs in CI. Nobody is at a terminal, and
the process that started the review is long dead by the time a reviewer looks at the escalation.
The synchronous version would be a demo of a thing that does not exist in production.

**Falsification condition, stated up front.** If Chapter 7's measurements show the framework costing
more than it returns even under the async requirement, that goes in `ALTERNATIVES.md` as a finding.
The point of measuring is that the answer is allowed to come back negative.

**Prior version of this decision.** My original reasoning was "checkpointing gets reused in Chapter
12's approval flow," which assumed the async case without naming it. The assumption was right and
the argument was incomplete. Corrected here.

---

### D-015 · Ratchet never ships fixes — because fix quality has no free ground truth

**Decided.** A candidate fix is generated internally to validate the proving test. It is never
surfaced in the review report.

**Why.** This is the sharper form of the argument, and it supersedes the "different product, scope
discipline" reasoning I gave originally. The entire course rests on one property: *I injected the
defect, so I know the answer.* That property is what made code review the right domain over
compliance (D-001).

"Is this patch good?" has no injected answer. Fix quality would require a human judge or an LLM
judge, and an LLM judge requires validation against labeled data, which is precisely the
hand-labeling burden the domain choice was made to avoid. Shipping fixes reintroduces the exact cost
the design was built to escape.

**The narrow exception, so the distinction stays sharp.** `proof/fix.patch` *is* known-correct
ground truth — but only for the internal fail-on-HEAD / pass-on-fix check, and only loosely even
there, since many valid fixes exist for one defect. It is an oracle for validating a test. It is not
a scoring mechanism for a patch Ratchet proposes, and it must never be used as one.

**Conditions under which shipping fixes would be right.** If the product goal were developer
throughput rather than defect evidence, and you were willing to fund the labeled patch-quality
corpus that a fix-shipping eval harness requires. That is a second course, not a chapter of this
one.

---

### D-016 · Two sandbox modes, with one non-negotiable exception

**Decided.** `--isolation=strict` (fresh container per run) is the Chapter 2 default and the
teaching path. `--isolation=pooled` (warm container, per-run overlay workdir) is what the eval
harness uses from Chapter 9. **The Chapter 12 adversarial suite always runs strict and the flag does
not apply to it.**

**Why both.** Posing this as a single choice was a false binary. Strict is the better lesson and the
stronger guarantee; pooled is the difference between an eval loop you run constantly and one you
avoid. Container startup across hundreds of PRs times several test runs each is the dominant cost of
every iteration, and an eval harness you are reluctant to run is worse than a slightly weaker
isolation story.

**What pooled actually gives up.** The boundary against the *host* is identical — same container,
same network policy, same mount. What weakens is isolation *between runs*: a test that binds a port,
spawns a background process, writes outside its overlay, or mutates site-packages can influence a
later run.

**Why the adversarial exception is absolute.** Persisting state across runs to poison a later review
is exactly the attack the Chapter 12 bucket exists to find. A pooled runner would hide the class of
attack the chapter is hunting. This is the general principle, and it is the `ALTERNATIVES.md` entry:
throughput can be traded against isolation when the code under review is merely *untrusted*, and
never when it is *adversarial*.

**Runtime, detected.** Docker Desktop — `/usr/local/bin/docker` symlinks into
`/Applications/Docker.app`, context `desktop-linux`. No OrbStack, no Colima. Docker Desktop has the
slowest container startup of the three macOS options, which is what promotes the pooled runner from
optimization to requirement. The daemon was stopped, so no benchmark yet; Chapter 2 measures both
modes on your machine and records the real numbers.

---

### D-017 · The cut order is fixed now, and Chapter 12 is not on it

**Decided.** Deferral order: Ch 3 → Ch 10 → Ch 14 → Ch 7, with stated costs. Chapter 12 is core and
is never cut. A decision checkpoint at the end of week 4 picks the lane.

**Why now rather than at week seven.** Deciding under time pressure is how the security chapter gets
dropped, because at week seven the cheapest thing to cut is whatever is next on the calendar rather
than whatever costs least. The week-4 checkpoint replaces an estimate with four weeks of measured
pace and costs nothing to make.

**On the shell-execution concern.** Raised in review, correct, and it lands one chapter earlier than
expected: the isolation boundary is **Chapter 2**, not Chapter 12. Ratchet executes tests against
code it is reviewing from the first moment it can execute anything, so the containerized runner with
no network, read-only mount, and wall-clock timeout is built before the agent has any capability
worth attacking. Chapter 12 does not introduce the boundary — it attacks the one Chapter 2 built.
The safety floor therefore sits in an early core chapter rather than a deferrable one, which is the
property that makes the cut order safe.

---

### D-018 · Trace labeling is human-only, enforced structurally

**Decided.** You label the 60+ traces in Chapter 8. Not me, not a model. The annotation tool ships
with **no model integration at all**.

**Why structural rather than a promise.** Chapter 8 opens by demonstrating why "let the agent
evaluate everything" fails. Delegating the annotation to an agent is that failure with extra steps,
and it would contaminate every judge, prevalence estimate, and improvement decision downstream. A
rule that depends on willpower at 11pm on a Tuesday is not a rule, so there is no suggest-a-label
button, no pre-filled category, and no similarity ranking to reach for.

**The line on what I may do.** Mechanical preparation only: rendering diffs beside findings, pulling
traces into readable shape, deduplicating byte-identical traces, and ordering the queue randomly
within stratum from a fixed seed.

**Explicitly excluded: model-ranked ordering**, including by "likely interesting." That biases which
failures you see toward what a model already believes is interesting, which is the contamination in
a subtler costume.

**Saturation, made concrete.** 60 traces is the floor, not the target. Continue until 10 consecutive
traces produce no new failure mode. If that has not happened by 60, that is a real signal and the
answer is to keep reading.
