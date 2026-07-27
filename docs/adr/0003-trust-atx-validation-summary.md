# Trust ATX's validation summary as the Leftover verdict

ATX writes a validation summary artifact per Conversation containing its code changes,
files touched, validation results and an overall status. This plugin treats that file as
the authoritative source of Leftovers rather than independently verifying the Target
against its own acceptance checks. Independent verification was considered and rejected as
scope we did not want to own — it would require a second, parallel definition of "done"
living outside the Recipe.

## Consequences

The Leftover report inherits ATX's blind spots: anything ATX silently omits from its own
summary is silently omitted from ours. We are accepting an agent's self-assessment as the
completion verdict, with eyes open.

This makes **Exit Criteria the entire quality mechanism**. Because nothing downstream
re-checks ATX's work, the concreteness of the Exit Criteria declared in a Recipe is the only
lever on how useful the Leftover report is. A vague Recipe produces a useless report. This is
why `author-recipe` treats Exit Criteria as mandatory rather than advisory.
