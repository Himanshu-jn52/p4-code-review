# p4-code-review

Automated code review for **Perforce changelists**, using multiple specialized agents with
confidence-based scoring. It drives the [`p4mcp-server`] tools and  Swarm reviews.

## What it does

Given a changelist number, the `/p4-code-review` command runs a multi-agent pipeline:

1. **Precheck** (haiku) — skip if the changelist is empty, already finished-reviewing, trivial/automated, or Claude has already commented on its Swarm review.
2. **Gather CLAUDE.md** (haiku) — collect depot paths of CLAUDE.md files that govern the changed files.
3. **Summarize** (sonnet) — describe what the changelist does.
4. **Review** (4 parallel agents) — 2 sonnet agents for CLAUDE.md compliance + 2 opus agents for bugs/logic/security, tuned for **high-signal-only** findings.
5. **Validate** (parallel subagents) — each candidate issue is independently re-checked against the full file content, not just the diff.
6. **Filter** — drop anything that didn't survive validation.
7. **Report** — print findings to the terminal.
8–9. **Comment** (opt-in) — with `--comment`, post inline + summary comments to the associated Swarm review.

## Usage

```
/p4-code-review:run <changelist>            # review and print findings only
/p4-code-review:run <changelist> --comment  # also post comments to the Swarm review
```

Examples:

```
/p4-code-review:run 2095
/p4-code-review:run 2095 --comment
```

## Requirements

- A Perforce MCP server connected to Claude Code, exposing the `query_changelists`,
  `query_files`, `query_reviews`, and `modify_reviews` tools.
- **The server name is not hardcoded.** The command resolves your Perforce MCP server at
  runtime by finding whichever connected server exposes those tools, so it works regardless
  of the name you registered it under (`p4mcp`, `p4mcp`, `p4`, `helix`, …). No edit
  required. The `allowed-tools` frontmatter is keyed to the default `p4mcp` name for
  auto-approval only — under a different name the tools still work, they just prompt for
  permission. To silence those prompts, edit the `mcp__p4mcp__` prefixes in
  `commands/run.md` to match your server.
- `--comment` mode requires a Helix Swarm review associated with the changelist, and
  write permission on that review. If no review exists, the command prints findings and
  stops rather than creating one.

## Installation

This repo is its own plugin marketplace. Add it, then install the plugin:

```
/plugin marketplace add Himanshu-jn52/p4-claude-review
/plugin install p4-code-review@p4-code-review
```

To test it locally without a marketplace, point Claude Code straight at the directory:

```
claude --plugin-dir /path/to/p4-code-review
```

Repository layout:

```
p4-code-review/
├── .claude-plugin/
│   ├── plugin.json        # plugin manifest
│   └── marketplace.json   # marketplace catalog (lists this plugin)
├── commands/run.md
├── LICENSE
└── README.md
```

## License

[MIT](LICENSE).

[`p4mcp-server`]: https://github.com/perforce/p4mcp-server
