# Cede Recipe refinement to ATX's Continual Learning

ATX already extracts Lessons from every Attempt and applies them automatically to later
ones. Rather than run a competing refinement loop that edits the Recipe between Attempts,
this plugin cedes refinement entirely to ATX: the Recipe is authored once, published, and
left alone. The alternative — editing the Recipe in git while ATX also learns server-side —
would mean two hidden inputs changing behaviour at once, so an improvement between Attempts
could never be attributed to either. We preferred a simple, honest tool over an
unattributable feedback loop.

## Consequences

Attempts are deliberately **not** reproducible: the same Recipe against the same Base Commit
can behave differently as Lessons accumulate. This is accepted, not a defect. Learning is
therefore left on — no `--do-not-learn`, auto-approval left enabled.

One limitation to know: ATX derives Lessons from developer feedback in interactive mode *and*
from code issues it hits. This plugin runs non-interactively, so the first source is
structurally unavailable and refinement will be slower and narrower than AWS's docs imply.
