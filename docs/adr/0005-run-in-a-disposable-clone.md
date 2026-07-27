# Run every Attempt in a Disposable Clone

Autonomous execution requires `atx custom def exec … -x -t`. AWS documents that
`--trust-all-tools` "bypasses most security guardrails", and that the one guardrail which
otherwise survives it — the `alwaysPromptCommands` deny list, covering patterns like
`rm -rf *` and `sudo *` — is **not enforced in non-interactive mode**. So in exactly the
configuration this plugin requires, ATX runs arbitrary shell commands as the user, with the
user's AWS credentials, with no backstop. Every Attempt therefore runs in a Disposable Clone
in scratch space, never in the user's working copy.

## Consequences

This bounds the *repository*, not the *machine*: `$HOME`, other repositories and ambient
credentials remain reachable by a misbehaving run. Users who need the machine bounded should
run `atx` in a container — AWS publishes an image for this at
`public.ecr.aws/d9h8z6l7/aws-transform:latest` — and/or use a dedicated AWS profile carrying
only `transform-custom:*` rather than day-to-day credentials.

Both were considered for the default and rejected: a container adds a Docker dependency to a
plugin whose whole premise is zero dependencies, and a scoped profile requires IAM setup the
plugin cannot perform for the user.
