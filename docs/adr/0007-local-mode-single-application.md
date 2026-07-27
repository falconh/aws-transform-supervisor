# Local mode only, one application at a time

This plugin drives `atx` locally against a single Target. It does not submit remote jobs.
AWS's own threshold is local execution for 1–9 repositories and remote AWS Batch/Fargate for
10 or more; supporting remote would pull in CDK infrastructure deployment, an S3 result
layout, Secrets Manager credential handling for private clones, and an entirely separate
monitoring path via Lambda status polling.

## Consequences

Bulk migration is explicitly out of scope. Users with fleets should use AWS's own
`aws-transform` skill, which already does remote orchestration well.

Supervision, as this plugin defines it, is inherently per-run: watching one Conversation,
deciding whether to resume or nudge it, and reading one validation summary. That does not
obviously generalise to 128 concurrent jobs, so remote support would be a redesign rather
than an extension.

## The scope test (added in 0.1.6)

The sharper boundary is not "repository versus environment" but **reversibility**: work belongs
here when a failed Attempt can be thrown away by deleting a branch.

This is why `atx` should not migrate a database even though nothing technically stops it — it
has unrestricted shell under `-t`, so given credentials it could run the DDL. The objection is
that ADR 0006's safety model evaporates. Attempts are siblings off a Base Commit precisely
because a bad one costs nothing to discard; once 400 GB has moved into Aurora there is no
"Attempt 2 from the Base Commit". Agentic transformation is tolerable because it is idempotent
and re-runnable, and data migration is neither. The blast radius compounds it: `-x -t` runs
without the `alwaysPromptCommands` deny list (ADR 0005), which is survivable in a Disposable
Clone and not survivable against a production database.

The test cuts *within* a job, not just between jobs. A SQL Server → Aurora migration splits into
schema and data movement (irreversible, out of scope) and application code adaptation —
`UseSqlServer()` → `UseNpgsql()`, ADO.NET class swaps, T-SQL in embedded queries — which is
ordinary reversible file rewriting and belongs here. AWS's managed workflow runs that as its own
step, which is the evidence the seam is real rather than invented.
