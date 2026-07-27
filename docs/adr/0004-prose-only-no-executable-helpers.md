# Prose only — no executable helpers

The supervision loop is fiddly: capture a conversation id from stdout, poll a pid, correlate
artifacts, avoid several documented traps. A shell or Python helper would make that
deterministic and testable. We ship prose instructions instead, so the plugin has zero
runtime dependencies and behaves identically under Claude Code, Codex, or any other harness
that can read a `SKILL.md` and run a shell command.

## Consequences

Nothing here can be unit-tested; correctness rests on how the instructions are written. The
mitigation is the technique AWS documents for authoring Transformation Definitions, applied
to our own skill text: give exact command strings rather than describing them, and state each
rule positively so the behaviour we want is the one named.

The guardrails at the top of `run-migration/SKILL.md` carry the load. They exist because each
defends against a signal that looks authoritative and is not — a "complete" log line, a stale
exit file, the newest directory by modification time — and those are precisely the mistakes a
model re-derives under pressure. They live in one place so that changing a rule is a
one-place edit; the reference files point at them by number rather than restating them.
