---
name: run-migration
description: >-
  Supervise an AWS Transform (`atx`) migration run: launch it non-interactively, monitor a
  long-running job, resume it when Agent Minutes run out, and turn the exit criteria it could not
  meet into a remediation plan. Use when the user wants to run, migrate, monitor, resume, or report
  on an atx transformation of an application. Not for authoring the Recipe (use author-recipe), not
  for fleet or remote migrations across many repositories (use AWS's own aws-transform skill), and
  not for AWS Transform's managed console workflows such as mainframe or VMware migration.
---

# Run a supervised AWS Transform migration

Drives one `atx` transformation of one application, watches it, intervenes in exactly two
sanctioned ways, and writes a remediation plan for whatever it could not finish.

`../../CONTEXT.md` holds the vocabulary — Recipe, Attempt, Base Commit, Nudge, Leftover.
Paths below are relative to this skill's own directory.

## Guardrails

These hold at every step. Each defends against a signal that looks authoritative and is not.

1. **The process is the only witness.** Completion is process exit, checked with
   `kill -0 <pid>`. ATX prints `TRANSFORMATION COMPLETE` and then keeps working — validation
   summary generation still follows — and a stale `.exit` file can survive from an earlier
   run. Both of those lie. The process does not.
2. **The Conversation id comes from stdout**, off the `Conversation log:` line. Modification
   time picks the wrong run: `ls -t` returns a previous Attempt's directory, and every
   conclusion drawn afterwards describes the wrong run.
3. **Relay signal, skip spinner frames.** Surface planning, file changes, build results and
   errors. `Thinking` lines repeat dozens of times.
4. **Write `additionalPlanContext` as comma-free prose.** Commas break the CLI parser.

## Steps

Each step ends on its completion criterion. Move on only once it holds.

1. **Preflight** — [references/preflight.md](references/preflight.md).
   *Done when:* every check passes and the flag table has been reconciled against
   `atx custom def exec --help`.

2. **Confirm the Recipe.** It must be published and declare Exit Criteria. If none exists,
   stop and hand off to `author-recipe`.
   *Done when:* the Recipe appears in `atx custom def list --json` and its Exit Criteria have
   been read — they bound everything the Leftover report can say.

3. **Prepare the Disposable Clone** — [references/monitor.md](references/monitor.md).
   *Done when:* the clone exists in scratch space, the Base Commit is written to a file, and
   the Attempt branch is checked out.

4. **Launch and monitor** — [references/monitor.md](references/monitor.md).
   *Done when:* `kill -0` reports the process gone and its exit code has been read
   (guardrail 1).

5. **React** — [references/react.md](references/react.md).
   *Done when:* the outcome is accept, resume or nudge. For anything else, report and stop.

6. **Report** — [references/leftovers.md](references/leftovers.md).
   *Done when:* `LEFTOVERS.md` exists and **every** unmet Exit Criterion in it carries a
   concrete remediation naming the change to make.

## Boundaries

Deliberate, each recorded in `../../docs/adr/`.

- Recipe refinement belongs to ATX's Continual Learning (ADR 0002).
- Stopping a run is the user's call — report what the logs show and let them decide.
- Attempt branches are evidence; each persists on its own branch off the Base Commit
  (ADR 0006).
- Every Attempt runs in a Disposable Clone, because autonomous mode has no shell backstop:
  `-t` bypasses most guardrails by AWS's own description, and the one that survives it —
  the `alwaysPromptCommands` deny list in `~/.aws/atx/trust-settings.yaml`, covering patterns
  like `rm -rf *` and `sudo *` — is **not enforced under `-x`**. So `-x -t` runs arbitrary
  shell as the user, with their AWS credentials. The clone bounds the repository; it does not
  bound the machine (ADR 0005).
- ATX's validation summary is the verdict; this skill reports it rather than re-auditing it
  (ADR 0003).

## Cost

`atx` bills **Agent Minutes**. A resume continues an existing Conversation; a Nudge starts a
fresh one and bills from zero. Say which is about to happen before it happens, and point at
<https://aws.amazon.com/transform/pricing/> rather than quoting prices.

Cap an unfamiliar Recipe with `--limit <agent-minutes>`. Reaching the ceiling ends the
Conversation cleanly and resumably, which beats an open-ended run.
