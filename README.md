# Shireito (司令塔)

Commander pattern for Claude Code. Bundles 5 specialized subagents plus orchestration rules and interactive permission setup.

> 日本語版: [README.ja.md](./README.ja.md)

## What you get

- **5 subagents** (`agents/`):
  - `explorer` — fast codebase mapping (haiku)
  - `code-analyst` — deep architecture analysis (sonnet)
  - `code-reviewer` — security and quality review, read-only by design (sonnet)
  - `implementer` — code writing, modification, refactoring (sonnet, has Edit/Write)
  - `debugger` — root-cause analysis and minimal fixes (sonnet, has Edit)
- **`/shireito:orchestrate`** skill — loads orchestration rules (parallel vs sequential, worktree isolation, permission gotchas) into the commander's context when planning multi-subagent work
- **`/shireito:setup`** skill — walks you through configuring `.claude/settings.json` with the right permissions and worktree path for the current project

## Install

In Claude Code:

```
/plugin marketplace add ijust/shireito
/plugin install shireito
```

Then, in each project where you want to use shireito:

```
/shireito:setup
```

The setup skill detects the project root, asks where you want worktrees and which permission strategy you prefer, and merges the result into `.claude/settings.json`. It does **not** touch your other settings.

## Why a setup skill instead of shipping permissions

Subagents do not inherit the parent session's `permissions.allow`. Worktrees live at paths outside the original repo, so path-scoped rules like `Edit(/path/to/repo/**)` do not cover them. As a result, write-capable subagents (`implementer`, `debugger`) running in parallel hit permission prompts and effectively serialize.

The right configuration depends on your project's absolute paths, so it cannot ship as a plugin file. The setup skill writes it once, project-specific.

## Quick start

After install + setup, ask Claude in the project:

> Use the explorer subagent to map the authentication module, then code-analyst to evaluate its design.

The commander will fire both subagents (explorer in parallel with code-analyst since they're independent reads), aggregate the results, and report back.

For implementation work that needs parallelism:

> Implement feature A and feature B in parallel. Use the implementer subagent in separate worktrees.

The commander will create worktrees, dispatch one `implementer` per worktree, and integrate after.

If you're not sure when to use which subagent, run `/shireito:orchestrate` to load the decision rules into context.

## The commander pattern in one diagram

```
[Main Claude Code session: commander (司令塔)]
   ├─ Agent("explorer", "...")     ← parallel-safe (read-only)
   ├─ Agent("code-analyst", "...") ← parallel-safe (read-only)
   └─ Agent("implementer", isolation: "worktree", "...")  ← sequential or isolated parallel
       ↓
   commander aggregates and reports to the human
```

The human only talks to the commander. The commander figures out what to delegate, when to run in parallel, and how to integrate results.

## Files

```
shireito/
├── .claude-plugin/plugin.json     # plugin manifest
├── agents/                        # 5 subagent definitions
│   ├── explorer.md
│   ├── code-analyst.md
│   ├── code-reviewer.md
│   ├── implementer.md
│   └── debugger.md
├── skills/
│   ├── setup/SKILL.md             # interactive permission setup
│   └── orchestrate/SKILL.md       # orchestration rules
└── README.md
```

## License

MIT

## Author

[ijust (Yoshishige Tsuji)](https://github.com/ijust)
