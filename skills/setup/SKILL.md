---
name: setup
description: Configure shireito permissions and worktree paths for the current project. Detects the project root, asks the user about worktree location and permission strategy, then merges the result into .claude/settings.json. Run this once per project after installing the plugin.
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Bash
---

# Shireito Setup

Configure the current project so that the 5 shireito subagents can write code without permission prompts and use git worktrees for parallel implementation.

## Why a setup skill instead of shipping settings.json

Subagents do **not** inherit the main session's `permissions.allow`. Worktrees are created at a path outside the original repo, so any `Edit(/path/to/repo/**)` rule does not cover them. This means write-capable subagents (`implementer`, `debugger`) hit permission prompts in parallel and effectively serialize.

The fix is project-specific (it embeds absolute paths), so it cannot ship as a plugin file. This skill asks the user the right questions and writes the merged result.

## Steps

Run all of the following. Ask the user before any write.

### 1. Detect project root

```bash
pwd
```

Confirm with the user: "Set up shireito for this project? (`<pwd>`)" — abort if they say no.

### 2. Read existing `.claude/settings.json`

```bash
cat .claude/settings.json 2>/dev/null || echo "(no existing settings.json)"
```

Note any existing `permissions.allow` / `additionalDirectories` keys so the merge preserves them.

### 3. Ask: where should worktrees live?

Default: `<pwd>/.worktrees`. Offer alternatives:

- `<pwd>/.worktrees` (recommended — repo-local, easy to gitignore)
- `~/worktrees/<repo-name>` (out-of-tree, keeps repo clean)
- custom path

Make sure the chosen path is **absolute** in the final settings.

### 4. Ask: which permission strategy?

Present the trade-offs and let the user pick one:

| Strategy | What it does | Trade-off |
|---|---|---|
| **(A) Broad allow rule (recommended)** | Adds `Edit(<pwd>/**)`, `Write(<pwd>/**)`, `Read(<pwd>/**)` to `permissions.allow`. Combined with `additionalDirectories` for the worktree path. | Subagents write within the repo and worktree without prompts. Path-scoped, so still least-privilege. |
| **(B) `acceptEdits` mode per subagent** | Adds `permissionMode: acceptEdits` to each agent's frontmatter. | Per-agent control. Note: ignored under auto mode. |
| **(C) auto mode** | Suggest the user set `permissionMode: "auto"` globally. | Most permissive. Classifier decides per-action. |

Default to (A) unless the user has reasons to choose otherwise.

### 5. Show the merge plan

Before writing, show the user exactly what will be added. Example for strategy (A):

```json
{
  "permissions": {
    "allow": [
      "Edit(/Users/example/myproject/**)",
      "Write(/Users/example/myproject/**)",
      "Read(/Users/example/myproject/**)"
    ],
    "additionalDirectories": [
      "/Users/example/myproject/.worktrees"
    ]
  }
}
```

Highlight that this is **merged** into existing settings, not replacing them.

### 6. Write the merged result

Use `Edit` (or `Write` if no existing file) to update `.claude/settings.json`. Preserve any existing keys outside the touched paths.

### 7. (Optional) Add `.worktrees` to `.gitignore`

Ask the user if you should append `.worktrees/` to the project's `.gitignore`. Skip if already present.

### 8. Verify with a dry-run

Suggest the user run a single write-capable subagent in dry-run mode to confirm permissions work:

```
Use the implementer subagent in a worktree to make a trivial change (e.g., add a comment to a file). If it completes without prompting, setup is working.
```

If the user wants to actually run the dry-run, set up a worktree:

```bash
git worktree add .worktrees/shireito-dry-run -b shireito-dry-run HEAD
```

Then invoke `Agent` with subagent `implementer` against that worktree's absolute path.

## After setup

Tell the user: shireito's 5 subagents are now available via `/agents` or by name. Try `/shireito:orchestrate` to load the orchestration rules into context when planning multi-subagent work.
