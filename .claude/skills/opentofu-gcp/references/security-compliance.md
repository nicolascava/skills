# Security & Compliance for GCP

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** Security best practices and compliance patterns for OpenTofu/GCP

This document provides security hardening guidance, three-tier scanning integration, and compliance automation strategies for GCP infrastructure-as-code.

---

## Table of Contents

1. [Three-Tier Security Scanning](#three-tier-security-scanning)
2. [State Encryption with GCP KMS](#state-encryption-with-gcp-kms)
3. [Secrets Management](#secrets-management)
4. [Common Security Issues](#common-security-issues)
5. [GCP IAM Best Practices](#gcp-iam-best-practices)
6. [VPC Service Controls](#vpc-service-controls)
7. [Compliance Testing](#compliance-testing)
8. [State File Security](#state-file-security)

---

## Three-Tier Security Scanning

### Overview

| Tool | Type | GCP Checks | When to Use |
|------|------|-----------|-------------|
| **Trivy** | IaC static analysis | AVD-GCP-* checks | Pre-commit, CI/CD - scan code before apply |
| **Checkov** | IaC policy scanner | CKV_GCP_* checks | CI/CD - policy compliance, graph analysis |
| **Prowler** | Cloud posture | 100+ GCP controls | Post-deploy - audit deployed infrastructure |

### Trivy for IaC Scanning (Proactive)

**Install:**

```bash
# macOS
brew install trivy

# Linux
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Docker
docker pull aquasec/trivy
```

**Basic Usage:**

```bash
# Scan OpenTofu configuration files
trivy config . --severity CRITICAL,HIGH

# Scan with specific exit code for CI
trivy config . --severity CRITICAL,HIGH --exit-code 1

# Scan plan file before apply
tofu plan -out=tfplan
tofu show -json tfplan > tfplan.json
trivy config tfplan.json

# GCP-specific checks only
trivy config . --severity CRITICAL,HIGH --ignore-unfixed
```

**Common GCP Trivy Checks (AVD-GCP-*):**

| Check ID | Description | Severity |
|----------|-------------|----------|
| AVD-GCP-0013 | Compute disk encryption | HIGH |
| AVD-GCP-0051 | IAM service account key rotation | MEDIUM |
| AVD-GCP-0025 | GKE private cluster enabled | HIGH |
| AVD-GCP-0029 | Cloud SQL public IP | CRITICAL |
| AVD-GCP-0031 | GCS bucket public access | HIGH |

**Configuration (.trivy.yaml):**

```yaml
# .trivy.yaml
severity:
  - CRITICAL
  - HIGH
  - MEDIUM

scan:
  scanners:
    - misconfig

misconfig:
  terraform:
    vars:
      - terraform.tfvars

exit-code: 1
```

### Checkov for Policy Compliance (Proactive)

**Install:**

```bash
pip install checkov
```

**Basic Usage:**

```bash
# Scan all OpenTofu files
checkov -d . --framework terraform

# Scan plan file for deeper analysis
tofu plan -out=tfplan
tofu show -json tfplan > tfplan.json
checkov -f tfplan.json

# GCP-specific checks only
checkov -d . --check CKV_GCP*

# Skip known exceptions
checkov -d . --skip-check CKV_GCP_11,CKV_GCP_25

# Output SARIF for GitHub Security tab
checkov -d . -o sarif > checkov.sarif

# Generate JSON report
checkov -d . -o json > checkov-report.json
```

**Common GCP Checkov Checks (CKV_GCP_*):**

| Check ID | Description | Severity |
|----------|-------------|----------|
| `CKV_GCP_6` | Cloud SQL DB not public | HIGH |
| `CKV_GCP_11` | Cloud SQL not open to 0.0.0.0/0 | CRITICAL |
| `CKV_GCP_26` | GKE pod security policy enabled | MEDIUM |
| `CKV_GCP_1` | GKE basic auth disabled | HIGH |
| `CKV_GCP_12` | Network policy enabled for GKE | MEDIUM |
| `CKV_GCP_18` | GKE private nodes | HIGH |
| `CKV_GCP_21` | BigQuery encryption with CMEK | MEDIUM |
| `CKV_GCP_29` | Cloud SQL database flags | MEDIUM |

**Skip List Configuration (.checkov.yaml):**

```yaml
# .checkov.yaml
compact: true
framework:
  - terraform
skip-check:
  - CKV_GCP_25  # Example: Skip if not applicable
soft-fail-on:
  - CKV_GCP_26  # Warning only for pod security policy
```

### Prowler for Cloud Posture (Reactive)

Prowler audits **deployed** GCP infrastructure, not just code.

**Install:**

```bash
pip install prowler

# Or with Docker
docker pull toniblyx/prowler
```

**Basic Usage:**

```bash
# Full GCP security assessment
prowler gcp --project-id my-project

# CIS GCP Benchmark
prowler gcp --project-id my-project --compliance cis_gcp

# Specific service checks
prowler gcp --project-id my-project --service gke,cloudsql,iam

# Output for Security Command Center integration
prowler gcp --project-id my-project --output-formats json-ocsf

# Save to GCS
prowler gcp --project-id my-project --output-directory gs://security-reports/prowler/
```

**Prowler GCP Service Checks:**

| Service | Example Checks |
|---------|----------------|
| `iam` | Service account keys, role bindings, admin access |
| `gke` | Private cluster, network policy, RBAC |
| `cloudsql` | Encryption, public IP, backups |
| `compute` | Serial port disabled, OS login, disk encryption |
| `storage` | Public buckets, versioning, encryption |
| `bigquery` | Dataset access, encryption |
| `kms` | Key rotation, access policies |

**Scheduled Prowler Assessment (Cloud Build):**

```yaml
# cloudbuild-prowler.yaml
steps:
  - name: 'python:3.11'
    id: 'prowler'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        pip install prowler
        prowler gcp \
          --project-id ${PROJECT_ID} \
          --compliance cis_gcp \
          --output-formats json-ocsf \
          --output-directory /workspace/prowler-output

  - name: 'gcr.io/cloud-builders/gsutil'
    id: 'upload'
    args:
      - 'cp'
      - '-r'
      - '/workspace/prowler-output/*'
      - 'gs://${PROJECT_ID}-security-reports/prowler/${BUILD_ID}/'

# Schedule with Cloud Scheduler
# gcloud scheduler jobs create http prowler-weekly \
#   --schedule="0 2 * * 0" \
#   --uri="https://cloudbuild.googleapis.com/v1/projects/${PROJECT_ID}/triggers/${TRIGGER_ID}:run"
```

---

## State Encryption with GCP KMS

### Configure Client-Side Encryption

OpenTofu 1.7+ supports client-side state encryption using GCP KMS.

**Step 1: Create KMS Key**

```hcl
# Create KMS keyring and key
resource "google_kms_key_ring" "tofu_state" {
  name     = "tofu-state"
  location = "global"
}

resource "google_kms_crypto_key" "state_key" {
  name     = "state-key"
  key_ring = google_kms_key_ring.tofu_state.id

  rotation_period = "7776000s"  # 90 days

  lifecycle {
    prevent_destroy = true
  }
}
```

**Step 2: Configure State Encryption**

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
    bucket = "my-tofu-state"
    prefix = "prod"
  }
}
```

**Step 3: Grant Access**

```hcl
resource "google_kms_crypto_key_iam_member" "state_encrypter" {
  crypto_key_id = google_kms_crypto_key.state_key.id
  role          = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
  member        = "serviceAccount:${var.tofu_service_account}"
}
```

---

## Secrets Management

### Google Secret Manager Pattern

```hcl
# Create secret
resource "google_secret_manager_secret" "db_password" {
  secret_id = "prod-database-password"

  replication {
    auto {}
  }

  labels = local.required_labels
}

# Generate secure password
resource "random_password" "db_password" {
  length  = 32
  special = true
}

# Store password version
resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = random_password.db_password.result
}

# Grant access to service account
resource "google_secret_manager_secret_iam_member" "app_access" {
  secret_id = google_secret_manager_secret.db_password.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.app.email}"
}
```

### Ephemeral Resources (OpenTofu 1.11+)

**Secrets that never touch state:**

```hcl
# Fetch secret - value never stored in state
ephemeral "google_secret_manager_secret_version" "db_password" {
  secret = "projects/my-project/secrets/db-password"
}

resource "google_sql_database_instance" "main" {
  name             = "main-instance"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier = "db-f1-micro"
  }

  # Ephemeral value - never stored in state
  root_password = ephemeral.google_secret_manager_secret_version.db_password.secret_data
}
```

### Reference Secrets in Modules

```hcl
# Reference existing secret
data "google_secret_manager_secret_version" "db_password" {
  secret = "prod-database-password"
}

resource "google_sql_database_instance" "main" {
  # ...
  root_password = data.google_secret_manager_secret_version.db_password.secret_data
}
```

---

## Common Security Issues

### ❌ DON'T: Store Secrets in Variables

```hcl
# BAD: Secret in plaintext
variable "database_password" {
  type    = string
  default = "SuperSecret123!"  # ❌ Never do this
}
```

### ✅ DO: Use Secret Manager

```hcl
# Good: Reference secrets from Secret Manager
data "google_secret_manager_secret_version" "db_password" {
  secret = "prod-database-password"
}

resource "google_sql_database_instance" "main" {
  root_password = data.google_secret_manager_secret_version.db_password.secret_data
}
```

### ❌ DON'T: Use Default Network

```hcl
# BAD: Default network has permissive firewall rules
resource "google_compute_instance" "app" {
  name = "app"
  network_interface {
    network = "default"  # ❌ Avoid default network
  }
}
```

### ✅ DO: Create Dedicated VPC

```hcl
# Good: Custom VPC with private subnets
resource "google_compute_network" "this" {
  name                    = "main-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "private" {
  name          = "private"
  ip_cidr_range = "10.0.1.0/24"
  network       = google_compute_network.this.id
  region        = var.region

  private_ip_google_access = true
}
```

### ❌ DON'T: Skip Encryption

```hcl
# BAD: Unencrypted Cloud SQL
resource "google_sql_database_instance" "main" {
  name             = "main"
  database_version = "POSTGRES_15"
  # ❌ No CMEK encryption
}
```

### ✅ DO: Enable CMEK Encryption

```hcl
# Good: Cloud SQL with CMEK
resource "google_sql_database_instance" "main" {
  name             = "main"
  database_version = "POSTGRES_15"

  encryption_key_name = google_kms_crypto_key.sql.id

  settings {
    tier = "db-f1-micro"
  }

  depends_on = [google_kms_crypto_key_iam_member.sql]
}
```

### ❌ DON'T: Open Firewall to Internet

```hcl
# BAD: Firewall open to internet
resource "google_compute_firewall" "allow_all" {
  name    = "allow-all"
  network = google_compute_network.this.name

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }

  source_ranges = ["0.0.0.0/0"]  # ❌ Never do this
}
```

### ✅ DO: Use Least-Privilege Firewall Rules

```hcl
# Good: Restrict to specific ports and sources
resource "google_compute_firewall" "allow_https" {
  name    = "allow-https"
  network = google_compute_network.this.name

  allow {
    protocol = "tcp"
    ports    = ["443"]
  }

  source_ranges = ["10.0.0.0/8"]  # ✅ Internal only

  target_tags = ["web-server"]
}
```

---

## GCP IAM Best Practices

### ❌ DON'T: Use Primitive Roles

```hcl
# BAD: Primitive roles are too broad
resource "google_project_iam_member" "bad" {
  project = var.project_id
  role    = "roles/owner"  # ❌ Never use Owner/Editor/Viewer
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

### ✅ DO: Use Predefined or Custom Roles

```hcl
# Good: Predefined roles with minimal permissions
resource "google_project_iam_member" "storage_viewer" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}

# Good: Custom role for specific needs
resource "google_project_iam_custom_role" "log_reader" {
  role_id     = "logReader"
  title       = "Log Reader"
  description = "Read-only access to logs"

  permissions = [
    "logging.logEntries.list",
    "logging.logs.list",
  ]
}
```

### Workload Identity Federation

**For GitHub Actions CI/CD:**

```hcl
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-pool"
  display_name              = "GitHub Actions Pool"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Provider"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.repository" = "assertion.repository"
  }

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  attribute_condition = "assertion.repository_owner == 'my-org'"
}

resource "google_service_account_iam_member" "github_wif" {
  service_account_id = google_service_account.cicd.id
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/my-org/my-repo"
}
```

---

## VPC Service Controls

### Service Perimeter

```hcl
resource "google_access_context_manager_service_perimeter" "main" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/main"
  title  = "Main Perimeter"

  status {
    resources = [
      "projects/${var.project_number}"
    ]

    restricted_services = [
      "storage.googleapis.com",
      "bigquery.googleapis.com",
      "secretmanager.googleapis.com",
    ]

    vpc_accessible_services {
      enable_restriction = true
      allowed_services   = ["RESTRICTED-SERVICES"]
    }
  }
}
```

---

## Compliance Testing

### Policy as Code with OPA

```rego
# policy/gcp_security.rego
package terraform.gcp

# Deny Cloud SQL with public IP
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "google_sql_database_instance"
  resource.change.after.settings[_].ip_configuration[_].ipv4_enabled == true

  msg := sprintf("Cloud SQL instance '%s' must not have public IP enabled", [resource.address])
}

# Require CMEK for Cloud SQL
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "google_sql_database_instance"
  not resource.change.after.encryption_key_name

  msg := sprintf("Cloud SQL instance '%s' must use CMEK encryption", [resource.address])
}

# Require labels
deny[msg] {
  resource := input.resource_changes[_]
  types_requiring_labels := [
    "google_compute_instance",
    "google_container_cluster",
    "google_sql_database_instance",
    "google_storage_bucket"
  ]
  resource.type == types_requiring_labels[_]
  not resource.change.after.labels.environment

  msg := sprintf("Resource '%s' must have 'environment' label", [resource.address])
}
```

### Run OPA

```bash
# Generate plan JSON
tofu plan -out=tfplan
tofu show -json tfplan > tfplan.json

# Evaluate policy
opa eval --input tfplan.json --data policy/ "data.terraform.gcp.deny"
```

---

## State File Security

### Secure GCS Backend

```hcl
# Create state bucket with security controls
resource "google_storage_bucket" "tofu_state" {
  name     = "${var.project_name}-tofu-state"
  location = var.region

  # Prevent accidental deletion
  force_destroy = false

  # Enable versioning for recovery
  versioning {
    enabled = true
  }

  # Lifecycle rule to clean old versions
  lifecycle_rule {
    condition {
      num_newer_versions = 5
    }
    action {
      type = "Delete"
    }
  }

  # Uniform bucket-level access (no ACLs)
  uniform_bucket_level_access = true

  # CMEK encryption
  encryption {
    default_kms_key_name = google_kms_crypto_key.state.id
  }

  labels = local.required_labels
}

# Block public access
resource "google_storage_bucket_iam_member" "state_admin" {
  bucket = google_storage_bucket.tofu_state.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${var.tofu_service_account}"
}
```

### Restrict State Access

```hcl
# Limit who can access state
resource "google_storage_bucket_iam_binding" "state_access" {
  bucket = google_storage_bucket.tofu_state.name
  role   = "roles/storage.objectViewer"

  members = [
    "group:infrastructure-team@company.com",
    "serviceAccount:${var.cicd_service_account}",
  ]
}
```

---

## Compliance Checklists

### CIS GCP Benchmark

- [ ] Enable OS Login for compute instances
- [ ] Disable serial port access
- [ ] Use VPC flow logs
- [ ] Enable Cloud Audit Logs
- [ ] Use CMEK for sensitive data
- [ ] Enable VPC Service Controls
- [ ] Use private GKE clusters
- [ ] Enable binary authorization

### SOC 2 Compliance

- [ ] Encryption at rest for all data stores
- [ ] Encryption in transit (TLS/SSL)
- [ ] IAM policies follow least privilege
- [ ] Logging enabled for all resources
- [ ] MFA required for privileged access
- [ ] Regular security scanning in CI/CD

### HIPAA Compliance

- [ ] PHI encrypted at rest and in transit
- [ ] Access logs enabled
- [ ] Dedicated VPC with private subnets
- [ ] Regular backup and retention policies
- [ ] Audit trail for all infrastructure changes
- [ ] BAA with Google Cloud

---

## Resources

- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Checkov Documentation](https://www.checkov.io/)
- [Prowler Documentation](https://github.com/prowler-cloud/prowler)
- [Google Cloud Security Best Practices](https://cloud.google.com/security/best-practices)
- [CIS Google Cloud Computing Platform Benchmark](https://www.cisecurity.org/benchmark/google_cloud_computing_platform)
- [VPC Service Controls](https://cloud.google.com/vpc-service-controls)

---

**Back to:** [Main Skill File](../SKILL.md)
