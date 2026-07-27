# Trigger evals

Twenty queries per skill testing one thing: does the description fire on the right requests
and stay quiet on the wrong ones. Half should trigger, half should not, and the negatives are
deliberately near-misses rather than easy rejects — `"review the diff on my atx/attempt-2
branch"` (mentions atx, is actually code review), `"our sagemaker batch transform job is
failing"` (keyword collision on *transform*), `"im writing a claude code skill … cant get the
description right"` (also about `SKILL.md`, nothing to do with Recipes).

## Results (2026-07-27, claude-opus-5, 3 runs per query, 40% held out)

| Skill | Precision | Recall | Held-out |
|---|---|---|---|
| `run-migration` | **100%** | 17% | 14/24 |
| `author-recipe` | **100%** | 8% | 13/24 |

**Precision is perfect and recall is poor.** Every negative passed, including all the planted
near-misses. Neither skill ever fires when it shouldn't; both under-fire when they should.

`scripts/run_loop` was then given that signal and asked to do better. Over three iterations on
`run-migration` it produced longer, trigger-richer rewrites and **every one scored worse**:

| Candidate | Train | Held-out |
|---|---|---|
| Original (shipped) | 23/36, P=100% R=28% | **14/24** |
| Rewrite 1 | 19/36, P=67% R=11% | 13/24 |
| Rewrite 2 | 22/36, P=100% R=22% | 12/24, R=0% |

It selected the original (`best == original`). The `author-recipe` loop crashed in its third
improve step on a transient `claude -p exited 1`; its one completed rewrite scored 14/24
against the original's 13/24 — a single query out of 24, on an identical train score, which is
noise. **Both descriptions therefore ship unchanged**, on evidence rather than by default.

## What the recall number does and does not mean

The harness scores a bare query with no repo context, no conversation history, and no tools.
Real use has all three: the user is inside the repo, mid-conversation, having already mentioned
`atx`. Skill-creator's own guidance notes that Claude declines to consult a skill for work it
believes it can handle directly, which is exactly how an isolated one-line question reads.

So treat 17% as a floor, not an estimate. The practical consequence: **these skills may need
naming explicitly** (`/aws-transform-supervisor:run-migration`) rather than being discovered.
That is a real limitation, and it is the honest reading — four attempted rewrites failed to
move it, which is evidence the cause is not description wording.

## Does the skill beat no skill?

A separate A/B, because trigger accuracy says nothing about whether the skill *helps* once it
fires. Eight probes on the load-bearing facts, asked with the `run-migration` text in context
and without it, 3 runs each.

Two caveats on the numbers, both mine. The first attempt scored the with-skill arm 0/24 —
a bug: the prompt begins with YAML frontmatter (`---`) and `claude -p` parsed it as a CLI
flag, so every call returned empty with exit 1. Passing the prompt on stdin fixed it. The
regex grader then produced false negatives on three probes, marking correct answers wrong for
quoting the anti-pattern in order to warn against it (`ls -t`), or for saying "read-only"
instead of "cannot be modified". **Treat the aggregate scores as unreliable; the per-probe
readings below were verified by hand.**

What survives that:

- **The skill fixes the fact that gates everything.** Baseline named the CLI correctly 1/3;
  with the skill, 3/3. A model that thinks the command is `aws transform …` never starts.
- **The baseline is stronger than expected.** It already knew to wait for process exit rather
  than the `TRANSFORMATION COMPLETE` line, that `AWS/*` definitions are read-only, the comma
  restriction, and `validation_summary.md`. AWS Transform is in training data — the skill's
  value is narrower than "teaches the model ATX".
- **It found a real gap.** Asked whether `alwaysPromptCommands` still applies under `-x`, the
  with-skill run answered: *"the `run-migration` skill doesn't cover `trust-settings.yaml` or
  `alwaysPromptCommands` at all."* Correct — that fact lived only in ADR 0005 and the README,
  which an executing agent never reads. Fixed in 0.1.1 by moving the reasoning into the
  `Boundaries` section and `monitor.md`, so the Disposable Clone carries its justification
  rather than a bare cross-reference.

The honest summary: on the facts measured, the skill's demonstrated marginal value over a
capable baseline is **narrow but load-bearing** — it wins where being wrong stops the run
dead. Its broader value (leftover planning, branch-per-attempt, autonomy without AWS's
confirm gates) is unmeasured, and cannot be measured without a live `atx`.

## Re-running

```bash
cd <skill-creator-path>
python3 -m scripts.run_eval \
  --eval-set <repo>/evals/run-migration-triggers.json \
  --skill-path <repo>/skills/run-migration \
  --model claude-opus-5 --runs-per-query 3 --verbose
```

Swap `run_eval` for `run_loop` (adding `--max-iterations` and `--holdout`) to search for a
better description. Note that the eval set here was authored without the interactive review
step skill-creator recommends — worth a human pass before trusting a future result.
