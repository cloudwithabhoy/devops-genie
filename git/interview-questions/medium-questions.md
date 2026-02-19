# Git — Medium Questions

---

## 1. What branching strategy do you follow across Dev, QA, Staging, and Prod?

The most common interview answer is "we use Gitflow." The better answer explains why you chose a specific model and what problems it solved — or created.

**What we use: Environment branches with trunk-based short-lived feature branches.**

We have four long-lived branches, one per environment:

```
main         ← Production (protected, no direct push)
staging      ← Pre-prod, mirrors prod infra exactly
develop      ← Integration branch, synced to QA env
feature/*    ← Short-lived (< 2 days), one per task
```

**The flow:**

```
feature/my-feature
        ↓  PR + review + CI passes
      develop   →  Auto-deploys to QA env via CI/CD
        ↓  QA signs off, PR raised
      staging   →  Auto-deploys to Staging env
        ↓  Smoke tests pass, stakeholder approval
       main     →  Deploys to Production
```

**Branch protection rules we enforce on GitHub:**

- `main` and `staging`: require 1 approving review, require passing CI, no force-push, no direct commits
- `develop`: require passing CI (but no mandatory review — too slow for daily dev)
- Feature branches: auto-deleted after merge (keeps the repo clean)

**What actually happens in practice:**

A developer creates `feature/JIRA-123-add-payment-retry` from `develop`. They commit, push, open a PR. Jenkins runs tests, SonarQube scans, Trivy scans the Docker image. Once CI is green and a reviewer approves, they merge to `develop`. That triggers a Jenkins pipeline that deploys to the QA environment.

QA team tests on the QA env. When they sign off, the developer raises another PR: `develop` → `staging`. After staging deploy and smoke tests pass, a release manager reviews and approves `staging` → `main`. Production deploy happens.

**Why we don't use full Gitflow (with separate `release` branches and `hotfix` branches):**

We tried Gitflow. It created two real problems:

1. **Long-lived `release` branches** became a parallel codebase. By the time a release branch was deployed, `develop` had 3 weeks of additional commits. Cherry-picking hotfixes back was a constant source of merge conflicts.

2. **Cognitive overhead.** New engineers spent their first week learning the branching model instead of shipping code. "Do I branch from `develop` or `release/1.4`?" caused real mistakes.

We simplified to 4 branches. The mental model is: feature → develop → staging → main. One direction, no loops.

**Hotfix flow** (the one exception):

For production emergencies:

```
hotfix/critical-bug  ← Branch from main (NOT develop)
        ↓  Fix, test locally, PR to main (expedited review)
       main           → Deploy to prod
        ↓  Cherry-pick or merge back
      develop         → Keep develop in sync
```

Branching a hotfix from `main` ensures the fix only contains what's in production — not the 2 weeks of features sitting in `develop` that aren't deployed yet.

**What broke without this discipline:** Before we had branch protection, a developer force-pushed to `main` to "fix a quick typo" and accidentally reverted 3 merged PRs. Nobody noticed until QA found bugs that had already been fixed. After that incident: branch protection enabled, force-push disabled, all merges via PR. Non-negotiable.

**Key insight:** The "best" branching strategy is the simplest one your team will actually follow consistently. A perfect Gitflow that people work around is worse than a simple model everyone uses correctly.

---

## 2. GitFlow vs Trunk-based development — which do you follow, and how do you avoid breaking the release branch?

This is less about which is "better" and more about what fits your team's release cadence and risk tolerance.

**Trunk-based development (TBD):**

Everyone commits to `main` (the "trunk") — either directly or via very short-lived feature branches (< 1-2 days). Releases are cut from `main` at any time using tags or release branches.

```
main ←── feature/add-retry (merged same day)
     ←── fix/null-check    (merged same day)
     ←── feature/new-ui    (merged, feature-flagged if not ready)
     ↓ tag v1.4.2          (release cut from main)
```

**GitFlow:**

Multiple long-lived branches: `main`, `develop`, `release/*`, `hotfix/*`. Features branch from `develop`, get merged back to `develop`, then `develop` is merged to a `release` branch for stabilization, then to `main`.

```
main ←── release/1.4 ←── develop ←── feature/add-retry
     ←── hotfix/1.4.1
```

**What we use: a simplified environment-branch model** (closer to GitFlow, lighter on ceremony).

We have `feature` → `develop` → `staging` → `main`. No separate `release` branches. Why: we release often enough (multiple times per week) that a dedicated release branch is just extra overhead. The staging branch serves as our stabilization period.

**How we avoid breaking the release branch (main/staging):**

1. **Branch protection** — `main` and `staging` require passing CI and at least one approving review. No direct commits. No force push. This is non-negotiable.

2. **Feature flags for incomplete work.** If a feature is being deployed but isn't ready to show users, it's behind a flag. The code merges to `main` but is toggled off. This is the core enabler of trunk-based development — you can merge incomplete features without blocking a release.

   ```python
   if feature_flags.is_enabled('new-checkout-flow', user_id):
       return new_checkout_handler(request)
   return legacy_checkout_handler(request)
   ```

3. **Short-lived branches.** Feature branches older than 3 days accumulate merge conflicts and drift from `develop`. We enforce this culturally — PRs that sit for > 3 days get flagged in standup.

4. **Mandatory CI on PRs.** A PR can't be merged if tests fail. This sounds obvious, but we had a period where the CI check was "advisory" — engineers could merge a failing build. We had 3 broken `develop` deployments in one week from this. Mandatory CI is now enforced at the GitHub branch protection level.

5. **Smoke tests on staging before promoting to main.** After deploying to staging, an automated smoke test suite runs. If any smoke test fails, the pipeline blocks the `staging → main` PR pipeline. This caught a broken login flow once — the unit tests passed but the full end-to-end flow was broken due to a config change.

**Real scenario where we almost broke the release branch:**

A developer was working on a "small" refactor. It took 5 days instead of 2. By day 5, `develop` had 20 additional commits. The merge created 8 conflicts. The developer resolved them manually under time pressure. One resolution accidentally reverted a bug fix from day 3. The merged code had a regression. This made it to staging before QA caught it.

After this, we introduced a rule: if a feature branch is more than 2 days behind `develop`, the developer must rebase before the PR can be approved. The shorter the branch lives, the smaller the diff, the less risk in the merge.
