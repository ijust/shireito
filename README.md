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

When `/plugin install` prompts for a scope, choose **User** to make shireito available in every project you open. Pick **Project** if you want the install committed to this repo's `.claude/` so your team gets it via git. **Local** is rarely the right choice.

Then, in each project where you want to use shireito, run:

```
/shireito:setup
```

The setup skill detects the project root, asks where you want worktrees and which permission strategy you prefer, and merges the result into `.claude/settings.json`. It does **not** touch your other settings.

## Quick start

After install + setup, ask Claude in the project:

> Use the explorer subagent to map the authentication module, then code-analyst to evaluate its design.

The commander will fire both subagents (explorer in parallel with code-analyst since they're independent reads), aggregate the results, and report back.

For implementation work that needs parallelism:

> Implement feature A and feature B in parallel. Use the implementer subagent in separate worktrees.

The commander will create worktrees, dispatch one `implementer` per worktree, and integrate after.

### When to invoke `/shireito:orchestrate` yourself

The `orchestrate` skill is auto-invocable — the commander may load it on its own when it recognizes a multi-subagent planning task — but **auto-invocation is not guaranteed**. The model decides based on description match, and the decision can miss. Invoke it yourself when:

- You're about to spawn parallel `implementer` subagents in separate worktrees
- Permission prompts are stalling parallel subagent runs (likely the permission inheritance trap)
- You want the full decision rules in context upfront, before committing to an approach
- You're new to the commander pattern and want to skim the operational rules

```
/shireito:orchestrate
```

Explicit invocation guarantees the rules are loaded; auto-invocation does not.

## Usage examples

Concrete prompts to try once shireito is installed and `/shireito:setup` has been run in your project. Copy-paste and adjust the file paths to your codebase.

### Mapping a codebase (explorer)

> Use the explorer subagent to map this codebase. Summarize each top-level directory in one line.

Fast haiku-powered overview without flooding your main context with file contents.

### Architecture review (code-analyst)

> Use code-analyst to evaluate the design of `src/auth.py`. Are responsibilities split cleanly? What would you change?

Deep sonnet-powered analysis. Read-only — surfaces issues, does not edit.

### Parallel implementation in isolated worktrees (implementer × N) — the killer use case

> Implement two features in parallel, each with `isolation: "worktree"`:
> - Feature A: add `--format=json` to `cli.py`
> - Feature B: add `--filter=<glob>` to `cli.py`

The commander dispatches two `implementer` subagents at once. Each runs in its own auto-created worktree, so editing the same file from both does not conflict. After both finish, review the worktree diffs and integrate. **This is where spec-driven development meets true parallelism.**

### Post-change code review (code-reviewer)

> Use code-reviewer on the last 2 commits. Check security, code quality, and engineering principles.

Read-only by design: never edits, only flags.

### Root-cause debugging (debugger)

> The test in `tests/test_foo.py::test_edge_case` fails. Use the debugger subagent to find the root cause and apply the minimal fix.

`debugger` has Edit access for the fix and reports root cause, evidence, the fix applied, and a prevention recommendation.

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

## Why this combination is fast (spec-driven + worktree + subagents)

Spec-driven development (Kiro, cc-sdd, or any `.kiro/specs/<feature>/{requirements,design,tasks}.md` flavor) splits a feature into discrete, well-defined work units before any code is written. With shireito and git worktrees, the commander dispatches several of those units to parallel `implementer` subagents at once, each in its own isolated worktree.

The wins:

- **N features → N parallel implementers.** Throughput scales with how many independent specs you have ready, not with how fast one person can type.
- **No file-collision serialization.** Subagents in separate worktrees can edit the same files without conflict; conflicts get resolved at integration time, not blocked at edit time.
- **Reviews and debugging also run in parallel.** Spin up one `code-reviewer` per worktree, or one `debugger` that owns a failing worktree without touching the others.
- **Specs survive across sessions.** Months later, a fresh subagent reads the same spec and the implementation context is restored. No re-explaining the requirements.

The combination turns spec-driven development from "writing specs is overhead" into "writing specs unlocks parallelism." The commander only needs enough independent specs queued up to keep N implementer worktrees busy.

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

## Why a setup skill instead of shipping permissions

Subagents do not inherit the parent session's `permissions.allow`. Worktrees live at paths outside the original repo, so path-scoped rules like `Edit(/path/to/repo/**)` do not cover them. As a result, write-capable subagents (`implementer`, `debugger`) running in parallel hit permission prompts and effectively serialize.

The right configuration depends on your project's absolute paths, so it cannot ship as a plugin file. The setup skill writes it once, project-specific.

## Attribution

The subagent set in `agents/` was originally inspired by the example subagents in Anthropic's official Claude Code documentation (https://docs.claude.com/en/docs/claude-code/sub-agents), particularly the `code-reviewer` and `debugger` patterns. The definitions in this repository have been substantially restructured and extended for the commander-pattern orchestration this plugin distributes. The `orchestrate` and `setup` skills, the 5-subagent curation, and the overall plugin structure are original.

## License

MIT

## Author

[ijust (Yoshishige Tsuji)](https://github.com/ijust)
