# GitHub Actions — Medium Questions

---

## 1. How do you reuse workflows across repositories?

GitHub Actions provides two mechanisms: **reusable workflows** (call an entire workflow from another) and **composite actions** (reuse a set of steps).

**Option 1: Reusable workflows — call a full workflow from another repo.**

```yaml
# .github/workflows/deploy.yml  (in a shared repo: myorg/shared-workflows)
name: Reusable Deploy Workflow
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-south-1

      - name: Deploy to EKS
        run: |
          helm upgrade --install myapp ./chart \
            --set image.tag=${{ inputs.image-tag }} \
            --namespace ${{ inputs.environment }}
```

```yaml
# .github/workflows/ci.yml  (in a consuming repo: myorg/myapp)
name: CI
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp:${{ github.sha }} .

  deploy:
    needs: build
    uses: myorg/shared-workflows/.github/workflows/deploy.yml@main
    with:
      environment: staging
      image-tag: ${{ github.sha }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

**Option 2: Composite actions — reuse steps within a workflow.**

```yaml
# .github/actions/setup-node/action.yml  (shared action)
name: Setup Node with cache
description: Sets up Node.js with npm cache

inputs:
  node-version:
    description: Node version
    default: '20'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash
```

```yaml
# Any workflow can use this:
- uses: myorg/shared-actions/.github/actions/setup-node@main
  with:
    node-version: '20'
```

**When to use which:**

| Use case | Tool |
|---|---|
| Share an entire deploy/test workflow | Reusable workflow (`workflow_call`) |
| Share a set of steps (setup, auth) | Composite action |
| Share across multiple repos | Store in a dedicated `shared-workflows` repo |

---

## 2. How do you manage large workflow files efficiently?

Large workflow files become hard to read and maintain. Break them apart using reusable workflows, composite actions, and job matrices.

**Strategy 1: Split into multiple workflow files by trigger/purpose.**

```
.github/workflows/
  ci.yml           ← on: pull_request — lint, test, build
  cd-staging.yml   ← on: push to main — deploy to staging
  cd-prod.yml      ← on: release published — deploy to prod
  security.yml     ← on: schedule — weekly security scans
  cleanup.yml      ← on: schedule — nightly image cleanup
```

Each file is focused on one purpose. The CI workflow doesn't contain deploy logic.

**Strategy 2: Use job matrices to replace repeated jobs.**

```yaml
# Before — repeated for each service:
jobs:
  test-service-a:
    runs-on: ubuntu-latest
    steps:
      - run: pytest services/service-a/

  test-service-b:
    runs-on: ubuntu-latest
    steps:
      - run: pytest services/service-b/

# After — one job with matrix:
jobs:
  test:
    strategy:
      matrix:
        service: [service-a, service-b, service-c]
    runs-on: ubuntu-latest
    steps:
      - run: pytest services/${{ matrix.service }}/
```

**Strategy 3: Extract complex steps into composite actions.**

Any sequence of 3+ steps that appears in multiple jobs → extract to a composite action:

```yaml
# Repeated in every job:
- uses: actions/checkout@v4
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE }}
    aws-region: ap-south-1
- run: aws ecr get-login-password | docker login ...

# Replace with:
- uses: ./.github/actions/aws-setup
```

**Strategy 4: Use `workflow_call` to delegate complex jobs.**

```yaml
# ci.yml — clean, readable
jobs:
  test:
    uses: ./.github/workflows/test.yml

  security:
    uses: ./.github/workflows/security-scan.yml

  deploy:
    needs: [test, security]
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
```

---

## 3. What is the difference between public and private workflow repositories?

When sharing workflows or actions across repositories in GitHub Actions, visibility controls who can use them.

**Public repositories:**

Workflows and actions in public repos are accessible to anyone on GitHub — including outside your organisation.

```yaml
# Anyone can use an action from a public repo:
- uses: actions/checkout@v4               # GitHub's public action
- uses: aws-actions/configure-aws-credentials@v4   # AWS's public action
```

Risks: anyone can reference your workflow; if your action has a bug or security issue, all consumers are affected.

**Private repositories:**

Actions and reusable workflows in private repos can only be used by repos within the **same organisation** (if you enable it in org settings), or only the same repo.

```yaml
# Private shared action — only accessible within your org
- uses: myorg/shared-actions/.github/actions/deploy@main
  # This fails if called from outside myorg
