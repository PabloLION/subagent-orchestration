---
description: Remove subagent orchestration rules from the current project. Deletes ./.claude/rules/orchestration/ and its contents.
disable-model-invocation: true
allowed-tools: Bash Read
---

# Uninstall Subagent Orchestration Rule

Remove orchestration rule files that were installed into the current project.
To update, run this skill first then re-run the install skill.

## Steps

1. Verify `././.claude/rules/orchestration/` exists in the project root
2. List the files that will be removed and confirm with the user
3. Delete `././.claude/rules/orchestration/` and its contents
4. Report what was removed

## Files Removed

- `roles.md` — prerequisites, orchestrator role, agent role, named agents
- `dispatch.md` — issue sizing, batching, dispatch workflow
- `execution.md` — worktree lifecycle, overlap, QA, post-dispatch
- `agent-template.md` — named agent definition template with required settings
