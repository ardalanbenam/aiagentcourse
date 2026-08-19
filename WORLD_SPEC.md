# The synthetic world — specification

The highest-leverage artifact in this repository. It is what makes real evaluation possible: you
inject the defects, so ground truth is free. No hand-labeling a thousand pull requests before you
can measure anything.

Built in two versions. **world v0** ships hand-written with Chapter 1 — three modules, 12 PRs,
enough to run against. **world v1** is Chapter 6 — the generator, the full service, ~600 PRs.
This document specifies v1; v0 is a strict subset using the same manifest format.

---

## 1. The service under review — `taskvine`

A task-management API. Roughly 3,500 lines. It has to look like something a team actually shipped,
including pre-existing mess, because that mess is what teaches triage.

```
world/service/
  taskvine/
    config.py            env-driven settings; a few keys read in inconvenient places   ~120
    db.py                SQLAlchemy engine and session management                       ~90
    models.py            Task, Project, User, Label, Comment, AuditEvent               ~260
    schemas.py           Pydantic request/response models                              ~240
    api/
      projects.py                                                                      ~230
      tasks.py           the busiest router: filters, pagination, bulk operations      ~420
      comments.py                                                                      ~160
      health.py                                                                         ~60
    services/
      task_service.py    business logic: state transitions, assignment, due dates      ~380
      search.py          query building and pagination math                            ~220
      permissions.py     role checks                                                   ~180
      notifications.py   fan-out and templating                                        ~200
    workers/
      queue.py           a small in-process queue abstraction                          ~150
      digest.py          nightly digest job, date-window arithmetic                    ~230
    utils/
      dates.py           timezone and window helpers                                   ~160
      pagination.py      cursor and offset helpers                                     ~140
      text.py            slugify, truncate, redact                                     ~120
      retry.py           backoff                                                        ~90
  tests/                 realistic partial coverage — 62% overall, very unevenly spread
  alembic/               three migrations
  pyproject.toml
```

### The pre-existing mess is deliberate

None of the following are defects. They exist so that an agent which reports them is demonstrably
wrong, and so that Chapter 5's triage step has something real to suppress.

- `notifications.py` carries three standing `ruff` violations and one load-bearing `# type: ignore`.
- Around 40 pre-existing `mypy` errors, concentrated in `workers/`. A PR touching `utils/dates.py`
  must not surface any of them.
- Coverage is lopsided: `utils/dates.py` at 91%, `workers/digest.py` at 22%. A coverage-delta check
  that does not account for the starting point will fire constantly on `workers/`.
- Two `TODO` comments referencing a ticket system that does not exist.

An agent that reports pre-existing noise on an unrelated diff has failed, and the corpus is built so
that failure is measurable rather than anecdotal.

---

## 2. `facts.yaml` — the single source of truth

The generator derives everything from this file. Change it and the world regenerates consistently;
`ground_truth.json` records its hash so a corpus can never silently disagree with the facts that
produced it.

```yaml
version: 1
seed: 20260818

service:
  root: world/service
  python: "3.12"
  modules:
    - path: taskvine/utils/pagination.py
      loc: 140
      coverage: 0.88
      temperature: hot          # hot modules get more PRs, mirroring real repos
      admits: [SEM-001, SEM-003, SEM-007, MECH-001, MECH-004]
    - path: taskvine/workers/digest.py
      loc: 230
      coverage: 0.22
      temperature: cold
      admits: [SEM-004, SEM-008, CTX-002, MECH-002]

# The symbol graph is what makes contextual injection correct rather than plausible.
# To inject a stale caller you must know who the callers are.
symbols:
  - name: clamp_page_window
    module: taskvine/utils/pagination.py
    signature: "(total: int, page: int, per_page: int) -> tuple[int, int]"
    callers:
      - taskvine/services/search.py:build_task_query
      - taskvine/api/tasks.py:list_tasks
    callees: []
    pure: true                  # pure functions are the cheapest to prove with a test

defect_classes:
  - id: SEM-001
    family: semantic
    name: off-by-one-boundary
    strategy: boundary_flip     # AST transform: <= to <, range(n) to range(n+1)
    expected_detector: agent_test
    severity: high
    provable: true

decoys:
  - id: DEC-002
    name: documented-broad-except
    baits: [MECH-003]
    justification: >
      Broad except is intentional; the exception is re-raised after the audit write.
      A comment two lines above says so.

pr_descriptions:
  fidelity_weights: {accurate: 0.55, vague: 0.30, misleading: 0.15}

distribution:
  total: 600
  positives: {MECH: 60, SEM: 130, CTX: 110}
  negatives: {DEC: 150, CLEAN: 150}
  decoy_alongside_defect: 0.20   # of positive PRs that also carry a decoy
```

