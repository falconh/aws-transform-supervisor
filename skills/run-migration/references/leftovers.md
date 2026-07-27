# Leftovers and the remediation plan

ATX reports which Exit Criteria it did not meet. This step reads that report, turns each
unmet criterion into a Leftover with a proposed remediation, and writes it down.

## 1. Read the validation summary

Addressed by the Conversation id captured at launch (guardrail 2) — a Leftover report built
from the wrong run is worse than none:

```bash
cat ~/.aws/atx/custom/<conversation-id>/artifacts/validation_summary.md
```

Overall verdict:

```bash
grep "OVERALL STATUS" ~/.aws/atx/custom/<conversation-id>/artifacts/validation_summary.md
```

A richer report may sit alongside it:

```bash
ls ~/.aws/atx/custom/<conversation-id>/artifacts/
```

If the file is absent, treat that as a finding rather than something to work around: confirm
the process actually exited (`kill -0`), then report that the summary was not produced and
give the user the Attempt log path.

## 2. Establish what actually changed

```bash
git -C "$ATX_WORK/repo" diff --stat "$(cat "$ATX_WORK/base-commit")" HEAD
git -C "$ATX_WORK/repo" log --oneline "$(cat "$ATX_WORK/base-commit")"..HEAD
```

This is context for the report, not a second verdict. This plugin does not re-adjudicate
ATX's own assessment (ADR 0003).

## 3. Write `LEFTOVERS.md`

Write it into the Attempt branch, so it travels with the code it describes:

```
$ATX_WORK/repo/LEFTOVERS.md
```

Do **not** use a heredoc — heredoc blocks can hang in some shells. Use the Write tool, or
`printf '%s'`.

```markdown
# Leftovers — <recipe-name>

- **Attempt**: atx/attempt-N
- **Conversation**: <conversation-id>
- **Base commit**: <sha>
- **Overall status**: <from validation summary>
- **Exit code**: <n>

## Unmet exit criteria

### 1. <the criterion, quoted from the Recipe>

- **ATX reported**: <what the validation summary says about it>
- **Affected files**: <paths, where identified>
- **Why it is left over**: <ATX's stated reason, or "not stated">
- **Proposed remediation**: <concrete next step — the change to make, not "investigate">
- **Estimated shape**: <mechanical | needs judgement | needs a decision from the team>

### 2. …

## Met criteria

<brief list, so the report stands alone as a record of the run>

## Notes

<anything from the logs that would help whoever picks this up — recurring build errors,
constraints ATX flagged, decisions it deferred>
```

## 4. Quality bar for a remediation

A Leftover is only useful if the next step is actionable. "Investigate the failing test" is
not a remediation; "the migration left `OrderRepository` on the old client API — port the
three call sites to the new builder pattern" is.

Where the summary genuinely does not say enough to propose a step, say so explicitly rather
than inventing one. Reading the Conversation log around the failure usually helps:

```bash
grep -n -i -B 5 -A 20 "<criterion keyword>" ~/.aws/atx/custom/<conversation-id>/logs/*-conversation.log
```

## 5. Report

Tell the user:

- Where the work is: `$ATX_WORK/repo`, branch `atx/attempt-N`
- Overall status and how many criteria were met of how many
- The Leftover count and their shape (mechanical vs needs judgement)
- The Conversation id, and that it is resumable for 30 days
- How to review: `git -C "$ATX_WORK/repo" diff <base> HEAD`
- How to keep it: the Disposable Clone is scratch space, so merging or pushing the Attempt
  branch somewhere durable is the user's next move

Leave the Disposable Clone in place, and ask before merging anything into the user's working
copy.

## Note on trust

Every Leftover here is ATX's own account of its work. This plugin deliberately does not
independently verify it (ADR 0003), which means a criterion ATX believed it met may not
actually be met. Say so when presenting the report — it is a summary of what ATX reported,
not an audit.
