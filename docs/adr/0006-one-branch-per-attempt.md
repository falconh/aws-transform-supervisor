# One branch per Attempt, never reset

ATX writes checkpoint commits into the repository as it works, so a failed Attempt does not
leave a clean tree — it leaves partially transformed code, already committed. Each Attempt
therefore branches from the recorded Base Commit and keeps its own history; nothing is reset
and nothing is discarded.

## Consequences

Attempts are siblings, not a chain, so each one starts from identical ground and they can be
diffed against each other and against the Base Commit. That comparison is the raw material
for the Leftover report.

The two rejected alternatives both had real costs. Hard-resetting between Attempts throws
away genuinely useful partial work. Letting a Nudge continue on top of a failed Attempt is
worse than it sounds: a Nudge starts a *new* Conversation with no memory of the previous one,
so ATX would inspect half-migrated code and may take it for the intended starting state.

The cost we accept is branch clutter in the Disposable Clone, which is discarded anyway.
