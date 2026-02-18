# Git Cheatsheet

> Quick-reference Git commands and workflows.

## Core Commands / Concepts

### Setup & Init
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git init
git clone <url>
```

### Staging & Committing
```bash
git status
git add <file>
git add -p              # interactive hunk staging
git commit -m "message"
git commit --amend      # amend last commit
```

### Branching
```bash
git branch              # list branches
git branch feature/xyz  # create branch
git checkout -b feature/xyz   # create and switch
git switch -c feature/xyz     # modern equivalent
git merge feature/xyz
git rebase main
git branch -d feature/xyz     # delete branch
```

### Remote Operations
```bash
git remote -v
git fetch origin
git pull origin main
git push origin feature/xyz
git push --force-with-lease    # safe force push
```

### Undoing Changes
```bash
git restore <file>                # discard working dir changes
git restore --staged <file>       # unstage
git reset HEAD~1                  # undo last commit (keep changes)
git reset --hard HEAD~1           # undo last commit (discard changes)
git revert <commit>               # safe undo (creates new commit)
git stash && git stash pop
```

### Inspection
```bash
git log --oneline --graph --all
git diff HEAD
git show <commit>
git blame <file>
git reflog
```

### Tags
```bash
git tag v1.0.0
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin --tags
```

## Common Patterns

<!-- Add branching strategies, merge vs rebase guidelines, commit message conventions here -->
