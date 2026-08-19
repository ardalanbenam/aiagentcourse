# Open questions

**Status: resolved through two review rounds. Three defaults still standing (#3, #7, #8), none of
which bite before Chapter 8.** Chapter 1 is unblocked.

Resolved answers are recorded in full in `DESIGN_DECISIONS.md`; this file is the index and the
record of what is still outstanding.

---

## Resolved

| # | Question | Decision | Entry |
|---|---|---|---|
| 1 | API budget and sampling | 150-PR dev slice: proportionally stratified, fixed and seeded, aggregate FP only. Full 600 at gates. | D-013 |
| 2 | Framework for Ch 7 | LangGraph, justified by Ch 12's approval flow being **asynchronous** — stated so it can be falsified. | D-014 |
| 4 | Does Ratchet ship fixes? | No. Generated internally to validate the test, never surfaced. Fix quality has no free ground truth. | D-015 |
| 6 | Sandbox runtime | Both modes, flag-gated. Strict for teaching, pooled for the harness, **strict always for the adversarial suite**. Runtime detected: Docker Desktop. | D-016 |
| — | Cut order | Ch 3 → Ch 10 → Ch 14, and it ends there. Ch 7 and Ch 12 are never cut. Week-4 decision checkpoint. | D-017, D-019 |
| — | Trace labeling | Human-only, enforced structurally — the annotation tool has no model integration, and it reads the JSONL sink so cutting Ch 3 cannot break it. | D-018, D-004 |
| 5 | Cassettes in acceptance tests | Accepted, with two conditions: `make record` + cassette provenance (model id, prompt hash) with stated re-record triggers; green-≠-good restated in every README from Ch 8 on. | D-010 |
| — | Ch 7 classification | Moved to core — load-bearing for Ch 12's checkpointing and for 36.7% of the positive corpus. Core is 11 chapters; depth is Ch 3, 10, 14. | D-019 |
| — | FP attribution | Diff-scoping made explicit: scoreable findings are regressions (absent at base, present at head), base revision is the oracle for touched-but-already-bad lines, pre-existing issues route to a capped unscored advisory channel. | D-020 |

Three corrections were adopted from review rather than merely accepted, and are worth noting because
they changed the design rather than confirming it:

- **The dev slice reports aggregate FP only.** Without this, D-007's refuse-to-print rule fires on
  every development run and becomes a guardrail you learn to switch off. This is the kind of failure
  where a discipline decays through friction rather than through disagreement.
- **The LangGraph justification names sync vs. async.** The original reasoning assumed the
  asynchronous case without stating it, which made a correct conclusion rest on an unstated premise.
- **The no-fixes argument is about ground truth, not scope.** Shipping fixes would require a labeled
  patch-quality corpus, reintroducing exactly the hand-labeling cost that made code review the right
  domain in the first place.

The second round added three more:

- **Chapter 7 was misclassified**, and the tell was in my own table — a deferral entry marked "only
  cut if abandoning the course" is not a deferral entry (D-019).
- **The FP metric rested on an unstated assumption.** Diff-scoping is now the stated ground of the
  headline number, and the touched-but-already-bad case has a mechanical rule instead of an
  accident of implementation (D-020).
- **D-003 was stale about compaction.** The SDK's client-side compaction is deprecated in favor of
  server-side context editing — verified against the platform docs and amended, which is the
  fetch-current-docs rule catching drift in the very entry that recorded the last drift.

---

## Proceeding on default — override any of these before the relevant chapter

Nothing here bites before Chapter 8.

### 3. Annotation interface is a small local web app — bites at Chapter 8

FastAPI + HTMX, no build step, keyboard shortcuts. Chosen over a terminal TUI because you will be
reading diffs beside findings beside traces for 60+ traces and side-by-side rendering matters more
than launch speed. No model integration, per D-018.

### 7. Appendix B targets `httpx` plus one repo of your own — bites at Appendix B

Real test suite, type annotations, non-trivial dependency graph, no database required. If you have a
repository in mind — especially one of your own with a real test suite — name it whenever you
like; before Appendix B is built is all that matters, and the adapter is cheaper to build once
than twice.

### 8. Repository stays public — bites whenever you decide

The corpus is entirely synthetic, so nothing in ground truth is sensitive, and Deliverable 2 is
worth showing. The one thing to keep in mind: from Chapter 8 the repo will contain your own
annotations, which are your judgements about failure modes and slightly more personal than code.
Easy to keep those local if you would rather — say so before Chapter 8.

---

## Decided without asking

**The synthetic-data generation "skill"** from your §3 checklist gets built as both a Python module
(`world/generate.py`) and a thin Claude Code skill wrapping it. The module is what the course uses;
the skill is what ports to compliance work, where you will want to generate seeded control
violations the same way.

**Chapter 5 does not resolve fail-open vs. fail-closed.** You asked for the tension surfaced rather
than settled. The chapter gives you the concrete case, data from both configurations, and both
arguments. Acceptance tests pass either way, checking only that your choice is explicit and that a
checker failure is visible in the report rather than silent.
