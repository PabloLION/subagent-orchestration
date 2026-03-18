# Operational Lessons

Hard-won lessons from multi-agent orchestration sessions. These supplement the
core rules (roles, dispatch, execution, agent-template) with practical
knowledge that prevents repeated mistakes.

## Git Operations

### Always merge from the main worktree directory

Run `git merge` from the project root, never from inside a worktree directory.
Git resolves merge targets relative to the current HEAD. If you are inside a
worktree, `git merge --no-ff <branch>` reports "Already up to date" because the
worktree branch is its own HEAD.

```sh
# Wrong — inside worktree, merge does nothing
cd ./.claude/worktrees/agent-abc123
git merge --no-ff worktree-agent-abc123  # "Already up to date"

# Right — from main directory
cd /path/to/project
git merge --no-ff worktree-agent-abc123  # Merge made
```

### Run `bun install` after merging worktree branches

Worktrees have their own `./node_modules/`. After merging a branch that changed
`package.json` or lockfile, run `bun install` on main before running tests.
Even without dependency changes, test failures after merge may indicate stale
`./node_modules/` — try `bun install` first.

### Nested worktrees from stale paths

Agents launched from within a (removed) worktree directory create deeply nested
paths like `./.claude/worktrees/agent-abc/./.claude/worktrees/agent-def/`. This does
not break branch-based merges but leaves stale directories. Clean with:

```sh
rm -rf ./.claude/worktrees/agent-*
git worktree prune
```

Prevention: always use absolute paths when creating worktrees, and verify `pwd`
is the main project root before dispatching.

### Worktree removal may need `--force`

Untracked files (agent memory, test artifacts) prevent `git worktree remove`.
Use `--force` when removing after a successful merge. Then verify the branch is
merged before force-deleting — `git branch -d` can falsely refuse after
`--no-ff` merges:

```sh
git worktree remove --force ./.claude/worktrees/agent-abc123
git branch --merged main | grep worktree-agent-abc123  # verify merged
git branch -D worktree-agent-abc123
```

## Merge Conflicts

### Name changes create merge artifacts

When Agent A renames a function and Agent B writes new code using the old name,
merging both produces code that compiles but references undefined symbols. After
merging both:

1. Run the test suite — catch `ReferenceError` immediately
2. Use `replace_all` to fix old name → new name across affected files
3. Commit the fix separately before merging the next branch

### Uncommitted changes block merges

If the orchestrator made a direct fix (1-2 line edit) without committing, the
next `git merge` fails with "Your local changes would be overwritten." Always
commit orchestrator fixes before merging the next agent branch.

## Agent Dispatch

### Agents may exceed their scope

Agents sometimes fix adjacent issues they discover while working. Before
dispatching a dependent issue, check what the previous agent actually changed —
the dependent work may already be done.

Example: a server agent updating handler/UI types while fixing a server bug.
The handler issue that was supposed to fix those types is now redundant.

### Agents should report out-of-scope observations

Encourage agents to note issues they discover but do not fix. Include in agent
system prompts:

> If you notice adjacent issues outside your scope, mention them in your
> completion summary. Do not fix them — the orchestrator will create tracking
> issues.

### Closing resolved-by-proxy issues

When merging Agent A's work resolves issues assigned to Agent B, close those
issues without dispatching Agent B. Verify the fix by reading the merged code,
then close with reason explaining which merge resolved it:

```sh
bd close ch-xxx --reason="Resolved by ch-yyy merge (commit abc1234)"
```

## Issue Tracking (Beads)

### Use `--no-daemon` for reliability

The beads daemon caches state and may time out on socket reads. Always use
`--no-daemon` for `bd close`, `bd update`, `bd create`, and `bd show` during
orchestration sessions:

```sh
bd close ch-xxx --reason="Done" --no-daemon
bd show ch-xxx --no-daemon
```

### Parent-child issues

Use `bd create --parent=<id>` for sub-issues. This generates dot-suffix IDs
(ch-xxx.1, ch-xxx.2). Mark the parent as `in_progress` to unblock children.
Close children with `--force` if parent is still `in_progress`.

### Close with commit hash

Always include the merge commit hash when closing:

```sh
bd close ch-xxx --reason="<summary>. Commit: abc1234" --no-daemon
```

## Pre-Dispatch Protocol

### Number all doubts

When presenting doubts about an issue, always use numbered indexes. Users refer
back to doubts by number. Unnumbered doubts force the user to quote text.

```text
# Good
1. Schema doesn't specify the return type for edge case X
2. No test coverage for error path Y

# Bad
- Schema doesn't specify...
- No test coverage...
```

### Update issues before dispatch

When a doubt is resolved or a design decision is made during discussion, update
the issue description or notes immediately (`bd update <id> --description/--notes`).
The agent reads the issue at dispatch time — stale descriptions cause spec drift.

### Research agents for ambiguous scope

When an issue's scope is unclear or the orchestrator suspects deeper problems,
dispatch a research agent (Explore or QA) before a code agent. Research agents
produce reports; code agents produce commits. Dispatching code work on an
ambiguous spec wastes the agent's effort.

## Port Conflicts

### Test suite port collisions

If the project uses a fixed port (e.g., 6004), agent worktrees running tests
may occupy the port. Symptoms: `PortInUseError` or `EADDRINUSE` in test output.
Solution: re-run tests after the conflicting process exits. This is a known
pre-existing issue, not a merge failure.

## Context Management

### Orchestrator does not write code

The main thread conducts agents — it never plays an instrument. Exceptions:

- Trivially small fixes (1-2 lines) after a merge conflict
- biome format fixes on files the orchestrator touched
- Issue tracker operations and git merges

If you catch yourself reading source code to understand implementation details,
stop and dispatch a research agent instead.

### Probe scripts use temp cwd

Empirical probes (testing Claude Code behavior) should run from a temporary
directory, not the project root. This prevents probe sessions from appearing in
Claude Code's resume list:

```ts
const probeCwd = mkdtempSync(join(tmpdir(), "probe-cwd-"));
Bun.spawn(args, { cwd: probeCwd });
```

Also unset `CLAUDECODE` env var to avoid guard conflicts when running probes
from inside a Claude Code session.
