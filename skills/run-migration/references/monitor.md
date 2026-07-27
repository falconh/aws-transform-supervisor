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
is addressed by it — and report it to the user immediately.

> **CRITICAL: Get the Conversation id from that stdout line.** Never find it by
> modification time. `ls -t ~/.aws/atx/custom/ | head -1` will return a previous run's
> directory and every downstream conclusion will be about the wrong Attempt.

## 4. Poll

Every 60 seconds, fixed interval — no backoff:

```bash
kill -0 "$(cat "$ATX_WORK/attempt-N.pid")" 2>/dev/null && echo RUNNING || echo DONE
```

While `RUNNING`, read new lines from the Conversation log and relay meaningful progress —
planning, files changed, build results, errors. Keep polling without waiting for user input;
the user should see continuous progress.

Additional detail when a run looks stuck, spawned by ATX's own subagents:

```
~/.aws/atx/custom/<conversation-id>/logs/subagents/<name>.log
```

For CLI-level faults rather than transformation faults: `~/.aws/atx/logs/debug*.log` and
`~/.aws/atx/logs/error.log`.

> **CRITICAL: `kill -0` is the only liveness check.** A stale `attempt-N.exit` can survive
> from an earlier run. Its existence proves nothing.

> **CRITICAL: Never echo `Thinking` lines.** They are spinner frames, repeated dozens of
> times. Surface everything else.

## 5. Decide it is finished

> **CRITICAL: Completion is process exit, and nothing else.** ATX prints
> `TRANSFORMATION COMPLETE` and then continues working — validation summary generation still
> follows. Treating that line as the end reads the summary before it is written.

Only once `kill -0` reports `DONE`:

```bash
cat "$ATX_WORK/attempt-N.exit"
```

`0` is success. Non-zero is failure. Now go to [react.md](react.md).

## 6. One more parser trap

> **CRITICAL: No commas in `additionalPlanContext`.** They break the CLI parser. Rephrase
> rather than escaping. When building any `atx` command, quote JSON and configuration values
> with single quotes, never nest double quotes inside double quotes, and check the quoting
> balances before running it.
