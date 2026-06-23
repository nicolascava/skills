# Quick Reference

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** Command cheat sheets and decision flowcharts for OpenTofu/GCP

This document provides quick lookup tables, command references, and decision flowcharts for rapid consultation during development.

---

## Table of Contents

1. [Command Cheat Sheet](#command-cheat-sheet)
2. [Decision Flowchart](#decision-flowchart)
3. [Version-Specific Guidance](#version-specific-guidance)
4. [Troubleshooting Guide](#troubleshooting-guide)
5. [Migration Paths](#migration-paths)
6. [GCS Backend Configuration](#gcs-backend-configuration)

---

## Command Cheat Sheet

### Static Analysis

```bash
# Format and validate
tofu fmt -recursive -check
tofu validate

# Linting with GCP rules
tflint --init
tflint --enable-plugin=google

# Security scanning
trivy config . --severity CRITICAL,HIGH
checkov -d . --framework terraform --check CKV_GCP*
```

### Native Tests (1.6+)

```bash
# Run all tests
tofu test

# Run tests in specific directory
tofu test -test-directory=tests/unit/

# Verbose output
tofu test -verbose

# Run specific test file
tofu test -filter=tests/vpc_test.tftest.hcl
```

### Plan and Apply

```bash
# Generate and review plan
tofu plan -out tfplan

# Convert plan to JSON for analysis
tofu show -json tfplan > tfplan.json

# Check for specific changes
tofu show tfplan | grep "will be created"

# Apply with plan file
tofu apply tfplan

# Apply with auto-approve (CI/CD only)
tofu apply -auto-approve
```

### State Management

```bash
# List resources in state
tofu state list

# Show specific resource
tofu state show google_compute_instance.web

# Move resource (refactoring)
tofu state mv google_compute_instance.old google_compute_instance.new

# Import existing resource
tofu import google_compute_instance.web projects/my-project/zones/us-central1-a/instances/web

# Remove from state (doesn't destroy)
tofu state rm google_compute_instance.web
```

### Workspace Management

```bash
# List workspaces
tofu workspace list

# Create new workspace
tofu workspace new staging

# Select workspace
tofu workspace select prod

# Show current workspace
tofu workspace show
```

---

## Decision Flowchart

### Testing Approach Selection

```
Need to test OpenTofu code?
│
├─ Just syntax/format?
│  └─ tofu validate + fmt
│
├─ Static security scan?
│  └─ trivy + checkov
│
├─ OpenTofu 1.6+?
│  ├─ Simple logic test?
│  │  └─ Native tofu test
│  │
│  └─ Complex integration?
│     └─ Terratest
│
└─ Complex multi-service?
   └─ Terratest with GCP SDK
```

### Module Development Workflow

```
1. Plan
   ├─ Define inputs (variables.tf)
   ├─ Define outputs (outputs.tf)
   └─ Document purpose (README.md)

2. Implement
   ├─ Create resources (main.tf)
   ├─ Pin versions (versions.tf)
   └─ Add examples (examples/simple, examples/complete)

3. Test
   ├─ Static analysis (validate, fmt, lint)
   ├─ Security scan (trivy, checkov)
   ├─ Unit tests (native or Terratest)
   └─ Integration tests (examples/)

4. Document
   ├─ Update README with usage
   ├─ Document inputs/outputs
   └─ Add CHANGELOG

5. Publish
   ├─ Tag version (git tag v1.0.0)
   ├─ Push to registry
   └─ Announce changes
```

### GCP Resource Selection

```
Need compute resources?
│
├─ Containerized workload?
│  ├─ Kubernetes needed?
│  │  └─ GKE (google_container_cluster)
│  │
│  └─ Simple container?
│     └─ Cloud Run (google_cloud_run_service)
│
├─ VM needed?
│  ├─ Single instance?
│  │  └─ Compute Instance (google_compute_instance)
│  │
│  └─ Auto-scaling?
│     └─ Instance Group + Autoscaler
│
└─ Serverless?
   └─ Cloud Functions (google_cloudfunctions2_function)
```

---

## Version-Specific Guidance

### OpenTofu 1.6+

- ✅ Native `tofu test` command
- ✅ Built-in test framework
- ✅ Consider migrating simple tests from Terratest

### OpenTofu 1.7+

- ✅ Mock providers for unit testing
- ✅ **State encryption** with GCP KMS
- ✅ Reduce costs with mocking
- ✅ Faster test iteration

### OpenTofu 1.8+

- ✅ **Early variable evaluation**
- ✅ Variables in backend configuration
- ✅ Variables in module source

### OpenTofu 1.9+

- ✅ Cross-variable validation
- ✅ Reference other variables in validation blocks

### OpenTofu 1.11+

- ✅ **`enabled` meta-argument** for cleaner conditionals
- ✅ **Ephemeral resources** - secrets never in state
- ✅ Write-only arguments

---

## Troubleshooting Guide

### Issue: Tests fail in CI but pass locally

**Symptoms:**
- Tests pass on your machine
- Same tests fail in Cloud Build/GitHub Actions

**Common Causes:**
1. Different OpenTofu/provider versions
2. Different environment variables
3. Different GCP credentials/permissions

**Solution:**

```hcl
# versions.tf - Pin versions explicitly
tofu {
  required_version = "~> 1.11"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}
```

### Issue: State locking errors

**Symptoms:**
- Error: "Error acquiring the state lock"
- Concurrent runs failing

**Solution:**

```bash
# Force unlock (use with caution!)
tofu force-unlock LOCK_ID

# In CI, use lock timeout
tofu apply -lock-timeout=10m tfplan
```

### Issue: Parallel tests conflict

**Symptoms:**
- Tests fail when run in parallel
- Error: "Resource already exists"

**Cause:** Resource naming collisions

**Solution:**

```go
// In Terratest - use unique identifiers
import "github.com/gruntwork-io/terratest/modules/random"

uniqueId := random.UniqueId()
instanceName := fmt.Sprintf("test-instance-%s", uniqueId)
```

### Issue: High test costs

**Symptoms:**
- GCP bill increasing from tests
- Many orphaned resources in test project

**Solutions:**

1. **Use mocking for unit tests** (OpenTofu 1.7+)
   ```hcl
   mock_provider "google" { ... }
   ```

2. **Implement resource TTL labels**
   ```hcl
   labels = {
     environment = "test"
     ttl         = "2h"
   }
   ```

3. **Run integration tests only on main branch**
   ```yaml
   if: github.ref == 'refs/heads/main'
   ```

4. **Use smaller machine types**
   ```hcl
   machine_type = "e2-micro"  # Not "n2-standard-4"
   ```

### Issue: GCS backend access denied

**Symptoms:**
- Error: "Error configuring backend"
- 403 Forbidden on state bucket

**Solution:**

```bash
# Verify service account permissions
gcloud storage buckets get-iam-policy gs://my-tofu-state

# Grant access
gcloud storage buckets add-iam-policy-binding gs://my-tofu-state \
  --member="serviceAccount:my-sa@project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

### Issue: KMS encryption errors

**Symptoms:**
- Error: "Error encrypting state"
- KMS key access denied

**Solution:**

```hcl
# Grant encrypt/decrypt permissions
resource "google_kms_crypto_key_iam_member" "state" {
  crypto_key_id = google_kms_crypto_key.state.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:${var.tofu_service_account}"
}
```

---

## Migration Paths

### From Manual Testing → Automated

**Phase 1:** Static analysis
```bash
tofu validate
tofu fmt -check
trivy config .
```

**Phase 2:** Plan review
```bash
tofu plan -out=tfplan
# Manual review
```

**Phase 3:** Automated tests
- Native tests (1.6+)
- OR Terratest

**Phase 4:** CI/CD integration
- Cloud Build/GitHub Actions
- Automated apply on main branch

### From Terratest → Native Tests (1.6+)

**Strategy:** Gradual migration

1. **Keep Terratest for:**
   - Complex integration tests
   - Multi-step workflows
   - Cross-service tests

2. **Migrate to native tests:**
   - Simple unit tests
   - Logic validation
   - Mock-friendly tests

3. **During transition:**
   - Maintain both frameworks
   - Gradually increase native test coverage
   - Remove Terratest tests once replaced

**Example:** Mixed approach

```
tests/
├── unit/                    # Native tests
│   └── validation.tftest.hcl
└── integration/             # Terratest
    └── complete_test.go
```

---

## GCS Backend Configuration

### Basic GCS Backend

```hcl
tofu {
  backend "gcs" {
    bucket = "my-project-tofu-state"
    prefix = "prod"
  }
}
```

### With State Encryption (OpenTofu 1.7+)

```hcl
tofu {
  encryption {
    method "gcp_kms" "state_key" {
      kms_encryption_key = "projects/my-project/locations/global/keyRings/tofu-state/cryptoKeys/state-key"
    }
    state {
      method = method.gcp_kms.state_key
    }
    plan {
      method = method.gcp_kms.state_key
    }
  }

  backend "gcs" {
    bucket = "my-project-tofu-state"
    prefix = "prod"
  }
}
```

### With Variable Interpolation (OpenTofu 1.8+)

```hcl
variable "environment" {
  type = string
}

tofu {
  backend "gcs" {
    bucket = "my-project-tofu-state"
    prefix = var.environment
  }
}
```

### State Bucket Setup

```hcl
# Create state bucket with security controls
resource "google_storage_bucket" "tofu_state" {
  name     = "${var.project_id}-tofu-state"
  location = var.region

  versioning {
    enabled = true
  }

  uniform_bucket_level_access = true

  lifecycle_rule {
    condition {
      num_newer_versions = 5
    }
    action {
      type = "Delete"
    }
  }
}
```

---

## Pre-Commit Checklist

### Formatting & Validation

Run these commands before every commit:

```bash
# Format all OpenTofu files
tofu fmt -recursive

# Validate configuration
tofu validate

# Security scan
trivy config . --severity CRITICAL,HIGH
```

### Naming Convention Review

- [ ] GCP resource names use hyphens (kebab-case)
- [ ] HCL identifiers use underscores (snake_case)
- [ ] Resource names don't repeat resource type (no `google_compute_network.main_network`)
- [ ] Resource names use singular nouns (not `google_compute_instance.web_servers`)
- [ ] Single-instance resources named `this` or descriptive name
- [ ] Variables have plural names for lists/maps (`subnet_ids` not `subnet_id`)
- [ ] Variable names avoid double negatives (use `enabled` not `disabled`)
- [ ] All variables have descriptions
- [ ] All outputs have descriptions

### Code Structure Review

- [ ] `count`/`for_each`/`enabled` at top of resource blocks (blank line after)
- [ ] `labels` as last real argument in resources
- [ ] `depends_on` after labels (if used)
- [ ] `lifecycle` at end of resource (if used)
- [ ] Variables ordered: description → type → default → validation → nullable

### GCP-Specific Review

- [ ] Required labels applied (environment, project, managed-by, owner, cost-center)
- [ ] No primitive IAM roles (Owner, Editor, Viewer)
- [ ] No default network usage
- [ ] No 0.0.0.0/0 in firewall source_ranges (unless intended)
- [ ] CMEK encryption for sensitive data
- [ ] Private IP for Cloud SQL

---

## Common Patterns

### Resource Naming

```hcl
# ✅ Good: Descriptive, contextual (GCP uses hyphens)
resource "google_compute_instance" "web-server" { }
resource "google_storage_bucket" "application-logs" { }

# ❌ Bad: Generic
resource "google_compute_instance" "main" { }
resource "google_storage_bucket" "bucket" { }
```

### Variable Naming

```hcl
# ✅ Good: Context-specific
var.vpc_cidr_range
var.database_tier

# ❌ Bad: Generic
var.cidr
var.tier
```

### File Organization

```
Standard module structure:
├── main.tf          # Primary resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── versions.tf      # Provider versions
└── README.md        # Documentation
```

---

**Back to:** [Main Skill File](../SKILL.md)
