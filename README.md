# aws-transform-supervisor

Run an AWS Transform (`atx`) migration **under supervision**: publish a Recipe you own, watch
the run, intervene only to resume or nudge it, and turn the criteria it could not meet into a
written remediation plan.

A Claude Code + Codex plugin. Prose only — no scripts, no runtime dependencies.

## Why this exists

AWS already ships an official `aws-transform` agent skill that matches repositories to
transformations, executes them locally or across a fleet, and monitors progress. **If that is
what you need, use it** — it does that job well.

This plugin deliberately owns only the two things it does not do:

- **Supervision** — watching a long-running `atx` job and reacting when it exhausts its
  Agent Minutes budget or fails, without a human sitting on it.
- **Leftovers** — turning the exit criteria ATX could not meet into an actionable plan.

It does not call AWS's skill at run time; that skill requires user confirmation before every
execution, which is incompatible with an unattended loop. See
[ADR 0001](docs/adr/0001-decouple-from-the-aws-transform-skill.md).

## Skills

| Skill | Use it for |
|---|---|
| `run-migration` | Run one migration, monitor it, react, report Leftovers |
| `author-recipe` | Write, test and publish a custom Recipe with real exit criteria |

## How it works

```
preflight → publish Recipe → clone to scratch → Attempt on its own branch
    → poll until the process exits → accept | resume | nudge → LEFTOVERS.md
```

Each Attempt runs in a **Disposable Clone**, on a branch off a recorded base commit. Nothing
runs in your working copy, and no attempt is ever reset or discarded.

## Design decisions worth knowing before you use it

These are deliberate. Each is recorded in [`docs/adr/`](docs/adr/).

- **This plugin never refines your Recipe.** ATX's own Continual Learning does that. As a
  consequence runs are **not reproducible** — the same Recipe can behave differently as
  lessons accumulate ([ADR 0002](docs/adr/0002-cede-recipe-refinement-to-atx.md)).
- **ATX grades its own work.** The Leftover report is built from ATX's validation summary,
  not from independent verification. That makes the exit criteria in your Recipe the entire
  quality mechanism ([ADR 0003](docs/adr/0003-trust-atx-validation-summary.md)).
- **Autonomous runs have no shell guardrail.** `-x -t` is required for automation, and
  `--non-interactive` disables the `alwaysPromptCommands` deny list that otherwise survives
  `--trust-all-tools`. The Disposable Clone bounds the repository, not the machine
  ([ADR 0005](docs/adr/0005-run-in-a-disposable-clone.md)).
- **Local mode, one application.** Fleet migrations are out of scope
  ([ADR 0007](docs/adr/0007-local-mode-single-application.md)).

## Requirements

- macOS or Linux (native Windows unsupported — use WSL)
- AWS CLI with credentials, and `transform-custom:*` IAM permission
- The ATX CLI: `curl -fsSL https://transform-cli.awsstatic.com/install.sh | bash`
- The runtime the migration targets, active on `PATH`
- A git repository as the target

## Cost

`atx` bills **Agent Minutes**. A nudge starts a fresh conversation and bills from zero; a
resume continues an existing one and does not. Cap unfamiliar Recipes with
`--limit <agent-minutes>`. Pricing: <https://aws.amazon.com/transform/pricing/>.

## Telemetry

Runs include `--telemetry` by default. Ask to opt out and it is dropped for the session.
AWS's opt-out documentation:
<https://docs.aws.amazon.com/transform/latest/userguide/transform-usage-telemetry.html>

## Status

**0.1.0 — unreleased and unverified against a live `atx` installation.** The design is built
on AWS's published documentation and its shipped agent skill; the procedures have not yet
been exercised end to end against a real transformation. Treat the first run as a test of
this plugin as much as of your Recipe.

## Vocabulary

[`CONTEXT.md`](CONTEXT.md) is the glossary — Recipe, Attempt, Base Commit, Nudge, Leftover.

## License

MIT
