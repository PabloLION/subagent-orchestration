---
description: Install subagent orchestration rule into the current project. Copies orchestration reference files to ./.claude/rules/orchestration/ so the project supports multi-agent dispatch with beads, worktrees, and named agents.
disable-model-invocation: true
allowed-tools: Bash Read
---

# Install Subagent Orchestration Rule

Copy orchestration rule files from this skill's references into the current project.

## Steps

1. Check if `././.claude/rules/orchestration/` already exists in the project root
   - If it exists, stop and tell the user to run `/uninstall-subagent-orchestration-rule`
     first, then re-run this skill
2. Create `././.claude/rules/orchestration/` in the project root
3. Copy all files from [references/](references/) to `././.claude/rules/orchestration/`
4. Verify the project has the prerequisites listed in `roles.md`
5. Report which prerequisites are missing so the user can set them up

## Files Installed

- `roles.md` — prerequisites, orchestrator role, agent role, named agents
- `dispatch.md` — issue sizing, batching, dispatch workflow
- `execution.md` — worktree lifecycle, overlap, QA, post-dispatch
- `agent-template.md` — named agent definition template with required settings
