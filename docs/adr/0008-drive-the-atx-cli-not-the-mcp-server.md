# Drive the `atx` CLI, not the AWS Transform MCP server

AWS publishes an official MCP server (`awslabs.aws-transform-mcp-server`, 19 tools) and an
official Claude Code + Codex plugin (`aws-transform@agent-plugins-for-aws`) that bundles it.
This plugin uses neither, and shells out to the `atx` CLI instead. The reason is not
preference: the MCP server's own README states it supports "managed-service workspaces, jobs,
connectors, and HITL tasks only — **not** custom transformation definitions or the atx CLI
workflow." Its tools (`create_workspace`, `create_job`, `complete_task`, `create_connector`,
`upload_artifact`) all operate on a managed job running in AWS. None touches a local file.

Everything this plugin does is local: a Recipe executed against a working tree, a build command
run on the machine, checkpoint commits in a git repo, a Disposable Clone, an Attempt branch, a
`validation_summary.md` written under `~/.aws/atx/custom/<conversation-id>/`. There is no MCP
tool for any of it.

Calling the service API directly was also considered and is closed: `transform-custom` is not
in the public AWS CLI/SDK surface (`aws transform-custom` does not resolve), so neither the AWS
CLI nor an AWS MCP proxy can reach it.

## Consequences

The `atx` binary is a hard dependency, and its absence is a hard stop — which is why preflight
installs it rather than degrading.

The division of labour with AWS's own tooling is clean rather than competitive. Their plugin
covers the managed service (.NET, mainframe, VMware, SQL Server); their README lists the `atx`
CLI as a separate prerequisite "only required for custom transformations", which is exactly
this plugin's scope. Anyone needing managed-service migrations should install AWS's plugin —
not an extension of this one (see ADR 0007).

One MCP relationship does apply, in the opposite direction: `atx` is itself an MCP *client*
(`~/.aws/atx/mcp.json`, `atx mcp tools`), so a Recipe's transformation agent can be given extra
tools during a run. That is the only place MCP belongs in this design.
