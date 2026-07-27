# Preflight

Run once per session, before the first Attempt. Do not repeat per Attempt.

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

**Do not hardcode flags from memory or from this document.** ATX ships frequently and its
interface moves. Read the current truth off the machine:

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