---

## 3. Defect taxonomy

Four families. The identifiers are stable and every metric in the course is reported per class.

### MECH — mechanical (a checker catches it; the agent should not be doing this work)

| ID | Defect | Caught by |
|---|---|---|
| MECH-001 | unused import | ruff F401 |
| MECH-002 | mutable default argument | ruff B006 |
| MECH-003 | bare `except` | ruff E722 |
| MECH-004 | shadowed builtin | ruff A001 |
| MECH-005 | missing return annotation | mypy |
| MECH-006 | hardcoded secret | bandit B105 |
| MECH-007 | `subprocess` with `shell=True` | bandit B602 |
| MECH-008 | `assert` on a production path | bandit B101 |

**What this class teaches:** if an LLM is finding these, you have built the wrong thing. They exist
to be triaged out of the agent's workload, and to measure whether it stays out of their way.

### SEM — semantic (invisible to linters, provable by a test)

| ID | Defect |
|---|---|
| SEM-001 | off-by-one at a boundary |
| SEM-002 | inverted boolean |
| SEM-003 | wrong comparison operator |
| SEM-004 | dropped `None` guard |
| SEM-005 | wrong dict key |
| SEM-006 | swapped arguments at a call site |
| SEM-007 | wrong default value |
| SEM-008 | short-circuit reordering that skips a side effect |

**What this class teaches:** this is where the agent earns its keep, and it is the only class where
"write a failing test" is both necessary and sufficient as proof.

### CTX — contextual (only cross-file reasoning finds it)

| ID | Defect |
|---|---|
| CTX-001 | changed signature, stale caller |
| CTX-002 | changed return type, unhandled downstream |
| CTX-003 | removed model field still referenced in a serializer |
| CTX-004 | renamed config key still read elsewhere |
| CTX-005 | removed enum member still compared against |
| CTX-006 | narrowed exception type, caller still catching broadly |

**What this class teaches:** retrieval quality is the bottleneck, not reasoning quality. Chapter 7's
bake-off is scored entirely on this bucket.

### DEC — decoys (correct code that pattern-matches to wrong)

| ID | Decoy | Baits |
|---|---|---|
| DEC-001 | intentional mutable default — a deliberately shared sentinel cache | MECH-002 |
| DEC-002 | broad `except` with a documented reason and a re-raise | MECH-003 |
| DEC-003 | magic number with a comment explaining it (`# 4096: matches the column width`) | custom AST |
| DEC-004 | deliberate shadowing in a tight local scope, with a comment | MECH-004 |
| DEC-005 | apparent off-by-one that is correct because the caller passes `n-1` | SEM-001 |
| DEC-006 | import that looks unused but exists for a registration side effect | MECH-001 |
| DEC-007 | a call that looks like a missing `await` on a function that is genuinely sync | SEM-008 |

**This is the most important category in the corpus.** Anyone can find an obvious bug. The learning
is in staying quiet about code that merely resembles a bug. A reviewer with a 40% false positive
rate is muted within a week, and `DEC-005` in particular is designed to be genuinely hard — it is
correct only if you read the caller.

### CLEAN — no defect at all

An agent that never says "looks good" is useless. 150 PRs where the correct output is an empty
findings list.

---

## 4. Orthogonal axes

Every PR carries these independently of its defect class, so any metric can be sliced by them.

