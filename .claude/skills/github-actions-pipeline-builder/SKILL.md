---
name: github-actions-pipeline-builder
description: Build production CI/CD pipelines with GitHub Actions. Implements matrix builds, caching, deployments, testing, security scanning. Use for automated testing, deployments, release workflows.
  Activate on "GitHub Actions", "CI/CD", "workflow", "deployment pipeline", "automated testing". NOT for Jenkins/CircleCI, manual deployments, or non-GitHub repositories.
allowed-tools: Read,Write,Edit,Bash
metadata:
  category: DevOps & Site Reliability
  tags:
  - github
  - actions
  - pipeline
  - github-actions
  - ci/cd
  pairs-with:
  - skill: devops-automator
    reason: GitHub Actions is one of the primary CI/CD platforms that DevOps automation targets
  - skill: docker-containerization
    reason: Container builds and registry pushes are the most common GitHub Actions workflow steps
  - skill: git-workflow-expert
    reason: Git branching strategies determine pipeline trigger rules and deployment gates
  - skill: test-automation-expert
    reason: Automated test suites run as CI pipeline stages with matrix builds and caching
---

# GitHub Actions Pipeline Builder

Expert in building production-grade CI/CD pipelines with GitHub Actions that are fast, reliable, and secure.

## When to Use

✅ **Use for**:
- Automated testing on every commit
- Deployment to staging/production
- Docker image building and publishing
- Release automation with versioning
- Security scanning and dependency audits
- Code quality checks (linting, type checking)
- Multi-environment workflows

❌ **NOT for**:
- Non-GitHub repositories (use Jenkins, CircleCI, etc.)
- Complex pipelines better suited for dedicated CI/CD tools
- Self-hosted runners (covered in advanced patterns)

## Quick Decision Tree

```
Does your project need:
├── Testing on every PR? → GitHub Actions
├── Automated deployments? → GitHub Actions
├── Matrix builds (Node 16, 18, 20)? → GitHub Actions
├── Secrets management? → GitHub Actions secrets
├── Multi-cloud deployments? → GitHub Actions + OIDC
└── Sub-second builds? → Consider build caching
```

---

## Technology Selection

### GitHub Actions vs Alternatives

**Why GitHub Actions**:
- **Native integration**: No third-party setup
- **Free for public repos**: 2000 minutes/month for private
- **Matrix builds**: Test multiple versions in parallel
- **Marketplace**: 10,000+ pre-built actions
- **OIDC support**: Keyless cloud deployments

### When to Use Alternatives

| Scenario | Use | Why |
|----------|-----|-----|
| Self-hosted GitLab | GitLab CI | Native integration |
| Complex enterprise workflows | Jenkins | More flexible |
| Bitbucket repos | Bitbucket Pipelines | Native integration |
| Extremely large repos (&gt;10GB) | BuildKite | Better for monorepos |

---

## Common Anti-Patterns

**Anti-Pattern 1: No dependency caching** → Fresh installs waste 2-5 min per build.
```yaml
# ❌  - name: Install dependencies
#       run: npm install
# ✅
- uses: actions/cache@v3
  with: { path: ~/.npm, key: "${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}" }
- run: npm ci
```
Impact: install time 3 min → 30 sec.

---

**Anti-Pattern 2: Duplicate YAML instead of matrix builds** → Copy-paste jobs for each Node version.
```yaml
# ❌ separate test-node-16, test-node-18, test-node-20 jobs
# ✅
jobs:
  test:
    strategy:
      matrix:
        node-version: [16, 18, 20]
    steps:
      - uses: actions/setup-node@v3
        with: { node-version: "${{ matrix.node-version }}", cache: npm }
      - run: npm ci && npm test
```
Impact: 66% less YAML; jobs run in parallel.

---

**Anti-Pattern 3: Secrets in code** → Hardcoded tokens visible in repo history.
```yaml
# ❌ run: API_KEY=sk-abc123 ./deploy.sh
# ✅
- run: ./deploy.sh
  env: { API_KEY: "${{ secrets.PRODUCTION_API_KEY }}" }
```
Impact: prevents credential leaks; use OIDC to eliminate long-lived secrets entirely.

---

**Anti-Pattern 4: No failure notifications** → CI fails silently for hours.
```yaml
# ✅
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: '{"text":"Build failed: ${{ github.event.head_commit.message }}"}'
  env: { SLACK_WEBHOOK_URL: "${{ secrets.SLACK_WEBHOOK }}" }
```
Impact: team learns of failures in seconds, not hours.

---

**Anti-Pattern 5: Full test suite on every commit** → 10-minute feedback loop discourages frequent commits.
```yaml
# ✅ Unit tests on PR, full suite on merge to main
jobs:
  quick-tests:
    if: github.event_name == 'pull_request'
    steps: [{ run: npm run test:unit }]
  full-tests:
    if: github.event_name == 'push'
    steps: [{ run: npm test }]
```
Impact: PR feedback in 2 min vs. 10 min. Alternative: changed-files action for affected-only runs.

---

## Implementation Patterns

### Pattern 1: Basic CI Pipeline

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run typecheck

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

### Pattern 2: Multi-Environment Deployment

```yaml
name: Deploy

on:
  push:
    branches:
      - main        # → staging
      - production  # → production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}  # staging or production

    steps:
      - uses: actions/checkout@v3

      - name: Deploy to ${{ github.ref_name }}
        run: |
          if [ "${{ github.ref_name }}" == "production" ]; then
            ./deploy.sh production
          else
            ./deploy.sh staging
          fi
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### Pattern 3: Release Automation

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'  # Trigger on version tags (v1.0.0)

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write  # Required for creating releases

    steps:
      - uses: actions/checkout@v3

      - name: Build artifacts
        run: npm run build

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/**
          body: |
            ## What's Changed
            See CHANGELOG.md for details.
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Publish to npm
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Pattern 4: Docker Build & Push

```yaml
name: Docker

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            myapp:latest
            myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Production Checklist

```
□ Secrets rotated and stored in GitHub Secrets (not code)
□ Branch protection rules + required status checks enabled
□ Artifact signing (sigstore/cosign) for releases
□ Failure notifications wired (Slack/PagerDuty)
□ Concurrency limits set (cancel in-progress on PR, queue on main)
□ OIDC keyless auth for cloud deployments (no long-lived tokens)
□ Matrix strategy max-parallel capped to avoid runner exhaustion
□ Cache invalidation strategy defined (key rotation on dep changes)
```

---

## When to Use vs Avoid

| Scenario | Use GitHub Actions? |
|----------|---------------------|
| GitHub-hosted repo | ✅ Yes |
| Need matrix builds | ✅ Yes |
| Deploying to AWS/GCP/Azure | ✅ Yes (with OIDC) |
| GitLab repo | ❌ No - use GitLab CI |
| Extremely large monorepo | ⚠️ Maybe - consider BuildKite |
| Need GUI pipeline builder | ❌ No - use Jenkins/Azure DevOps |

---

## References

- `/references/advanced-caching.md` - Cache strategies for faster builds
- `/references/oidc-deployments.md` - Keyless cloud authentication
- `/references/security-hardening.md` - Security best practices

## Scripts

- `scripts/workflow_validator.ts` - Validate YAML syntax locally
- `scripts/action_usage_analyzer.ts` - Find outdated actions

## Assets

- `assets/workflows/` - Ready-to-use workflow templates

---

**This skill guides**: CI/CD pipelines | GitHub Actions workflows | Matrix builds | Caching | Deployments | Release automation

