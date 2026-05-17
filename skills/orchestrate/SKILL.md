---
name: orchestrate
description: Load this skill before orchestrating multi-step or multi-subagent work. Teaches the commander when to delegate to subagents in parallel vs sequentially, why write-capable subagents need git worktree isolation, and how to avoid the subagent permission inheritance trap. Also useful when the user asks about orchestration, or when a parallel subagent run is stalling on permissions.
allowed-tools: Read, Grep, Glob, Bash
---

# Shireito (司令塔) Pattern

The main Claude Code session is the **commander** (司令塔). It delegates focused tasks to the 5 specialized subagents via the `Agent` tool and aggregates their text results back into the main session. For worktree-level changes (e.g., parallel `implementer` runs in isolated worktrees), the human reviews each worktree and merges manually.

> Scope: this skill covers the `Agent` subagent system inside a single Claude Code session. The experimental multi-session feature gated by `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is a different mechanism and not in scope here.

```
[Main session: commander]
   ├─ Agent("explorer", "...")     ← parallel-safe
   ├─ Agent("code-analyst", "...") ← parallel-safe
   └─ Agent("implementer", "...")  ← sequential (depends on others)
       ↓
   commander aggregates and reports
```

## The 5 subagents

| Name | Model | Tools | Role |
|---|---|---|---|
| `explorer` | haiku | Read, Grep, Glob, Bash | Fast codebase mapping and file location |
| `code-analyst` | sonnet | Read, Grep, Glob, Bash | Deep architecture and design analysis |
| `code-reviewer` | sonnet | Read, Grep, Glob, Bash | Security and quality review (read-only by design) |
| `implementer` | sonnet | Read, **Edit, Write**, Bash, Grep, Glob | Writes and modifies code |
| `debugger` | sonnet | Read, **Edit**, Bash, Grep, Glob | Root-cause analysis and minimal fixes |

`explorer` is haiku for speed and cost on broad searches. The other four are sonnet for reasoning depth. `code-reviewer` deliberately lacks Edit/Write so it can only *suggest* fixes — never apply them silently.

## Commander responsibilities

- Decompose the user's request into tasks
- Pick the right subagent for each task (`description`-driven auto-delegation works once descriptions are tight)
- Decide parallel vs sequential
- Aggregate subagent results
- Report to the user

## MUST norms for the commander

The commander (the main session, typically running on opus) MUST follow these rules. Users may call out violations.

- **Delegate all code changes to a subagent.** The commander never invokes `Edit` / `Write` / `NotebookEdit` directly. Use `implementer` (sonnet) for implementation, `debugger` (sonnet) when the change is paired with investigation.
- **Delegate broad exploration, design review, and code review the same way.** Searches that span more than 1–2 files go to `explorer` (haiku). Design evaluation goes to `code-analyst` (sonnet). Reviewing a diff goes to `code-reviewer` (sonnet). The commander running `Grep` / `Read` directly is reserved for targeted 1–2 file checks.
- **Exceptions the commander handles directly:**
  - 1–2 line trivial changes (typos, comments) where subagent startup cost outweighs the work
  - Targeted reads needed to compose a delegate prompt
  - State checks: `git status`, `git diff`, `git log`, and similar
  - Conversation with the user, planning, and result aggregation
- **Be conscious of model selection.** Opus plans / aggregates / decides. Sonnet implements / reviews. Haiku explores. The `Agent` tool's `model` argument can override the subagent's frontmatter on a per-call basis; default to the frontmatter setting.
- **Write delegate prompts as self-contained.** Subagents do not see the parent's history. Every prompt must include the background, the goal, which files are in scope, and the shape of the expected deliverable.

## Subagent responsibilities

- One task, one purpose
- Run in an isolated context (parent history is not inherited)
- Return a final text message to the commander

## Parallel vs sequential — the decision rule

**Parallel-safe (run them together)**:
- Independent read-only investigations (explorer + code-analyst + code-reviewer on different angles)
- Backend vs frontend exploration
- Multiple independent review findings, each in its own subagent
- **Long-running prep paired with its consumer.** If a downstream step (verification data pulls, fixture builds, dataset snapshots, build warmup) sits on the critical path, fire it as a background subagent in the **same turn** you start the work that will consume it. Don't serialize "implement → then realize we need data → spawn prep". The implementer can finish while the prep is still running; integrate when both return.

**Sequential (chain them)**:
- design → implement → test
- explorer → implementer (the implementer needs the explorer's findings)
- Anything writing to the same file or branch

**Don't subagent at all**:
- 1–2 file edits that the commander can do directly. Subagent overhead (separate context, no parent history) costs more than the parallelism gains for trivial work.

## Write-capable subagents need worktree isolation

**Rule**: any subagent that uses Edit or Write (`implementer`, `debugger`, custom write-capable agents) should run in an **isolated git worktree**. This applies whether you run one or many in parallel.

Why:
- The subagent's `cwd` sandbox blocks writes outside its own working tree. If the subagent's cwd differs from the parent's cwd, it cannot write into the parent's directory (silent stop with "Edit/Write denied").
- Even a single subagent can collide with the main session editing the same file.
- A worktree commit → fetch / merge / rebase keeps subagent work atomic.

Two ways to do it. Pick what fits the work.

### (A) `isolation: "worktree"` parameter

The shortest path. The harness creates the worktree and cleans it up afterward.

```
Agent(implementer, isolation: "worktree", prompt: "Implement feature A...")
Agent(implementer, isolation: "worktree", prompt: "Implement feature B...")
Agent(code-reviewer, prompt: "Review the diffs")   # read-only, no worktree needed
```

### (B) Manual worktree

Useful when you want to keep the worktree around for inspection or have explicit control over the integration step.

1. Create the worktree ahead of time: `git worktree add .worktrees/<name> -b <branch> HEAD`
2. Pass the **absolute path** to the worktree in the subagent prompt (e.g., `/abs/path/to/repo/.worktrees/feature-a`)
3. Subagent edits + commits + (optionally) pushes inside the worktree
4. Commander fetches / merges / rebases to integrate

```
Agent(implementer, cwd: .worktrees/feature-a, prompt: "Implement feature A...")
Agent(implementer, cwd: .worktrees/feature-b, prompt: "Implement feature B...")
```

Commander reviews changes in the worktree before integrating into the main branch. Never auto-merge.

### Permission inheritance trap (read this before going parallel)

- Subagents do **not** inherit the parent's `permissions.allow`. Each subagent has its own permission context.
- Worktrees live at paths outside the original repo, so path-scoped rules like `Edit(/path/to/repo/**)` do not cover them.
- Result: N parallel subagents each show a permission prompt → effectively serialize, or stall silently in background runs.

Fixes (any one is enough; pick based on your security posture):

| Fix | What it does | Watch out for |
|---|---|---|
| Run `/shireito:setup` once | Adds path-scoped allow rules for the repo + `additionalDirectories` for the worktree path | Recommended — least-privilege and project-specific |
| `permissionMode: acceptEdits` in subagent frontmatter | Per-subagent override | Ignored under auto mode; plugin-shipped subagents may not honor it |
| `auto` permission mode globally | Classifier decides per-action | Broadest; review with caution |

**Always dry-run with 1 subagent first** to confirm no permission prompts before going to N parallel.

### Bash compound-command pitfall

Read-only commands can still trigger permission prompts when your allowlist uses `Bash(<cmd> *)` patterns but you compose commands with `cd`:

- `cd <dir> && git status` does **not** match `Bash(git *)` — the prefix is `cd`, not `git`.
- Use the tool's own directory flag instead: `git -C <dir> status`, `npm --prefix <dir> ...`, `docker --context <name> ...`.

Hits commander and subagents alike. The `fewer-permission-prompts` skill scans recent transcripts and proposes the allowlist patterns that match what you actually ran.

## Auto-delegation hints

- Subagent `description` strings drive automatic routing. Use phrases like "Use proactively" or "Use immediately after writing code" to encourage the commander to delegate without explicit instruction.
- Keep descriptions tight. A vague description routes the wrong subagent.

## Failure patterns

| Pattern | Cause | Fix |
|---|---|---|
| Context explosion | Subagent returns full file contents | Restrict tools, instruct "summarize only" in the prompt |
| Infinite loop | Task too complex for one subagent | Split the task, use `maxTurns` |
| Over-delegation | `description` too broad | Narrow it, drop "Use proactively" |
| Silent background failure | Permission deny while running in background | Run in foreground, or run `/shireito:setup` first |
| Parallel write subagent stalls | Permission inheritance trap (see above) | `/shireito:setup` or frontmatter override |
