# Launch and monitor an Attempt

## 1. Prepare the Disposable Clone

Every Attempt runs in scratch space, never in the user's working copy (ADR 0005). Do this
once per migration:

```bash
ATX_WORK="${TMPDIR:-/tmp}/atx-supervisor/<target-name>-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$ATX_WORK"
git clone <target-path-or-url> "$ATX_WORK/repo"
git -C "$ATX_WORK/repo" rev-parse HEAD > "$ATX_WORK/base-commit"
```

Record `$ATX_WORK` and report it to the user — it is where all their results live.

Then, for Attempt N, branch from the Base Commit. Never from the previous Attempt:

```bash
git -C "$ATX_WORK/repo" checkout -b "atx/attempt-N" "$(cat "$ATX_WORK/base-commit")"
```

## 2. Launch in the background

`atx` runs for a long time and must not block. Write a runner script and background *that*,
so the exit code is captured when ATX finishes:

```bash
cat > "$ATX_WORK/run-N.sh" << 'RUNNER'
#!/bin/bash
atx custom def exec -n <recipe-name> -p <clone-repo-path> -c "<build-command>" -x -t \
  --telemetry "client=claudecode,agent=claude,executionMode=local"
echo $? > "<work-dir>/attempt-N.exit"
RUNNER
chmod +x "$ATX_WORK/run-N.sh"
nohup "$ATX_WORK/run-N.sh" > "$ATX_WORK/attempt-N.log" 2>&1 &
echo $! > "$ATX_WORK/attempt-N.pid"
cat "$ATX_WORK/attempt-N.pid"
```

Substitute the placeholders — the heredoc is quoted (`<< 'RUNNER'`), so nothing expands
inside it.

Add `-g 'additionalPlanContext=…'` only for a Nudge. Add `--limit <n>` to cap Agent Minutes.
Drop `--telemetry` if the user asks to opt out; comply immediately and without argument.

## 3. Capture the Conversation id

**As soon as you have the pid, move on — do not wait for the user.** ATX prints the
Conversation log path within 30–60 seconds:

```bash
grep "Conversation log:" "$ATX_WORK/attempt-N.log"
```

It looks like:

```
Conversation log: /Users/<user>/.aws/atx/custom/20260319_063712_e3479843/logs/2026-03-19T06-37-26-conversation.log
```

If it has not appeared, wait 15 seconds and retry, up to four times. Extract both the full
log path and the Conversation id (`20260319_063712_e3479843`). Save the id — every artifact
is addressed by it — and report it to the user immediately (guardrail 2).

## 4. Poll

Every 60 seconds, fixed interval — no backoff:

```bash
kill -0 "$(cat "$ATX_WORK/attempt-N.pid")" 2>/dev/null && echo RUNNING || echo DONE
```

While `RUNNING`, read new lines from the Conversation log and relay progress (guardrail 3).
Keep polling without waiting for user input; the user should see continuous progress.

Additional detail when a run looks stuck, spawned by ATX's own subagents:

```
~/.aws/atx/custom/<conversation-id>/logs/subagents/<name>.log
```

For CLI-level faults rather than transformation faults: `~/.aws/atx/logs/debug*.log` and
`~/.aws/atx/logs/error.log`.

## 5. Decide it is finished

Only once `kill -0` reports `DONE` (guardrail 1):

```bash
cat "$ATX_WORK/attempt-N.exit"
```

`0` is success. Non-zero is failure. Now go to [react.md](react.md).

## 6. Shell quoting

Quote JSON and configuration values with single quotes, keep double quotes unnested, and
check that quoting balances before running any `atx` command — unbalanced quotes hang the
shell waiting for a terminator.