```

To allow cross-repo usage within the org:
```
Settings → Actions → General → Allow all actions and reusable workflows
Settings → Actions → General → Allow <org> to use actions and reusable workflows
```

**Internal repositories (GitHub Enterprise):**

A third type — internal repos are visible to all members of the enterprise but not the public:

```yaml
# Internal repo — accessible across all orgs in the enterprise
- uses: myenterprise/platform-workflows/.github/workflows/deploy.yml@main
```

**Comparison:**

| | Public | Private | Internal |
|---|---|---|---|
| Accessible by | Anyone on GitHub | Same repo / same org | All enterprise members |
| Use case | Open source actions | Company-internal tools | Enterprise-wide shared workflows |
| Secret exposure risk | High (public) | Low | Medium |

---

## 4. How do you implement workflow concurrency?

Concurrency controls prevent multiple workflow runs from interfering with each other — especially for deployments where two simultaneous applies could cause race conditions.

**Basic concurrency — cancel in-progress runs on new push.**

```yaml
name: Deploy
on:
  push:
    branches: [main]

concurrency:
  group: deploy-${{ github.ref }}   # One group per branch
  cancel-in-progress: true          # Cancel the old run when a new one starts
```

With this: if two pushes happen in quick succession, the first deploy is cancelled when the second starts. Only the latest commit deploys.

**Deployment concurrency — queue, don't cancel.**

For production deployments, cancelling mid-deploy is dangerous. Queue instead:

```yaml
concurrency:
  group: deploy-prod
  cancel-in-progress: false    # Wait for current deploy to finish, then run
```

This ensures deploys are serialised — never two `terraform apply` or `helm upgrade` running simultaneously against the same environment.

**Per-environment concurrency.**

```yaml
jobs:
  deploy-staging:
    concurrency:
      group: deploy-staging     # Separate group from prod
      cancel-in-progress: true  # Cancel old staging deploys — fast feedback

  deploy-prod:
    concurrency:
      group: deploy-prod
      cancel-in-progress: false # Serialise prod deploys
```

**Concurrency with matrix jobs.**

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        region: [us-east-1, eu-west-1, ap-south-1]
      max-parallel: 2    # Deploy to 2 regions at a time, not all 3 simultaneously
    concurrency:
      group: deploy-${{ matrix.region }}
      cancel-in-progress: false
```

---

## 5. How do you handle failed workflows?

A failed workflow needs to be caught, diagnosed, and either automatically retried or escalated to a human.

**Step 1: Notify on failure.**

```yaml
jobs:
  deploy:
    steps:
      - name: Deploy
        run: helm upgrade --install myapp ./chart

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment FAILED on ${{ github.ref }} by ${{ github.actor }}",
              "attachments": [{
                "color": "danger",
                "text": "Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**Step 2: Automatic retry for transient failures.**

```yaml
- name: Deploy (with retry)
  uses: nick-fields/retry@v3
  with:
    timeout_minutes: 10
    max_attempts: 3
    retry_wait_seconds: 30
    command: helm upgrade --install myapp ./chart
```

Transient failures (API rate limits, network timeouts) often succeed on the second attempt.

**Step 3: Continue-on-error for non-critical steps.**

```yaml
- name: Run integration tests
  id: integration-tests
  continue-on-error: true    # Don't fail the workflow, but capture the result

- name: Upload test report
  if: always()               # Always upload, even if tests failed
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/
```

**Step 4: Manual re-run from GitHub UI or CLI.**

```bash
# Re-run all failed jobs:
gh run rerun <run-id> --failed

# Re-run a specific job:
gh run rerun <run-id> --job <job-id>
```

**Step 5: `if` conditionals for cleanup on failure.**

```yaml
jobs:
  deploy:
    steps:
      - name: Create temp resources
        id: create-temp
        run: terraform apply temp-resources.tf

      - name: Run migration
        run: ./run-migration.sh

      - name: Cleanup temp resources
        if: always()   # Run even if migration failed
        run: terraform destroy temp-resources.tf
```

**Real scenario:** Our nightly security scan workflow was failing intermittently due to Trivy's vulnerability DB download timing out. We added `max_attempts: 3` retry with 60-second wait. Failure rate dropped from 20% to 0%. The Slack alert now fires only for genuine failures (bad code, actual CVE found), not transient network issues.
