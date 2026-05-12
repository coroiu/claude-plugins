# claude-plugins

Personal [Claude Code](https://code.claude.com) plugin marketplace.

## Install the marketplace

```
/plugin marketplace add coroiu/claude-plugins
```

Then install plugins from it:

```
/plugin install confluence-edit@coroiu
```

## Plugins

### `confluence-edit`

Edits Confluence pages in ADF (Atlassian Document Format) **without** round-tripping through markdown, so inline-comment anchors, layouts, and other ADF features survive the edit.

Why it exists: the MCP Atlassian server's `updateConfluencePage` tool with `contentFormat: "markdown"` rebuilds ADF from markdown, which has no way to represent the `annotation` marks that anchor inline comments. Every comment on the page becomes "dangling" after such an edit. This plugin teaches Claude to reach for a small ADF-native CLI instead.

**Bundled CLI:** [`bin/confluence-edit`](plugins/confluence-edit/bin/confluence-edit) — Python 3, stdlib only. Auto-added to `PATH` when the plugin is enabled.

**Configure** (one-time): generate an Atlassian API token at <https://id.atlassian.com/manage-profile/security/api-tokens>, then export in your shell rc:

```bash
export CONFLUENCE_EMAIL="<you>@example.com"
export CONFLUENCE_API_TOKEN="<token>"
export CONFLUENCE_BASE_URL="https://<your-org>.atlassian.net"
```

See [the skill](plugins/confluence-edit/skills/confluence-edit/SKILL.md) for the full workflow guidance Claude follows.
