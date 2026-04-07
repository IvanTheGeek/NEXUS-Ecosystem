# NEXUS Git Workflow

This strategy applies to all NEXUS projects. The goal is full traceability — no progression, decision, or experiment is ever lost.

---

## Principles

- **Commit often.** Small commits capture the progression of thought, not just the end result.
- **Branch for everything.** Every piece of work gets a branch. Direct commits to `main` are for single-commit trivial fixes only.
- **Merge with `--no-ff`.** Always. This preserves branch topology in the history so any body of work can be traced as a unit.
- **Delete after merge.** Branch names are cleaned up after merging. The history lives in the merge commit — the branch name is just a pointer.
- **Never rewrite pushed history.** No `rebase`, no `--amend` on pushed commits, no `push --force`.

---

## Branch Types

| Prefix | Use for | Merges to |
|---|---|---|
| `feature/` | New capabilities | `main` |
| `experiment/` | Uncertain-outcome explorations | `main` or `graveyard` |
| `fix/` | Bug fixes | `main` |
| `docs/` | Documentation-only changes | `main` |
| `refactor/` | Restructuring with no behavior change | `main` |

---

## Permanent Branches

**`main`** — always working, always the integration target. Every merge here represents a completed, coherent body of work.

**`graveyard`** — a long-running archive for abandoned and failed work. Experiments that don't pan out, directions that get superseded, dead ends worth remembering — they merge here before their branch is deleted. `graveyard` never merges back to `main`. It exists to answer the question: *what was tried?*

---

## Normal Work Lifecycle

```bash
# Start
git checkout main
git checkout -b feature/my-work

# Work — commit frequently
git add -A
git commit -m "feat: add X"
git commit -m "feat: handle edge case in Y"
git push -u origin HEAD        # push early, push often

# Finish — merge back to main
git checkout main
git merge --no-ff feature/my-work -m "feat: my work"
git push origin main

# Clean up
git push origin --delete feature/my-work
git branch -d feature/my-work
```

---

## Abandoned Work Lifecycle

```bash
# Work on experiment branch, then decide not to proceed
# Before deleting — preserve it in the graveyard
git checkout graveyard
git merge --no-ff experiment/my-dead-end -m "graveyard: experiment/my-dead-end — reason it was abandoned"
git push origin graveyard

# Return to main and clean up the branch
git checkout main
git push origin --delete experiment/my-dead-end
git branch -d experiment/my-dead-end
```

The commit message on the graveyard merge should explain *why* the work was abandoned. Future readers (including yourself) will find this context invaluable.

---

## Auto-Commit Hook

Each NEXUS project has a `.claude/settings.json` Stop hook. At the end of every Claude Code session, any uncommitted changes are captured in a checkpoint commit and pushed to the current branch. Nothing is lost between sessions.

The hook skips if there are no changes. It pushes to the current branch — always safe because you are always on a feature branch, never directly on `main`.

---

## Commit Message Conventions

Use a short type prefix for clarity:

| Prefix | Use |
|---|---|
| `feat:` | New capability |
| `fix:` | Bug fix |
| `docs:` | Documentation only |
| `refactor:` | No behavior change |
| `test:` | Test changes only |
| `chore:` | Tooling, hooks, config |
| `experiment:` | Exploratory work |
| `graveyard:` | Merge commit message for abandoned branch |

Keep the subject line under 72 characters. Add a body when the *why* isn't obvious from the code.

---

## Why `--no-ff`?

With a fast-forward merge, the feature branch commits land directly on `main` with no record that they were ever a separate line of work. The branch topology — and the fact that these commits formed a coherent unit — is lost.

With `--no-ff`, a merge commit is created that explicitly records: *at this point, branch X was merged as a unit*. The entire branch can be identified, reviewed, and if necessary reverted as one operation (`git revert -m 1 <merge-commit>`).

```
# Fast-forward (avoid)
A - B - C - D - E    main
            ↑
            merged invisibly

# No-FF (always use)
A - B - C --------- M    main
            ↓       ↑
            D - E --      feature/x (topology preserved)
```
