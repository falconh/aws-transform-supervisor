# Preflight

Run once per session, before the first Attempt. Do not repeat per Attempt.

## 0. Is this the right tool?

`atx` transforms **one git repository you can clone and build locally**. Its unit of work is a
buildable project, not an environment. Check this before anything else, because the later
checks give actively misleading advice otherwise — a database migration reaching step 7 gets
offered `git init`, which is nonsense for a database.

Hand off when the target is an **environment** rather than a repo:

| The user is migrating | Send them to |
|---|---|
| A live database (SQL Server → Aurora, schema + data + waves) | AWS Transform managed service |
| A VMware estate (discovery, dependency mapping, wave planning) | AWS Transform managed service |
| A mainframe estate (decomposition, business-rule extraction, traceability) | AWS Transform managed service |
| .NET Framework across many repos, needing private NuGet resolution | AWS Transform managed service |
| The same transformation across many repos | AWS's own `aws-transform` skill (remote mode) |

Make the handoff as concrete as the destination allows.

**Fleet work** (many repos, one transformation) is the one case that can be handed off
directly rather than described: AWS's own `aws-transform` skill does remote Batch/Fargate
orchestration, and it is reachable in-session wherever the AWS MCP tools are present — call
`retrieve_skill` with skill name `aws-transform` and follow it from there. Handing off is not
the same as calling it mid-loop, which ADR 0001 rules out: that rule exists because its
greet-and-wait and per-run confirm gates break an unattended run. Here the run is ending, so
those gates are appropriate rather than obstructive.

**Environment work** cannot be handed off in-session — it needs a plugin the user may not have:

```
/plugin install aws-transform@agent-plugins-for-aws
```

Name it, say plainly why this tool is wrong for the job, and stop. Do not attempt the work
here, and do not offer to `git init` an environment. Note the one real overlap: .NET *is* transformable by `atx` on a
single repo (AWS's own VB6-to-Blazor and Lambda-runtime examples do exactly that). The split is
scale, not language — one buildable repo belongs here; a cross-repo port with private packages
and approval gates belongs to the managed service.

## 1. Platform

```bash
uname -s
```

`Linux` or `Darwin` → continue. Anything Windows-like, or the command failing → **stop**.
AWS Transform custom does not support native Windows. Tell the user to install WSL
(`wsl --install` in an elevated PowerShell, then restart) and re-run from a WSL terminal.

## 2. AWS CLI and credentials

```bash
aws --version
aws sts get-caller-identity
```

If `aws` is missing: `brew install awscli` on macOS, or the bundled installer on Linux.

If credentials fail, have the user configure them — `aws configure`, `~/.aws/credentials`,
or `AWS_PROFILE`. Prefer the persistent forms: variables set with `export` do not survive
into a new shell, and this skill spawns background processes.

**Never** echo, log or display access keys, session tokens or secrets.

## 3. ATX CLI

```bash
atx --version
```

If missing, install it — this is the only supported install path:

```bash
curl -fsSL https://transform-cli.awsstatic.com/install.sh | bash
```

Then update unconditionally. Do not ask whether to, and do not skip it because the CLI
"looks current" — this also refreshes the available Recipes:

```bash
atx update
```

## 4. Permissions

This doubles as the permission check — if it succeeds, the caller has what local mode needs:

```bash
atx custom def list --json
```

On a permissions error the caller needs `transform-custom:*`. The AWS-managed policy
`AWSTransformCustomFullAccess` grants it. Explain what is needed and get explicit
confirmation before attaching anything. For SSO identities (role names starting
`AWSReservedSSO_`) it cannot be attached directly — it has to go into the IAM Identity
Center permission set, which is an administrator's job.

Read the JSON yourself to confirm the Recipe is present. Do not pipe it through `jq` or a
script.

## 5. Derive the current flags

**Read the current flags off the machine**, treating `--help` as the source of truth over
memory or this document. ATX ships frequently and its interface moves:

```bash
atx custom def exec --help
```

Confirm the flags this skill relies on still exist and still mean what is assumed:

| Purpose | Expected flag |
|---|---|
| Recipe name | `-n` / `--transformation-name` |
| Target path | `-p` / `--code-repository-path` |
| Build/validation command | `-c` / `--build-command` |
| Non-interactive | `-x` / `--non-interactive` |
| Trust all tools | `-t` / `--trust-all-tools` |
| Plan context | `-g` / `--configuration` |
| Agent Minutes ceiling | `--limit` |
| Recipe version | `--tv` / `--transformation-version` |

If any differ, follow `--help` and tell the user what changed.

## 6. Target runtime

The runtime the Recipe targets must be the active one, or builds and validation inside the
transformation will fail for reasons that have nothing to do with the migration. Check
whichever applies (`java -version`, `python3 --version`, `node --version`, …).

If the target version is installed but not active, switch to it (`export JAVA_HOME=…`,
`pyenv shell …`, `nvm use …`) and re-check. If it is not installed at all, **ask before
installing anything**.

## 7. Target is a git repository

ATX writes checkpoint commits as it works, so the Target must be a git repository. Confirm
with `git -C <target> status`. If it is not one, ask the user before running `git init` —
never initialise a repository in someone's directory unannounced.

Optionally attribute those checkpoints:

```bash
export ATX_GIT_COMMITTER_NAME="<name>"
export ATX_GIT_COMMITTER_EMAIL="<email>"
```

Unset, they are attributed to `ATX Bot <checkpoint@atx.bot>`.

## 8. Long builds

If the Target's build regularly exceeds 15 minutes, raise the shell timeout before
launching — the default is 900 seconds:

```bash
export ATX_SHELL_TIMEOUT=1800
```
