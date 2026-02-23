# Git — Basic Questions

---

## 1. What is the difference between `git fetch` and `git pull`?

Both bring changes from a remote repository — but they do it differently, and mixing them up causes confusion about what's in your local branch.

**`git fetch` — download changes, don't apply them.**

```bash
git fetch origin
# Downloads commits, branches, tags from the remote
# Updates remote-tracking branches (origin/main, origin/feature-x)
# Does NOT touch your local branch — your working directory is unchanged
```

After `git fetch`, you can inspect what came in before deciding what to do:

```bash
git fetch origin
git log origin/main --oneline   # See what's new on remote main
git diff main origin/main       # See exactly what would change if you merge
git merge origin/main           # Now apply when you're ready
```

**`git pull` — fetch + merge (or rebase) in one command.**

```bash
git pull origin main
# Equivalent to: git fetch origin && git merge origin/main
# Downloads changes AND applies them to your current branch
```

`git pull` is a convenience command — it's `git fetch` followed by either `git merge` or `git rebase` depending on your config (`pull.rebase = true/false`).

**When to use which:**

Use `git fetch` when you want to:
- See what changed before applying it
- Review remote branches without changing your working state
- Keep your local branch clean while pulling down new refs

Use `git pull` when you're on a clean branch and just want the latest changes applied immediately.

**The key difference — state of your working directory:**

```bash
# Scenario: colleague pushed 3 commits to origin/main
# You're on main with 2 local commits not yet pushed

git fetch origin
# Remote tracking updated, your main branch: UNCHANGED
# Your 2 local commits: STILL THERE
# Now you can inspect: git log origin/main

git pull origin main
# Immediately merges (or rebases) origin/main into your current main
# If there are conflicts, you deal with them right now
```

**`git pull --rebase` (what many teams prefer):**

```bash
git pull --rebase origin main
# = git fetch + git rebase origin/main
# Replays your local commits on top of the remote commits
# Keeps a linear history (no merge commit)
```

Without `--rebase`, `git pull` creates a merge commit when histories diverge. With `--rebase`, your local commits are replayed on top of the remote changes — cleaner history, but it rewrites local commit SHAs.

**Real scenario:** I use `git fetch` before code reviews — I want to see what's on the remote branches without accidentally modifying my working directory. I use `git pull --rebase` to update my feature branch with the latest from `develop` before opening a PR — this keeps the feature branch rebased on current develop and avoids a merge commit cluttering the history.

---

## 2. How do you handle merge conflicts in Git?

A merge conflict happens when two branches changed the same lines of the same file differently. Git can't automatically decide which change is correct — it needs a human.

**When conflicts occur:**

```bash
git merge feature/payment-retry
# CONFLICT (content): Merge conflict in src/payment.py
# Automatic merge failed; fix conflicts and then commit the result.
```

Or during a rebase:
```bash
git rebase main
# CONFLICT (content): Merge conflict in src/payment.py
# error: could not apply abc1234... add retry logic
```

**What the conflict looks like in the file:**

```python
<<<<<<< HEAD (current branch — your changes)
def process_payment(amount, currency="USD"):
    """Process with immediate retry on failure."""
    for attempt in range(3):
        try:
            return _charge(amount, currency)
        except PaymentError:
            continue
=======
def process_payment(amount, currency):
    """Simple payment processor."""
    return _charge(amount, currency)
>>>>>>> feature/payment-retry (incoming changes)
```

**Step-by-step resolution:**

```bash
# Step 1 — Find all conflicted files
git status
# Both modified: src/payment.py

# Step 2 — Open the file, understand BOTH changes, write the correct resolution
# Don't just pick one — often the right answer combines both

# After editing:
def process_payment(amount, currency="USD"):
    """Process with retry on failure (default USD)."""
    for attempt in range(3):
        try:
            return _charge(amount, currency)
        except PaymentError:
            if attempt == 2:
                raise
            continue
# This combines the default parameter from HEAD with the retry logic from the feature branch

# Step 3 — Mark the conflict as resolved
git add src/payment.py

# Step 4 — Continue the merge or rebase
git merge --continue    # For merge
git rebase --continue   # For rebase

# If you want to abort and get back to the pre-conflict state:
git merge --abort
git rebase --abort
```

