# CI/CD Workflows for OpenTofu/GCP

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** CI/CD integration patterns for OpenTofu with GCP

This document provides detailed CI/CD workflow templates and optimization strategies for OpenTofu/GCP infrastructure-as-code pipelines.

---

## Table of Contents

1. [Cloud Build Pipeline](#cloud-build-pipeline)
2. [GitHub Actions with Workload Identity](#github-actions-with-workload-identity)
3. [GitLab CI Template](#gitlab-ci-template)
4. [Cost Optimization](#cost-optimization)
5. [Automated Cleanup](#automated-cleanup)
6. [Best Practices](#best-practices)

---

## Cloud Build Pipeline

### Complete Cloud Build Configuration

```yaml
# cloudbuild.yaml
steps:
  # Initialize OpenTofu
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'init'
    entrypoint: 'tofu'
    args: ['init', '-backend-config=bucket=${_STATE_BUCKET}']
    dir: 'environments/${_ENVIRONMENT}'

  # Format check
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'fmt'
    entrypoint: 'tofu'
    args: ['fmt', '-check', '-recursive']
    dir: 'environments/${_ENVIRONMENT}'

  # Validate
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'validate'
    entrypoint: 'tofu'
    args: ['validate']
    dir: 'environments/${_ENVIRONMENT}'
    waitFor: ['init']

  # Trivy security scan
  - name: 'aquasec/trivy:latest'
    id: 'trivy'
    args: ['config', '.', '--severity', 'CRITICAL,HIGH', '--exit-code', '1']
    dir: 'environments/${_ENVIRONMENT}'

  # Checkov policy scan
  - name: 'bridgecrew/checkov:latest'
    id: 'checkov'
    args: ['-d', '.', '--framework', 'terraform', '--compact', '--quiet', '--check', 'CKV_GCP*']
    dir: 'environments/${_ENVIRONMENT}'
    env:
      - 'CHECKOV_OUTPUT_FORMAT=cli'

  # Plan
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'plan'
    entrypoint: 'tofu'
    args: ['plan', '-out=tfplan', '-input=false']
    dir: 'environments/${_ENVIRONMENT}'
    waitFor: ['validate', 'trivy', 'checkov']

  # Scan plan file (deeper analysis)
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'show-plan'
    entrypoint: 'sh'
    args: ['-c', 'tofu show -json tfplan > tfplan.json']
    dir: 'environments/${_ENVIRONMENT}'
    waitFor: ['plan']

  - name: 'bridgecrew/checkov:latest'
    id: 'checkov-plan'
    args: ['-f', 'tfplan.json', '--compact', '--check', 'CKV_GCP*']
    dir: 'environments/${_ENVIRONMENT}'
    waitFor: ['show-plan']

  # Apply (manual approval for prod)
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'apply'
    entrypoint: 'tofu'
    args: ['apply', '-auto-approve', 'tfplan']
    dir: 'environments/${_ENVIRONMENT}'
    waitFor: ['checkov-plan']

substitutions:
  _ENVIRONMENT: 'dev'
  _STATE_BUCKET: 'my-project-tofu-state'

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_MEDIUM'

timeout: '1800s'  # 30 minutes
```

### Cloud Build Trigger Configuration

```yaml
# cloudbuild-trigger.yaml
name: 'tofu-${_ENVIRONMENT}'
description: 'OpenTofu deployment for ${_ENVIRONMENT}'

github:
  owner: 'my-org'
  name: 'infrastructure'
  push:
    branch: '^main$'

includedFiles:
  - 'environments/${_ENVIRONMENT}/**'
  - 'modules/**'

substitutions:
  _ENVIRONMENT: 'prod'
  _STATE_BUCKET: 'my-project-tofu-state'

filename: 'cloudbuild.yaml'

approvalConfig:
  approvalRequired: true  # Require approval for prod
```

### Cloud Build with State Encryption

```yaml
# cloudbuild-encrypted.yaml
steps:
  - name: 'ghcr.io/opentofu/opentofu:latest'
    id: 'init'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        # Configure encryption key
        export TOFU_ENCRYPTION_KEY_KMS="projects/${PROJECT_ID}/locations/global/keyRings/tofu-state/cryptoKeys/state-key"
        tofu init -backend-config=bucket=${_STATE_BUCKET}
    dir: 'environments/${_ENVIRONMENT}'

  # ... rest of steps
```

---

## GitHub Actions with Workload Identity

### Workload Identity Federation Setup

First, create the Workload Identity Pool and Provider:

```hcl
# modules/github-wif/main.tf
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-pool"
  display_name              = "GitHub Actions Pool"
  description               = "Pool for GitHub Actions OIDC authentication"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Provider"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }

  attribute_condition = "assertion.repository_owner == '${var.github_org}'"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

resource "google_service_account" "github_actions" {
  account_id   = "github-actions-tofu"
  display_name = "GitHub Actions OpenTofu"
}

resource "google_service_account_iam_member" "github_wif" {
  service_account_id = google_service_account.github_actions.id
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/${var.github_org}/${var.github_repo}"
}

# Grant necessary permissions
resource "google_project_iam_member" "github_actions" {
  for_each = toset([
    "roles/compute.admin",
    "roles/container.admin",
    "roles/iam.serviceAccountUser",
    "roles/storage.admin",
    "roles/cloudsql.admin",
  ])

  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.github_actions.email}"
}

output "workload_identity_provider" {
  value = google_iam_workload_identity_pool_provider.github.name
}

output "service_account_email" {
  value = google_service_account.github_actions.email
}
```

### GitHub Actions Workflow

```yaml
# .github/workflows/tofu.yml
name: OpenTofu

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write  # Required for Workload Identity

env:
  TOFU_VERSION: '1.11.3'
  WORKLOAD_IDENTITY_PROVIDER: 'projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
  SERVICE_ACCOUNT: 'github-actions-tofu@my-project.iam.gserviceaccount.com'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}

      - name: OpenTofu Format
        run: tofu fmt -check -recursive

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: OpenTofu Init
        run: tofu init
        working-directory: environments/dev

      - name: OpenTofu Validate
        run: tofu validate
        working-directory: environments/dev

      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
      - run: |
          tflint --init
          tflint --enable-plugin=google
        working-directory: environments/dev

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          check: CKV_GCP*
          quiet: true

  test:
    needs: [validate, security-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: Run OpenTofu Tests
        run: |
          tofu init
          tofu test
        working-directory: environments/dev

  plan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: OpenTofu Init
        run: tofu init
        working-directory: environments/prod

      - name: OpenTofu Plan
        run: tofu plan -out=tfplan -input=false
        working-directory: environments/prod

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: environments/prod/tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: environments/prod

      - name: OpenTofu Init
        run: tofu init
        working-directory: environments/prod

      - name: OpenTofu Apply
        run: tofu apply -auto-approve tfplan
        working-directory: environments/prod
```

---

## GitLab CI Template

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - security
  - test
  - plan
  - apply

variables:
  TOFU_VERSION: "1.11.3"
  TF_ROOT: ${CI_PROJECT_DIR}/environments/${ENVIRONMENT}
  GOOGLE_APPLICATION_CREDENTIALS: ${CI_PROJECT_DIR}/gcp-key.json

.tofu_template:
  image: ghcr.io/opentofu/opentofu:${TOFU_VERSION}
  before_script:
    - cd ${TF_ROOT}
    - echo ${GCP_SERVICE_ACCOUNT_KEY} | base64 -d > ${GOOGLE_APPLICATION_CREDENTIALS}
    - tofu init

validate:
  extends: .tofu_template
  stage: validate
  script:
    - tofu fmt -check -recursive
    - tofu validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

security-scan:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - checkov -d ${TF_ROOT} --framework terraform --check CKV_GCP* --compact
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  allow_failure: false

trivy-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy config ${TF_ROOT} --severity CRITICAL,HIGH --exit-code 1
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

test:
  extends: .tofu_template
  stage: test
  script:
    - tofu test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

plan:
  extends: .tofu_template
  stage: plan
  script:
    - tofu plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

apply:
  extends: .tofu_template
  stage: apply
  script:
    - tofu apply tfplan
  dependencies:
    - plan
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  when: manual
  environment:
    name: ${ENVIRONMENT}
```

---

## Cost Optimization

### Strategy

1. **Use mocking for PR validation** (free)
2. **Run integration tests only on main branch** (controlled cost)
3. **Implement auto-cleanup** (prevent orphaned resources)
4. **Label all test resources** (track spending)

### Conditional Test Execution

```yaml
# GitHub Actions
test:
  runs-on: ubuntu-latest
  steps:
    - name: Run Unit Tests (Mocked)
      run: tofu test

    - name: Run Integration Tests
      if: github.ref == 'refs/heads/main'
      run: |
        cd tests/integration
        go test -v -timeout 30m
```

### Cost-Aware Test Labels

```hcl
locals {
  test_labels = {
    environment = "test"
    ttl         = "2h"
    created-by  = "ci"
    job-id      = var.ci_job_id
  }
}

resource "google_compute_instance" "test" {
  name         = "test-${var.ci_job_id}"
  machine_type = "e2-micro"  # Use smallest size for tests
  zone         = var.zone

  labels = local.test_labels

  # ...
}
```

---

## Automated Cleanup

### Cleanup Script

```bash
#!/bin/bash
# scripts/cleanup-test-resources.sh

# Find and delete resources older than 2 hours with test label
PROJECT_ID="${1:-$(gcloud config get-value project)}"
MAX_AGE_HOURS="${2:-2}"

echo "Cleaning up test resources in project: ${PROJECT_ID}"
echo "Max age: ${MAX_AGE_HOURS} hours"

# Calculate cutoff time
CUTOFF=$(date -u -d "${MAX_AGE_HOURS} hours ago" +%Y-%m-%dT%H:%M:%SZ)

# Clean up Compute Instances
echo "Checking compute instances..."
gcloud compute instances list \
  --project="${PROJECT_ID}" \
  --filter="labels.environment=test AND creationTimestamp<${CUTOFF}" \
  --format="value(name,zone)" | \
while read NAME ZONE; do
  echo "Deleting instance: ${NAME} in ${ZONE}"
  gcloud compute instances delete "${NAME}" \
    --project="${PROJECT_ID}" \
    --zone="${ZONE}" \
    --quiet
done

# Clean up GKE Clusters
echo "Checking GKE clusters..."
gcloud container clusters list \
  --project="${PROJECT_ID}" \
  --filter="resourceLabels.environment=test AND createTime<${CUTOFF}" \
  --format="value(name,location)" | \
while read NAME LOCATION; do
  echo "Deleting cluster: ${NAME} in ${LOCATION}"
  gcloud container clusters delete "${NAME}" \
    --project="${PROJECT_ID}" \
    --location="${LOCATION}" \
    --quiet
done

# Clean up Cloud SQL instances
echo "Checking Cloud SQL instances..."
gcloud sql instances list \
  --project="${PROJECT_ID}" \
  --filter="settings.userLabels.environment=test" \
  --format="value(name)" | \
while read NAME; do
  echo "Deleting SQL instance: ${NAME}"
  gcloud sql instances delete "${NAME}" \
    --project="${PROJECT_ID}" \
    --quiet
done

echo "Cleanup complete"
```

### Scheduled Cleanup (Cloud Build)

```yaml
# cloudbuild-cleanup.yaml
steps:
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'cleanup'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        chmod +x scripts/cleanup-test-resources.sh
        ./scripts/cleanup-test-resources.sh ${PROJECT_ID} 2

options:
  logging: CLOUD_LOGGING_ONLY

# Schedule with Cloud Scheduler:
# gcloud scheduler jobs create http cleanup-test-resources \
#   --schedule="0 */2 * * *" \
#   --location=us-central1 \
#   --uri="https://cloudbuild.googleapis.com/v1/projects/${PROJECT_ID}/triggers/${TRIGGER_ID}:run" \
#   --oauth-service-account-email="${SERVICE_ACCOUNT}"
```

### GitHub Actions Cleanup

```yaml
# .github/workflows/cleanup.yml
name: Cleanup Test Resources

on:
  schedule:
    - cron: '0 */2 * * *'  # Every 2 hours
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.SERVICE_ACCOUNT }}

      - name: Setup gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Run Cleanup Script
        run: ./scripts/cleanup-test-resources.sh ${{ vars.GCP_PROJECT_ID }} 2
```

---

## Best Practices

### 1. Separate Environments

```yaml
# Different workflows for different environments
.github/workflows/
  tofu-dev.yml
  tofu-staging.yml
  tofu-prod.yml
```

Or use reusable workflows:

```yaml
# .github/workflows/tofu-deploy.yml (reusable)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  deploy:
    environment: ${{ inputs.environment }}
    # ... deployment steps
```

### 2. Require Approvals for Production

```yaml
# GitHub Actions
apply:
  environment:
    name: production
    # Requires manual approval in GitHub

# Cloud Build
approvalConfig:
  approvalRequired: true
```

### 3. Use Remote State with Locking

```hcl
tofu {
  backend "gcs" {
    bucket = "my-project-tofu-state"
    prefix = "prod"
  }
}
```

GCS backend handles locking automatically.

### 4. Implement State Locking Timeout

```yaml
# In CI, use -lock-timeout to handle concurrent runs
- name: OpenTofu Apply
  run: tofu apply -lock-timeout=10m tfplan
```

### 5. Cache OpenTofu Plugins

```yaml
# GitHub Actions
- name: Cache OpenTofu Plugins
  uses: actions/cache@v4
  with:
    path: |
      ~/.terraform.d/plugin-cache
    key: ${{ runner.os }}-tofu-${{ hashFiles('**/.terraform.lock.hcl') }}
```

### 6. Security Scanning in CI

```yaml
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Run Trivy
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'config'
        scan-ref: '.'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'

    - name: Run Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: .
        framework: terraform
        check: CKV_GCP*
```

---

## Troubleshooting

### Issue: Workload Identity authentication fails

**Cause:** Missing permissions or incorrect configuration

**Solution:**

```bash
# Verify WIF pool exists
gcloud iam workload-identity-pools describe github-pool \
  --location=global

# Verify provider exists
gcloud iam workload-identity-pools providers describe github-provider \
  --workload-identity-pool=github-pool \
  --location=global

# Check service account binding
gcloud iam service-accounts get-iam-policy github-actions-tofu@project.iam.gserviceaccount.com
```

### Issue: Cloud Build times out

**Cause:** Long-running operations or slow providers

**Solution:**

```yaml
# Increase timeout
timeout: '3600s'  # 1 hour

# Use faster machine type
options:
  machineType: 'E2_HIGHCPU_8'
```

### Issue: State lock not released

**Cause:** Previous run crashed without releasing lock

**Solution:**

```bash
# GCS backend - locks are stored in the bucket
# Check for lock file
gsutil ls gs://my-tofu-state/.terraform.lock.hcl

# Force unlock (use with caution!)
tofu force-unlock LOCK_ID
```

---

**Back to:** [Main Skill File](../SKILL.md)
