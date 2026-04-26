# Code Quality Tools

smart fixup commits

## git-absorb - Smart Fixup Commits

### How It Works

1. You stage changes with `git add` (the fixes you want to absorb)
2. `git absorb` analyzes the staged hunks
3. For each hunk, it finds the most recent commit that modified those exact lines
4. It creates `fixup!` commits targeting those parent commits
5. `git rebase --autosquash` folds the fixup commits into their targets

### Typical Workflow

```bash
# 1. You have a feature branch with 5 commits
# 2. Code review requests changes to lines in commits 2 and 4
# 3. Make the fixes in your working tree
# 4. Stage them
git add -p

# 5. Let git-absorb figure out which commits they belong to
git absorb

# 6. Verify the fixup commits look correct
git log --oneline main..HEAD

# 7. Squash fixups into their parent commits
git rebase --autosquash main

# 8. Force-push the cleaned-up branch
git push --force-with-lease
```

### Comparison with Manual Fixup Workflow

| Step                   | Manual                         | git-absorb                        |
| ---------------------- | ------------------------------ | --------------------------------- |
| Identify target commit | `git log --oneline`, find SHA  | Automatic                         |
| Create fixup commit    | `git commit --fixup=<SHA>`     | `git absorb`                      |
| Apply fixups           | `git rebase --autosquash main` | `git absorb --and-rebase` or same |
| Multiple fixes         | Repeat per commit              | Single command handles all        |

### Limitations

- Only works with staged changes (use `git add -p` for partial staging)
- Cannot absorb changes to lines that were not in any prior commit (new code)
- Ambiguous hunks (lines modified in multiple commits) are skipped with a warning
- Requires a clean working tree for `--and-rebase`

### Installation

```bash
# macOS
brew install git-absorb

# Arch Linux
pacman -S git-absorb

# Cargo (any platform)
cargo install git-absorb

# Ubuntu/Debian (via cargo or from releases)
cargo install git-absorb
```
