---
name: git-workflow
description: "Use when establishing branching strategies, implementing Conventional Commits, creating or reviewing PRs, resolving PR review comments, merging PRs (including CI verification, auto-merge queues, and post-merge cleanup), managing PR review threads, merging PRs with signed commits, handling merge conflicts, creating releases, integrating Git with CI/CD, setting up git hooks (lefthook, captainhook, husky, pre-commit), or debugging hook-install failures in git worktrees."
compatibility: "Requires git, gh CLI."
allowed-tools: Bash(git:*) Bash(gh:*) Read Write
metadata:
  author: Kurb App
  version: "1.0.0"
  repository: https://github.com/Hanzm10/Kurb-app/skills
---

# Git Workflow Skill

Patterns for Git version control: branching, commits, collaboration, and CI/CD.

## Critical Rules (Non-Negotiable)

1. **No direct push to main** - always create a PR
2. **No merge before all review threads are resolved** - run the merge gate in `references/pull-request-workflow.md`.
3. **No squash at all** - atomic commits preserved; keeps GPG signatures and bisection.
4. **No "tested/verified/working" without pasted command output** - If the check cannot be run, say so.
5. **No edits to installed skill/plugin cache paths** (`~/.claude/skills/`, `~/.claude/plugins/cache/`, `**/.bare/**`) — always the repo worktree. Verify `pwd` first. 6.**Force-push only with `--force-with-lease`** — never plain `--force`.

See `references/pull-request-workflow.md` for the merge-gate command, atomic-commit guidance, and review-thread SHA-citation pattern.

## Reference Files

| Reference                             | Content Triggers                                                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `references/branching-strategies.md`  | Branching model, Git Flow, GitHub Flow, trunk-based, branch protection                           |
| `references/commit-conventions.md`    | Commit messages, conventional commits, semantic versioning, commitlint                           |
| `references/versioning.md`            | Versioning lifecycle, stages, and when to start                                                  |
| `references/pull-request-workflow.md` | PR create/review/merge, thread resolution, merge strategies, CODEOWNERS, signed commits + rebase |
| `references/ci-cd-integration.md`     | GitHub Actions, GitLab CI, semantic release, deployment                                          |
| `references/advanced-git.md`          | Rebase, cherry-pick, bisect, stash, worktrees, reflog, submodules, recovery                      |
| `references/git-hooks-setup.md`       | Hook frameworks, detection, recommended hooks per stage                                          |
| `references/claude-code-hooks.md`     | Claude Code `settings.json` hooks — merge gate, cache-path rejection, auto-lint                  |
| `references/code-quality-tools.md`    | git-absorb                                                                                       |

## Conventional Commits

```
<type>[scope]: <description>
```

**Types**: `feat` (MINOR), `fix` (PATCH), `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`

**Breaking change**: Add `!` after type or `BREAKING CHANGE:` in footer.

## Branch Naming

```
feature/TICKET-123-description
fix/TICKET-456-bug-name
release/1.2.0
hotfix/1.2.1-security-patch
```

## Hook Detection

Before first commit, deteect and install hooks:

```
ls lefthook.yml .lefthook.yml captainhook.json .pre-commit-config.yaml .husky/pre-commit 2>/dev/null || echo "No hooks"
```

## Critical Release Rules

1. **Immutable releases**: Deleted releases permanently block tag reuse; bump version instead.
2. **Multi-branch releases**: Use `--latest=false` from non-default branches.
3. **Pre-release**: Version bumped, CI green, CHANGELOG updated, `git pull` BEFORE `gh release create`.

## PR Merge Requirements

Before merging: all threads resolved, CI checks green (including annotations), branch rebased, commits signed (if required). For signed commits + rebase-only repos, use local `git merge --ff-only`.

## Verification

```bash
./scripts/verify-git-workflow.sh /path/to/repository
```

---

> **Attribution:**
> _This skill is a derivative of "Git Workflow Skill" by netresearch, used under CC BY-SA 4.0._
> _Source: [GitHub Repository](https://github.com/netresearch/git-workflow-skill)_
> _License: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)_
> _Modifications: Use and change only the needed parts for commit, push, pr, and merge conventions._
> _This modified skill is licensed under CC BY-SA 4.0._
