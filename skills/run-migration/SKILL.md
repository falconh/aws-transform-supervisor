---
name: run-migration
description: >-
  Use when running an AWS Transform (atx) code migration or upgrade on an application and you want
  the run supervised rather than fired and forgotten: launching a transformation non-interactively,
  watching a long-running atx job, reacting when it exhausts its Agent Minutes budget or fails, and
  turning the criteria it could not meet into a written remediation plan. Triggers on "run the
  transformation", "migrate this app with AWS Transform", "atx is still running", "what did the
  transformation not finish", or any request to monitor, resume, or report on an atx run. Not for
  authoring the Recipe itself (use author-recipe), not for bulk/remote fleet migrations across many
  repositories (use AWS's own aws-transform skill), and not for AWS Transform's managed console
  workflows like mainframe or VMware migration.
---

# Run a supervised AWS Transform migration

Drives one `atx` transformation of one application, watches it, intervenes in exactly two
sanctioned ways, and writes a remediation plan for whatever it could not finish.

Read `../../CONTEXT.md` for the vocabulary used throughout (Recipe, Attempt, Base Commit,
Leftover, Nudge). Paths below are relative to this skill's own directory.

## The loop

1. **Preflight** — verify tooling, credentials, permissions and runtime, and derive the
   current CLI flags. See [references/preflight.md](references/preflight.md).
2. **Confirm the Recipe** — it must already be published and must declare Exit Criteria. If
   it does not exist yet, stop and use the `author-recipe` skill first.
3. **Prepare the Disposable Clone** — clone the Target to scratch, record the Base Commit,
   create the branch for Attempt 1. See [references/monitor.md](references/monitor.md).
4. **Launch and monitor** — start the Attempt in the background, capture its Conversation id
   and pid, then poll. See [references/monitor.md](references/monitor.md).
5. **React** — on exit, read the exit code and decide between accept, resume and nudge.
   Only those. See [references/react.md](references/react.md).
6. **Report** — read the validation summary, write `LEFTOVERS.md`, present the result.
   See [references/leftovers.md](references/leftovers.md).

## Non-negotiable rules

These are the mistakes that get re-derived under pressure. The full form of each, with exact
commands, is in [references/monitor.md](references/monitor.md).

1. **CRITICAL: Completion is process exit, never log text.** ATX prints
   `TRANSFORMATION COMPLETE` and then keeps working — it still has validation summary
   generation to do. Never treat any log line as completion.
2. **CRITICAL: `kill -0 <pid>` is the only liveness check.** An exit-code file may be stale
   from a previous run. Never conclude a run finished because that file exists.
3. **CRITICAL: Get the Conversation id from the `Conversation log:` line in stdout.** Never
   locate it by modification time — `ls -t` will happily hand you a previous run's directory.
4. **CRITICAL: Never echo `Thinking` lines.** They are spinner frames and repeat dozens of
   times. Relay everything else.
5. **CRITICAL: No commas anywhere in `additionalPlanContext`.** They break the CLI parser.
   Rephrase to avoid them.

## What this skill will not do

Bounded on purpose — see `../../docs/adr/`.

- **Never edits the Recipe.** Refinement belongs to ATX's Continual Learning (ADR 0002).
- **Never kills a running Attempt.** Only the user does that.
- **Never resets or discards an Attempt branch.** Attempts are siblings off the Base
  Commit (ADR 0006).
- **Never runs in the user's working copy.** Every Attempt runs in a Disposable Clone
  (ADR 0005).
- **Never independently re-verifies ATX's work.** The validation summary is the verdict
  (ADR 0003).

## Cost and consent

`atx` bills **Agent Minutes**, and a Nudge starts a fresh Conversation that bills from zero
rather than continuing an existing one. Before launching the first Attempt, tell the user
what is about to run and roughly what it will touch. Before any Nudge, say plainly that it
is a new billable run. Never quote prices — point at
<https://aws.amazon.com/transform/pricing/>.

Consider setting a ceiling with `--limit <agent-minutes>` on the first Attempt of an
unfamiliar Recipe. Hitting the ceiling ends the Conversation cleanly and is resumable, which
is a much better failure mode than an open-ended run.