**Use a merge tool for complex conflicts:**

```bash
git mergetool                   # Opens configured tool
git config --global merge.tool vimdiff    # or vscode, intellij, etc.

# VS Code as merge tool:
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

A 3-way merge tool shows: LEFT (your branch), RIGHT (incoming), and CENTER (common ancestor). Seeing the ancestor makes it clear where each change originated.

**Preventing conflicts — the real solution:**

```bash
# Before opening a PR, rebase on the target branch:
git fetch origin
git rebase origin/main    # Replay your commits on top of current main
# Conflicts here are easier to resolve — you're doing them one commit at a time
# vs. resolving a large accumulated conflict at merge time

# Keep feature branches short-lived (< 2 days)
# Long-lived branches accumulate drift from the main branch = larger conflicts
```

**Real scenario:** We had a merge conflict in our Terraform state configuration — two engineers both modified `backend.tf` in the same week. One added a new S3 bucket key path, the other changed the DynamoDB table name. The conflict was clean to resolve (combine both changes), but we needed to make sure neither engineer's terraform had already applied with their version. We checked Terraform state before resolving to confirm no partial applies had happened, then resolved the conflict and ran `terraform plan` to verify the combined result was clean.

---

## 3. How do you generate a Personal Access Token (PAT) in GitHub?

A Personal Access Token (PAT) is used in place of a password when authenticating with GitHub from the CLI, scripts, CI/CD systems, or tools. GitHub removed password authentication for Git operations in 2021 — PAT is the replacement.

**Steps to generate a PAT (Classic):**

1. Go to **GitHub → Settings** (click your avatar top-right → Settings)
2. In the left sidebar, scroll down to **Developer settings**
3. Click **Personal access tokens → Tokens (classic)**
4. Click **Generate new token → Generate new token (classic)**
5. Give it a descriptive **Note** (e.g., "Laptop CLI", "Jenkins CI", "Deployment Script")
6. Set **Expiration** — choose a sensible expiry (30, 60, 90 days, or custom). Avoid "No expiration" for security
7. Select **Scopes** (permissions):
   - `repo` — full access to repositories (read, write, delete). Required for most operations
   - `read:org` — if you need to access organization resources
   - `workflow` — if you need to trigger or modify GitHub Actions
   - `write:packages` — if you need to push to GitHub Container Registry (GHCR)
8. Click **Generate token**
9. **Copy the token immediately** — GitHub shows it only once. If you close the page without copying, you must regenerate it.

**Using the token:**

```bash
# Option 1: Use token as password in git operations
git clone https://github.com/org/repo.git
# Username: your-github-username
# Password: <paste the token here>

# Option 2: Embed in the remote URL (for scripts — be careful with secrets)
git clone https://<username>:<token>@github.com/org/repo.git

# Option 3: Set in git credential store
git config --global credential.helper store
git clone https://github.com/org/repo.git
# Enter username + token once → stored in ~/.git-credentials

# Option 4: Use GitHub CLI (authenticates once)
gh auth login
# → select GitHub.com, HTTPS, paste token
# gh handles all subsequent authentication automatically
```

**For CI/CD systems (Jenkins, GitHub Actions, GitLab CI):**

```bash
# In GitHub Actions — use the built-in GITHUB_TOKEN (no manual PAT needed for same-repo operations)
- name: Checkout
  uses: actions/checkout@v4
  with:
    token: ${{ secrets.GITHUB_TOKEN }}

# For cross-repo access or external tools — store PAT as a secret:
# GitHub → repo Settings → Secrets and variables → Actions → New secret
# Name: GH_PAT, Value: <your token>

- name: Clone another repo
  run: git clone https://x-access-token:${{ secrets.GH_PAT }}@github.com/org/other-repo.git
```

**Fine-grained PATs (the newer, more secure option):**

GitHub now offers **Fine-grained tokens** (Settings → Developer settings → Personal access tokens → Fine-grained tokens) that allow per-repository permissions rather than all-or-nothing repository access. Prefer these for new integrations — they follow the principle of least privilege and can be scoped to specific repositories.

