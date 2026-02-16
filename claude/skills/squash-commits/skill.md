---
name: squash-commits
description: Analyze unmerged commits on the current branch and squash related ones together (especially fix-ups). Shows before/after summary.
disable-model-invocation: true
allowed-tools: Bash(git *)
argument-hint: "[base-branch]"
---

# Squash Related Commits

Analyze unmerged commits on the current branch and intelligently squash related ones together, especially fix/followup commits into their parent feature commits.

## Context

- Current branch: !`git rev-parse --abbrev-ref HEAD`
- Base branch argument (default "main"): $ARGUMENTS

## Step 1: Gather Commits

Determine the base branch — use `$ARGUMENTS` if provided, otherwise default to `main`.

Run `git log <base>..HEAD --oneline --reverse` to get all unmerged commits in chronological order (oldest first).

If there are **0 or 1 commits**, report that there's nothing to squash and stop.

### Handling Merge Commits

Check for merge commits in the range using `git log <base>..HEAD --merges --oneline`.

- **If no merge commits exist**: Proceed normally with `<base>` as the rebase base.
- **If a merge commit exists**: Use the **last** merge commit as the rebase base instead of `<base>`. This skips pre-merge history and only squashes commits after the merge. Run `git log <last-merge-sha>..HEAD --oneline --reverse` to get the actionable commits. If there are non-merge commits both before AND after the merge, **ask the user** if they want to proceed with only the commits after the last merge commit (the pre-merge commits will be left untouched). If the user declines, stop.

Save the final commit list as the **"BEFORE"** state — you'll display it later.

## Step 2: Analyze and Group

Examine the commit messages and identify **groups of related commits** that should be squashed together. Use these heuristics:

1. **Fix-up pattern**: A `fix:` or `fix(scope):` commit that clearly relates to an earlier `feat:`, `refactor:`, or other commit on the same topic. The fix should be squashed INTO the original.
2. **Continuation pattern**: Sequential commits working on the same feature/area (e.g., "feat: add user modal" followed by "feat: add validation to user modal").
3. **Cleanup pattern**: Commits like "style:", "chore: lint", "fix: typo", "fix: lint" that follow a substantive commit and are clearly cleaning up after it.
4. **Test pairing**: A test commit (e.g., "test: add user tests") that directly follows the feature it tests can optionally be squashed.

**Do NOT squash:**
- Unrelated commits that happen to be adjacent
- Commits that are already clean and standalone
- Commits across clearly different features/areas

## Step 3: Present the Plan

Display a clear plan showing:

```
BEFORE (N commits):
  abc1234 feat: add parts request model
  def5678 fix: correct foreign key in parts model
  ghi9012 feat: add parts API endpoint
  jkl3456 fix: lint errors
  mno7890 test: add parts request tests

PROPOSED SQUASH PLAN:
  Group 1: abc1234 + def5678 → "feat: add parts request model"
  Group 2: ghi9012 + jkl3456 → "feat: add parts API endpoint"
  Standalone: mno7890 test: add parts request tests

AFTER (3 commits):
  abc1234 feat: add parts request model
  ghi9012 feat: add parts API endpoint
  mno7890 test: add parts request tests
```

If no commits are worth squashing, say so and stop.

**Ask the user to confirm** before proceeding. This is a rewrite of git history — never proceed without explicit approval.

## Step 4: Execute the Rebase

Use `GIT_SEQUENCE_EDITOR` to programmatically perform a non-interactive rebase. For example:

```bash
GIT_SEQUENCE_EDITOR="sed -i '' '<sed commands to change pick to fixup>'" git rebase -i <base>
```

For each group, change the later commits from `pick` to `fixup` (which squashes them into the preceding commit and discards their messages) or `squash` (if you want to combine messages).

**Use `fixup`** when the fix/cleanup message adds no value (e.g., "fix: lint").
**Use `squash`** when both messages are meaningful and should be combined — then also set `GIT_SEQUENCE_EDITOR` for the message combination step, or use `fixup` and then `git commit --amend -m "combined message"` on the result.

**Preferred approach**: Use `fixup` for all squashed commits, then if the surviving commit's message needs updating, amend it afterward.

If the rebase encounters conflicts, abort with `git rebase --abort`, report the conflict, and stop.

## Step 5: Show Before/After Summary

After successful rebase, always display:

```
═══════════════════════════════════════════════
          SQUASH COMPLETE
═══════════════════════════════════════════════

BEFORE (N commits):
  <full list from Step 1>

AFTER (M commits):
  <run: git log <base>..HEAD --oneline --reverse>

Squashed N commits → M commits (K fewer)
═══════════════════════════════════════════════
```

## Important Rules

- **NEVER force-push automatically.** After squashing, remind the user they'll need `git push --force-with-lease` if the branch was already pushed.
- **Always ask for confirmation** before executing the rebase.
- **Abort on conflicts** — don't try to resolve rebase conflicts automatically.
- If the branch IS main/master, refuse to rebase and warn the user.
