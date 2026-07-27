---
name: author-recipe
description: >-
  Author a custom AWS Transform Recipe — a Transformation Definition — for a migration AWS's
  built-in transformations do not cover: write it, declare its exit criteria, test it as a draft,
  and publish it to the registry. Use when the user wants to create, revise, test or publish an atx
  transformation definition, or when no built-in transformation fits a framework migration, API or
  logging swap, language port, or org-specific upgrade. Not for running a migration (use
  run-migration); AWS-managed transformations cannot be modified.
---

# Author a Recipe

A Recipe is a directory — a `SKILL.md` plus a `references/` folder — that `atx` publishes to
your account's registry and executes. This skill writes one, tests it, and publishes it.

Read `../../CONTEXT.md` for vocabulary. Paths below are relative to this skill's directory.

## Before anything: does one already exist?

```bash
atx custom def list --json
```

Read the JSON yourself. AWS ships transformations for language upgrades, SDK migrations,
framework upgrades, Graviton, and codebase analysis. If one fits, **use it** — do not author
a competing Recipe.

> **AWS-managed transformations (`AWS/…`) cannot be modified.** If one nearly fits, the only
> customisation available is plan context passed at run time via `additionalPlanContext`.
> That is a run-time nudge, not a Recipe. Say so plainly rather than implying the Recipe can
> be tuned.

## The one hard requirement: Exit Criteria

Because nothing downstream re-checks ATX's work (ADR 0003), the Exit Criteria declared in
the Recipe are the **entire** quality mechanism. ATX validates against them and reports which
went unmet, and that report is the whole Leftover story. A vague Recipe produces a useless
report.

**Publish only a Recipe whose Exit Criteria are concrete and checkable.** See
[references/exit-criteria.md](references/exit-criteria.md).

## Structure

Start from [`../../templates/recipe/`](../../templates/recipe/). Copy it and rename
`SKILL.md.template` to `SKILL.md`:

```
my-recipe/
├── SKILL.md          # objective, phases, exit criteria, constraints
└── references/       # migration guides, before/after examples, API docs
```

`references/` limits, which are enforced by the service:

- **Text only** — `.md`, `.html`, `.txt`, source files. No PDF, PNG or DOCX. Extract the text
  from those instead.
- **10 MB total** across all files.
- Prefer a few well-named files over many small ones; concatenate where sensible.

The highest-value reference material is before/after example code, then human-readable
migration guides, then API documentation.

## Writing style that survives execution

AWS's guidance for Recipe text, which materially affects run-to-run consistency:

- Mark must-follow rules with a literal `CRITICAL:` or `IMPORTANT:` prefix.
- Give **exact literal strings** for commands and values, wrapped in backticks, rather than
  describing them. Vague instructions produce varied results.
- State source and target explicitly — "upgrade from X to Y", not "modernise".
- Decompose. Several small phases beat one large instruction.
- Supply a deterministic build or validation command. This is what lets ATX check its own
  work, and what feeds its Continual Learning.

## Test before publishing

Save as a draft first — drafts are versioned and executable, and are the point of iteration:

```bash
atx custom def save-draft -n <name> --description "<description>" --sd ./my-recipe
```

Note the version id it returns. Run the draft against a sample Target with the
`run-migration` skill, using `--tv <version-id>` and a modest `--limit`. Read the validation
summary: if the Exit Criteria did not produce a clear verdict, the criteria need work — fix
those before anything else.

Iterate on the files and re-save as needed.

## Publish

Only once a draft has actually run and its Exit Criteria produced a meaningful report:

```bash
atx custom def publish -n <name> --description "<description>" --sd ./my-recipe
```

Or publish a tested draft by version:

```bash
atx custom def publish -n <name> --tv <version-id>
```

Confirm with `atx custom def list`. Published Recipes are visible to anyone in the account
with the right IAM permissions, so treat publishing as a team-visible act and confirm with
the user first.

## Keep the Recipe in git

The Recipe directory belongs in version control — it is reviewable prose that changes how a
codebase gets rewritten. `atx custom def get -n <name> --td <dir>` downloads a published
Recipe back to disk, so an existing one can be brought under version control.

## What this skill does not do

- **Does not iterate the Recipe from run feedback.** Once published, improvement is ATX's
  Continual Learning, not an edit loop (ADR 0002). Author deliberately, revise deliberately.
- **Does not modify AWS-managed transformations.** Not permitted by the service.
- **Does not delete Recipes.** `atx custom def delete` is permanent; leave it to the user.