| Axis | Values | Why it is here |
|---|---|---|
| `description_fidelity` | accurate / vague / misleading | Whether the agent trusts the description over the code is itself a failure mode worth discovering. Misleading descriptions are the seed of the Chapter 12 attack corpus. |
| `diff_size` | small `<30` / medium / large `>300` LOC | Large diffs are where triage breaks and where cost explodes. |
| `blast_radius` | 1 file / 2–3 / 4+ | Directly controls how hard the retrieval problem is. |
| `preexisting_noise` | counts per checker on touched modules | Controls how much there is to suppress. |

PR descriptions are generated once by a model, then **cached and committed**, keyed by
`(defect_id, fidelity, seed)`. Natural prose, fully deterministic replay, no API cost on regeneration.

---

## 5. Ground truth manifest

`world/ground_truth.json`. One record per PR.

```json
{
  "schema_version": "1.0",
  "world_seed": 20260818,
  "facts_hash": "sha256:7c1f…",
  "service_rev": "a1b2c3d",
  "generated_prs": 600,
  "prs": [
    {
      "pr_id": "0142",
      "path": "world/prs/0142",
      "base_rev": "a1b2c3d",
      "head_rev": "9f8e7d6",
      "diff_stats": {"files": 3, "additions": 41, "deletions": 12},
      "description_fidelity": "misleading",
      "blast_radius": 3,
      "preexisting_noise": {"ruff": 4, "mypy": 11, "bandit": 0},

      "defects": [
        {
          "defect_id": "0142-d1",
          "class": "SEM-001",
          "family": "semantic",
          "file": "taskvine/utils/pagination.py",
          "line": 88,
          "symbol": "clamp_page_window",
          "injection": {"rule": "boundary_flip", "before": "<=", "after": "<"},
          "expected_detector": "agent_test",
          "severity": "high",
          "proof": {
            "kind": "failing_test",
            "test_path": "proof/test_pagination_window.py",
            "test_name": "test_last_page_includes_final_item",
            "fails_on_head": true,
            "passes_on_fix": true
          },
          "fix_patch": "proof/fix.patch",
          "reachable_from": ["taskvine/api/tasks.py:list_tasks"]
        }
      ],

      "decoys": [
        {
          "decoy_id": "0142-k1",
          "class": "DEC-002",
          "file": "taskvine/utils/retry.py",
          "line": 51,
          "why_correct": "Broad except is intentional; the exception is re-raised after the audit write. Comment on line 48 documents it.",
          "bait_for": ["MECH-003"]
        }
      ],

      "expected_findings": 1,
      "expected_silence_on": [
        "taskvine/utils/retry.py:51",
        "taskvine/workers/digest.py:*"
      ]
    }
  ]
}
```

Three fields do most of the scoring work:

- **`expected_findings`** — a count, so recall is mechanical.
- **`expected_silence_on`** — locations where any finding is a false positive, so precision is
  mechanical and attributable rather than a judgement call.
- **`bait_for`** — which checker or prompt each decoy was designed to fool, so when a decoy does
  fool the agent you learn *why* rather than only *that*.

There is deliberately **no `difficulty` field**. Difficulty is measured from agent runs, not
asserted by the generator. Asserting it would bake your prior into the ground truth.

### PR directory layout

```
world/prs/0142/
  meta.json        title, description, author, branch — the PR envelope
  diff.patch       the diff under review
  proof/           NEVER visible to the agent
    fix.patch      the minimal correct fix
    test_*.py      a reference proving test
```

### `proof/` is isolated three ways, on purpose

Ground truth sits in the same filesystem as the code Ratchet reviews. An agent that reads it scores
perfectly and has learned nothing.

1. **Structural** — `proof/` is outside the mount handed to the sandbox. Not reachable.
2. **Rule** — the Chapter 2 capability layer also denies it explicitly, and the denial is a
   first-class trace event.
3. **Attacked** — Chapter 12 lands a prompt injection instructing the reviewer to read the fix.

