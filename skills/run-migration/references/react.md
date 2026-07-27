# React

The Attempt has exited and its exit code has been read. There are exactly **three**
permitted outcomes. This is a whitelist, not a starting point — if the situation fits none
of them, report to the user and stop.

## Accept — exit code 0

Go to [leftovers.md](leftovers.md). A zero exit means ATX finished; it does **not** mean it
met every Exit Criterion. Partial success is the normal case, and the validation summary is
where that gets settled.

## Resume — Agent Minutes budget exhausted

Look for ATX's budget message in the Attempt log:

```
⚠️ Budget limit reached: 30.00 / 30.00 Agent Minutes. Exiting.
```

This is not a failure. The Conversation is intact and continues with full context — which
makes resuming far cheaper than re-running, and the only intervention that keeps ATX's
accumulated state.

Tell the user the ceiling was hit and what the new one will be, then:

```bash
atx --conversation-id <conversation-id> -t --limit <higher-limit>
```

Background it and monitor exactly as before, reusing the same Attempt branch and the same
Conversation id — this continues the Attempt rather than starting a new one.

Agent Minutes accumulate across the interruption, so the new `--limit` must exceed the total
already consumed, not the increment you want to add.

**Conversations expire 30 days after creation.** Past that, resume is unavailable and the
only route forward is a fresh Attempt.

## Nudge — failure worth one more shot with better context

A Nudge is a **new Attempt** with adjusted plan context. It is the weaker intervention and
carries real cost:

- It starts a **new Conversation** with no memory of the failed one.
- It bills Agent Minutes **from zero**.
- It gets a **new branch off the Base Commit** — never the failed Attempt's branch, and
  never a reset (ADR 0006).

Worth doing when the failure is explicable and context would plausibly fix it: a wrong
target version, a build command that needed different arguments, a constraint ATX could not
have known. Not worth doing when the failure is environmental (credentials, network,
missing runtime) — fix the environment and re-run instead.

```bash
git -C "$ATX_WORK/repo" checkout -b "atx/attempt-<N+1>" "$(cat "$ATX_WORK/base-commit")"
```

Then relaunch as in [monitor.md](monitor.md), adding:

```bash
-g 'additionalPlanContext=<comma-free guidance>'
```

Because a Nudge is billable and re-runs from scratch, say so plainly before starting one.

**Cap it at two Nudges.** If a third looks warranted, the problem is the Recipe rather than
the context — stop, and report that the Recipe needs revision through `author-recipe`.
