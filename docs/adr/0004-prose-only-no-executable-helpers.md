# Prose only — no executable helpers

The supervision loop is fiddly: capture a conversation id from stdout, poll a pid, correlate
artifacts, avoid several documented traps. A shell or Python helper would make that
deterministic and testable. We ship prose instructions instead, so the plugin has zero
runtime dependencies and behaves identically under Claude Code, Codex, or any other harness
that can read a `SKILL.md` and run a shell command.

## Consequences

Nothing here can be unit-tested; correctness rests on how the instructions are written. The
mitigation is the technique AWS documents for authoring Transformation Definitions, applied
to our own skill text: mark must-follow rules with a literal `CRITICAL:` prefix and give
exact command strings rather than describing them, which measurably reduces variability.

The five traps in `run-migration/references/monitor.md` exist because they are precisely the
mistakes a model re-derives under pressure — trusting a "complete" log line, trusting a stale
exit file, or finding the wrong Conversation by modification time.
