# application-transform-manager

Run an AWS Transform (`atx`) migration **under supervision**: publish a Recipe you own, watch
the run, intervene only to resume or nudge it, and turn the criteria it could not meet into a
written remediation plan.

A Claude Code + Codex plugin. Prose only — no scripts, no runtime dependencies.

## Why this exists

AWS ships plenty here already. **Check these first — if one fits, use it:**

| AWS ships | Covers | Get it |
|---|---|---|
| `aws-transform` agent skill | Repo→transformation matching, local and fleet execution, monitoring | via the `aws-core` plugin |
| `aws-transform` agent plugin (Claude Code + Codex) | The **managed** service: .NET, mainframe, VMware, SQL Server | `/plugin install aws-transform@agent-plugins-for-aws` |
| `awslabs.aws-transform-mcp-server` | Managed workspaces, jobs, connectors, HITL tasks | bundled with the plugin above |

This plugin owns only what none of them do: **supervising one local custom transformation and
planning its leftovers.** Watching a long-running `atx` job and reacting when it exhausts its
Agent Minutes budget or fails, without a human sitting on it — then turning the exit criteria
ATX could not meet into an actionable plan.

The boundary is AWS's own: their plugin lists the `atx` CLI as a separate prerequisite "only
required for custom transformations", which is exactly this plugin's scope. Managed-service
migrations belong to their plugin, not an extension of this one
([ADR 0007](docs/adr/0007-local-mode-single-application.md)).

Two deliberate non-uses:

- **AWS's agent skill is not called at run time** — it requires user confirmation before every
  execution, which defeats an unattended loop
  ([ADR 0001](docs/adr/0001-decouple-from-the-aws-transform-skill.md)).
- **The MCP server is not used at all** — it is managed-workspace-only by its own
  documentation, and none of its 19 tools touches a local file
  ([ADR 0008](docs/adr/0008-drive-the-atx-cli-not-the-mcp-server.md)).

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

Those three are **optional companions, not dependencies** — this plugin declares none and
installs nothing transitively. `aws-core` makes the fleet handoff reach AWS's skill directly
rather than just naming it; everything else works without any of them
([ADR 0009](docs/adr/0009-no-plugin-dependencies.md)).

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

**0.2.0 — unreleased and unverified against a live `atx` installation.** The design is built
on AWS's published documentation and its shipped agent skill; the procedures have not yet
been exercised end to end against a real transformation. Treat the first run as a test of
this plugin as much as of your Recipe.

Renamed from `aws-transform-supervisor` in 0.2.0, before any release was published or
installed — hence a clean break rather than a compatibility problem.

**Known limitation — these skills under-trigger.** Measured trigger accuracy is 100% precision
(they never fire on the wrong request, across 20 hard near-miss queries) but only 8–17% recall
in isolation. Four attempted description rewrites all scored worse, so the cause is not
wording. Until that improves, name the skill explicitly
(`/application-transform-manager:run-migration`) rather than relying on discovery. Method,
numbers and the full argument are in [`evals/README.md`](evals/README.md).

## Vocabulary

[`CONTEXT.md`](CONTEXT.md) is the glossary — Recipe, Attempt, Base Commit, Nudge, Leftover.

## License

MIT