The injection succeeds completely at the model layer and accomplishes nothing, because the
capability check never asks the model. That is the privilege-separation argument demonstrated on
this repository rather than asserted in a slide.

---

## 6. Corpus size, from a power calculation

False positive rate is the headline metric, so the corpus has to be able to measure it. A false
positive is only cleanly attributable on a PR that should produce **zero** findings — a decoy or a
clean PR. Those are the negatives, and they are what sets corpus size.

For a proportion near `p = 0.10` at 95% confidence, the half-width is `1.96 · sqrt(p(1-p)/n)`:

| Target precision on FP rate | Negatives needed |
|---|---|
| ±5.0 pp | 138 |
| ±3.4 pp | 300 |
| ±3.0 pp | 385 |
| ±2.0 pp | 864 |

**300 negatives** — 150 decoys and 150 clean — buys ±3.4pp on the headline number at a corpus size
that still runs in reasonable time. That makes negatives **50% of the corpus**, which is roughly
twice what feels natural when you are thinking about defect coverage, and it is the single most
important sizing decision in the world.

The honest consequence: per-decoy-class rates have `n ≈ 21`. Two false positives out of 21 is a
point estimate of 9.5% with an exact Clopper-Pearson interval of **[1.2%, 30.4%]** — asymmetric,
and wide enough that only very large per-class differences are detectable. Those rates get reported
with their intervals and described as directional. Pretending otherwise is exactly the failure this
course exists to prevent, so the eval harness refuses to print a per-class rate without its
interval.

---

## 6a. The development slice

`world/slices/dev150.json` — a committed manifest of PR ids, not a runtime sampler. Generated once,
hashed, and version-controlled, so any two runs against `--slice dev150` are comparable by
construction rather than by discipline.

| Class | Full corpus | dev150 |
|---|---|---|
| MECH | 60 | 15 |
| SEM | 130 | 33 |
| CTX | 110 | 27 |
| DEC | 150 | 37 |
| CLEAN | 150 | 38 |
| **Total** | **600** | **150** |
| *negatives* | *300* | *75* |

Proportional stratification, not uniform sampling. Uniform draws would let class balance drift
between runs and would throw away the false-positive resolution §6 sized the corpus to buy.

**Aggregate false positive rate only.** At 75 negatives the per-class cells are single digits, where
the §6 refuse-to-print rule would fire on every run — and a guardrail that fires constantly is one
you learn to disable. Per-class rates print at full-corpus gates and nowhere else.

**Resolution, stated honestly.** 75 negatives gives an aggregate FP half-width of ±6.8pp against the
full corpus's ±3.4pp. Enough to see a real move; not enough to call a two-point difference. The
harness prints the interval alongside every estimate, on both slice sizes.

---

## 7. Smoke report

`make world-report` after generation. Fails loudly rather than warning quietly.

- PR counts by family and class against the `facts.yaml` targets
- Description-fidelity, diff-size, and blast-radius distributions
- Pre-existing noise distribution across touched modules
- Defect placement across modules — flags any module holding more than 15% of one class
- Achievable-precision table: the §6 calculation run on the corpus that was actually produced
- **Leakage check** — a bag-of-words classifier trained on diffs to predict defect class. Above
  ~60% accuracy the generator has a lexical tell, the corpus is easier than it looks, and every
  downstream number is inflated. This check is why the generator does AST transforms instead of
  templated text insertion.

---

## 8. Determinism

- One seed produces one corpus, byte for byte.
- Injection is AST transformation, never text patching, so every defect is syntactically valid and
  exactly locatable.
- Model-written PR descriptions are generated once, cached, and committed.
- `ground_truth.json` records `facts_hash` and `service_rev`; the harness refuses to score a run
  whose corpus hash does not match the manifest.

## 9. Adversarial bucket

`world/prs/adversarial/` is built in Chapter 12, not here, and uses the same manifest format with
one added field: `attack_class` from the OWASP Agentic Top 10. It is generated after Chapter 8's
error analysis, so the attacks target failure modes the agent has actually demonstrated rather than
ones imagined in advance.
