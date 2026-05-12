---
name: confluence-edit
description: Edit a Confluence page while preserving inline-comment anchors, layouts, and other ADF features that the regular MCP `updateConfluencePage` tool would strip via markdown round-trip. Use this skill when (a) the page has open inline comments that need to stay attached after the edit, (b) the edit is small and you want to avoid passing a huge ADF body as a tool argument, or (c) the user explicitly mentions preserving anchors. Wraps a local CLI tool `confluence-edit` (in `~/.local/bin`).
---

# confluence-edit

## When to use

Any edit to a Confluence page where one or more of these apply:

- The page has **open inline comments**. Markdown round-trip via `updateConfluencePage` with `contentFormat: "markdown"` strips the `annotation` marks that anchor those comments to text, leaving them "dangling" in the comment list with no visual anchor on the page.
- The edit is **small** (a typo, a function-name swap, a paragraph rewrite) and you'd rather not push 80KB+ of ADF JSON through a tool call.
- You want to verify anchors **before and after** the edit (the `list-anchors` subcommand).

If none of these apply — fresh page, no comments, big rewrite — the standard `mcp__plugin_atlassian_atlassian__updateConfluencePage` with markdown is fine.

## Why ADF, not markdown

Confluence stores inline-comment anchors as `{"type": "annotation", "attrs": {"id": "<uuid>", "annotationType": "inlineComment"}}` marks on text nodes inside ADF (Atlassian Document Format). The markdown representation has no equivalent. So:

- **markdown → ADF**: the API conversion fabricates ADF from your markdown. Annotation marks aren't reconstructed (markdown didn't carry them). All inline-comment anchors become dangling.
- **ADF → ADF**: round-trip is lossless. Annotation marks are preserved unless you explicitly delete the text node they sit on.

This means you can **change the highlighted text itself** without breaking the anchor — just keep the mark on the replacement text node. The anchor is the mark, not the text content.

## Tool: `confluence-edit`

CLI lives at `~/.local/bin/confluence-edit` (Python 3, stdlib only).

### Auth setup (one-time)

User generates an API token at https://id.atlassian.com/manage-profile/security/api-tokens, then exports three env vars (in their shell rc):

```bash
export CONFLUENCE_EMAIL="<user>@bitwarden.com"
export CONFLUENCE_API_TOKEN="<token>"
export CONFLUENCE_BASE_URL="https://bitwarden.atlassian.net"
```

If env vars are missing, the tool exits with a clear message pointing at the token-generation URL.

### Commands

| Command | What it does |
| --- | --- |
| `confluence-edit fetch <id> [-o FILE]` | Pulls page ADF + metadata. With `-o`, writes to file; without, prints to stdout. |
| `confluence-edit push <id> <file> [-m MSG]` | Uploads a modified ADF body. Auto-increments version. |
| `confluence-edit replace-text <id> <old> <new> [--dry-run] [-m MSG]` | Walks every text node and replaces `<old>` with `<new>`, preserving marks (including annotation marks). One-shot edit; pushes automatically unless `--dry-run`. |
| `confluence-edit apply-patch <id> <patch.py> [--dry-run] [-m MSG]` | Loads a Python file that defines `patch(body: dict) -> dict`, applies it to the ADF body, and pushes. |
| `confluence-edit list-anchors <id>` | Prints every inline-comment annotation ID and the text fragment(s) it covers. Use before and after edits to verify the anchor set is unchanged. |

### Workflows

**Quick text swap on annotated text:**
```bash
confluence-edit list-anchors 2923724969     # baseline
confluence-edit replace-text 2923724969 'to_rfc3339()' 'to_rfc3339_opts(SecondsFormat::Millis, true)' -m "resolve comment 3"
confluence-edit list-anchors 2923724969     # verify same set of IDs
```

**Larger surgical edit (paragraph rewrite, multiple swaps, etc.):**
```bash
confluence-edit fetch 2923724969 -o page.json
# edit page.json with whatever tool — preserve "marks" arrays on text nodes you want anchored
confluence-edit push 2923724969 page.json -m "rewrite assumption #2"
```

**Programmatic transform:**
```python
# patch.py
def patch(body):
    # walk and modify nodes...
    return body
```
```bash
confluence-edit apply-patch 2923724969 patch.py --dry-run    # inspect first
confluence-edit apply-patch 2923724969 patch.py -m "..."
```

## Patterns

- **Always `list-anchors` before pushing** if the page has open comments. Compare against your post-edit `list-anchors` to confirm no anchor IDs disappeared.
- **Mark preservation is per-text-node.** If you want to change the highlighted span (e.g. rename a function inside an anchored code span), edit the `text` field of that node and **leave its `marks` array alone**. The anchor moves with the new text.
- **Don't delete annotated text nodes** unless you're OK with the comment becoming dangling. Splitting the node and putting the mark on one side is fine.
- **Confirm before pushing** when the user has flagged anchor-fragility. Show what changed and ask before invoking `push` / non-dry-run edits.

## Anti-patterns

- ❌ Calling `updateConfluencePage` with `contentFormat: "markdown"` on a page that has open inline comments. **Strips all annotation marks.** Comments end up dangling.
- ❌ Passing 80KB+ of ADF JSON inline as a tool-call argument when `confluence-edit replace-text` or `apply-patch` would do. Bloats the conversation context for no benefit.
- ❌ Deleting an annotated text node "to clean up" — the comment loses its anchor. Either keep at least one text node with the mark, or accept that the anchor will dangle (and tell the user).

## Known MCP tool gotcha

The `mcp__plugin_atlassian_atlassian__updateConfluencePage` tool's `contentFormat` description claims `"html"` is supported and "round-trip safe and preserves inline comments." The enum, however, only accepts `"markdown"` and `"adf"`. Trying `"html"` returns `InputValidationError`. If you need round-trip safety, **use `adf` via this skill's CLI**, not the misleading `html` value.
