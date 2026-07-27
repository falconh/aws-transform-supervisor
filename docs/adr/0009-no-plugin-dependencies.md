# Declare no plugin dependencies

Claude Code supports declaring plugin dependencies, and Claude Code **auto-installs** them —
`claude plugin prune` exists specifically to remove "auto-installed plugin dependencies… that
Claude Code pulled in to satisfy another plugin's `dependencies` field". A sibling plugin in
the same marketplace (`terraform-module-steering`) uses this. This plugin deliberately declares
none, and bundles no MCP server.

Three candidates were considered and rejected:

- **`awslabs.aws-transform-mcp-server`** — none of its 19 tools touches a local file
  (ADR 0008), so declaring it would load 19 tool schemas into every turn for every user,
  permanently, in exchange for no capability we can use.
- **`aws-transform@agent-plugins-for-aws`** — bundles that same MCP server, so it carries the
  identical cost, and additionally requires adding a new marketplace to the catalog's
  `allowCrossMarketplaceDependenciesOn` list.
- **`aws-core`** — the closest call. `run-migration` preflight reaches AWS's `aws-transform`
  skill through its `retrieve_skill` tool when handing off fleet work, and
  `claude-plugins-official` is already an allowed dependency source, so this would have been
  frictionless. Rejected because it buys reliability for one **out-of-scope** edge case
  (ADR 0007) at the cost of every user auto-installing an MCP proxy, and because it would make
  the plugin harness-dependent — the property that lets prose-only skills run anywhere
  (ADR 0004).

## Consequences

The only hard requirements are the `atx` CLI and AWS credentials, both handled in preflight.
Installation stays a single plugin with no transitive footprint.

The fleet handoff is therefore a **soft** dependency: it uses `retrieve_skill` where the AWS
MCP tools are present and falls back to naming the alternative where they are not. That
degradation is deliberate, and preflight is written to expect it.

Revisit if this plugin ever uses `aws-core` on the main path rather than at a handoff — for
instance calling `search_documentation` to diagnose obscure `atx` failures. A dependency that
serves the common path earns its install; one that serves an out-of-scope branch does not.
