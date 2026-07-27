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
