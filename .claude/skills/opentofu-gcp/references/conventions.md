# GCP Conventions & Standards

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** Required GCP labels, naming conventions, and organization templates

This document provides standardized conventions for OpenTofu/GCP projects to ensure consistency, cost tracking, and compliance.

---

## Table of Contents

1. [Required Labels](#required-labels)
2. [Naming Conventions](#naming-conventions)
3. [Resource Naming Patterns](#resource-naming-patterns)
4. [Organization Templates](#organization-templates)
5. [Environment Configuration](#environment-configuration)

---

## Required Labels

### Standard Label Template

All GCP resources that support labels MUST include these labels:

```hcl
locals {
  required_labels = {
    environment = var.environment      # dev, staging, prod
    project     = var.project_name     # application/service name
    managed-by  = "opentofu"           # always "opentofu"
    owner       = var.owner            # team or individual responsible
    cost-center = var.cost_center      # billing allocation code
  }
}
```

### Label Variables

```hcl
variable "environment" {
  description = "Environment name for resource labeling"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "project_name" {
  description = "Application or service name for labeling"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,62}$", var.project_name))
    error_message = "Project name must be lowercase, start with a letter, and be 3-63 characters."
  }
}

variable "owner" {
  description = "Team or individual responsible for these resources"
  type        = string
}

variable "cost_center" {
  description = "Cost center code for billing allocation"
  type        = string
}
```

### Applying Labels to Resources

```hcl
# Compute Instance
resource "google_compute_instance" "web-server" {
  name         = "web-server-${var.environment}"
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network    = google_compute_network.this.id
    subnetwork = google_compute_subnetwork.this.id
  }

  labels = local.required_labels
}

# GKE Cluster
resource "google_container_cluster" "primary" {
  name     = "primary-${var.environment}"
  location = var.region

  initial_node_count       = 1
  remove_default_node_pool = true

  resource_labels = local.required_labels
}

# Cloud SQL
resource "google_sql_database_instance" "main" {
  name             = "main-${var.environment}"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier = var.database_tier

    user_labels = local.required_labels
  }
}

# GCS Bucket
resource "google_storage_bucket" "data" {
  name     = "${var.project_name}-data-${var.environment}"
  location = var.region

  labels = local.required_labels
}

# Pub/Sub Topic
resource "google_pubsub_topic" "events" {
  name = "events-${var.environment}"

  labels = local.required_labels
}
```

### Optional Labels

Add these labels when applicable:

```hcl
locals {
  optional_labels = {
    # Lifecycle management
    ttl         = "24h"                 # For test resources
    created-at  = formatdate("YYYY-MM-DD", timestamp())

    # Compliance
    data-class  = "confidential"        # public, internal, confidential, restricted
    compliance  = "pci-dss"             # hipaa, pci-dss, soc2, etc.

    # Operations
    backup      = "daily"               # none, daily, weekly
    monitoring  = "enhanced"            # basic, enhanced

    # Team
    team        = "platform"            # Team name
    slack       = "platform-alerts"     # Alert channel
  }

  all_labels = merge(local.required_labels, local.optional_labels)
}
```

---

## Naming Conventions

### General Rules

| Element | Convention | Example |
|---------|------------|---------|
| GCP resource names | lowercase with hyphens | `web-server-prod` |
| HCL identifiers | lowercase with underscores | `google_compute_instance.web_server` |
| Project IDs | lowercase, hyphens, globally unique | `acme-webapp-prod-a1b2c3` |
| Service accounts | lowercase, hyphens | `webapp-sa@project.iam.gserviceaccount.com` |

### Naming Pattern

```
{service}-{component}-{environment}[-{region}][-{instance}]

Examples:
- webapp-api-prod
- webapp-db-staging
- webapp-worker-prod-us-central1
- webapp-cache-prod-01
```

### HCL Resource Naming

```hcl
# ✅ GOOD - Descriptive, contextual
resource "google_compute_instance" "web_server" { }
resource "google_compute_instance" "api_server" { }

# ✅ GOOD - "this" for singletons
resource "google_compute_network" "this" { }

# ❌ BAD - Generic names
resource "google_compute_instance" "main" { }
resource "google_compute_instance" "instance" { }
```

### Naming Best Practices

**Avoid redundancy - don't repeat resource type in name:**
```hcl
# ✅ GOOD
resource "google_compute_network" "main" {}

# ❌ BAD - Redundant "network" in name
resource "google_compute_network" "main_network" {}
```

**Use singular nouns for resource names:**
```hcl
# ✅ GOOD - Singular noun
resource "google_compute_instance" "web_server" {}

# ❌ BAD - Plural noun
resource "google_compute_instance" "web_servers" {}
```

**Avoid double negatives in variable names:**
```hcl
# ✅ GOOD - Positive naming
variable "encryption_enabled" {
  description = "Enable encryption for the resource"
  type        = bool
  default     = true
}

# ❌ BAD - Double negative possible
variable "encryption_disabled" {
  description = "Disable encryption for the resource"
  type        = bool
  default     = false  # !encryption_disabled is confusing
}
```

---

## Resource Naming Patterns

### Compute Resources

```hcl
# Instance naming pattern: {role}-{env}[-{zone-suffix}]
resource "google_compute_instance" "web_server" {
  name = "web-server-${var.environment}"
  # In multi-zone: "web-server-prod-a", "web-server-prod-b"
}

# Instance group: {role}-{env}-{region}
resource "google_compute_region_instance_group_manager" "web" {
  name = "web-${var.environment}-${var.region}"
}

# Instance template: {role}-{env}-{version}
resource "google_compute_instance_template" "web" {
  name_prefix = "web-${var.environment}-"
}
```

### Networking Resources

```hcl
# VPC: {project}-{env}-vpc
resource "google_compute_network" "this" {
  name = "${var.project_name}-${var.environment}-vpc"
}

# Subnet: {project}-{env}-{purpose}-{region}
resource "google_compute_subnetwork" "private" {
  name = "${var.project_name}-${var.environment}-private-${var.region}"
}

# Firewall: {vpc}-{action}-{protocol}-{purpose}
resource "google_compute_firewall" "allow_http" {
  name    = "${var.project_name}-${var.environment}-allow-tcp-http"
  network = google_compute_network.this.name
}

# Cloud NAT: {vpc}-nat-{region}
resource "google_compute_router_nat" "this" {
  name   = "${var.project_name}-${var.environment}-nat-${var.region}"
  router = google_compute_router.this.name
}
```

### Data Resources

```hcl
# Cloud SQL: {project}-{env}-{engine}
resource "google_sql_database_instance" "main" {
  name = "${var.project_name}-${var.environment}-postgres"
}

# GCS Bucket: {project}-{purpose}-{env}
resource "google_storage_bucket" "data" {
  name = "${var.project_name}-data-${var.environment}"
}

# Memorystore: {project}-{env}-redis
resource "google_redis_instance" "cache" {
  name = "${var.project_name}-${var.environment}-redis"
}
```

### GKE Resources

```hcl
# Cluster: {project}-{env}-{purpose}
resource "google_container_cluster" "primary" {
  name = "${var.project_name}-${var.environment}-primary"
}

# Node pool: {cluster}-{purpose}
resource "google_container_node_pool" "general" {
  name    = "${var.project_name}-${var.environment}-general"
  cluster = google_container_cluster.primary.name
}
```

### Pub/Sub Resources

```hcl
# Topic: {project}-{event-type}-{env}
resource "google_pubsub_topic" "orders" {
  name = "${var.project_name}-orders-${var.environment}"
}

# Subscription: {topic}-{consumer}-sub
resource "google_pubsub_subscription" "orders_processor" {
  name  = "${var.project_name}-orders-processor-${var.environment}-sub"
  topic = google_pubsub_topic.orders.name
}
```

### IAM Resources

```hcl
# Service Account: {purpose}-sa
resource "google_service_account" "webapp" {
  account_id   = "${var.project_name}-webapp-sa"
  display_name = "WebApp Service Account"
}

# Custom Role: {org}.{purpose}
resource "google_project_iam_custom_role" "log_reader" {
  role_id = "${replace(var.project_name, "-", "_")}_log_reader"
  title   = "Log Reader"
}
```

---

## Organization Templates

### Project Structure

```
{org}-{project}/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── gke/
│   ├── cloudsql/
│   └── pubsub/
├── examples/
│   ├── minimal/
│   └── complete/
├── scripts/
│   └── qa_runner.py
├── .pre-commit-config.yaml
├── .gitignore
└── README.md
```

### Standard versions.tf

```hcl
tofu {
  required_version = "~> 1.11"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.0"
    }
  }
}

provider "google" {
  project = var.gcp_project_id
  region  = var.region
}

provider "google-beta" {
  project = var.gcp_project_id
  region  = var.region
}
```

### Standard backend.tf (with encryption)

```hcl
tofu {
  encryption {
    method "gcp_kms" "state_key" {
      kms_encryption_key = "projects/${var.gcp_project_id}/locations/global/keyRings/tofu-state/cryptoKeys/state-key"
    }
    state {
      method = method.gcp_kms.state_key
    }
    plan {
      method = method.gcp_kms.state_key
    }
  }

  backend "gcs" {
    bucket = "${var.project_name}-tofu-state"
    prefix = "${var.environment}"
  }
}
```

### Standard .gitignore

```gitignore
# OpenTofu
**/.terraform/*
.terraform.lock.hcl
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfvars
*.tfvars.json
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc

# OpenTofu plan files
*.tfplan
*.tfplan.json

# Secrets
.env
.env.*
secrets/
*.secret
*.pem
*.key
credentials.json
service-account.json

# IDE
.idea/
.vscode/
*.swp
*.swo
*~
.DS_Store
```

### Standard .editorconfig

Include an `.editorconfig` file to maintain consistent formatting across editors:

```editorconfig
[*]
indent_style = space
indent_size = 2
trim_trailing_whitespace = true
insert_final_newline = true

[*.{tf,tfvars}]
indent_style = space
indent_size = 2

[Makefile]
indent_style = tab
```

### Standard .pre-commit-config.yaml

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_docs
        args:
          - --args=--config=.terraform-docs.yml

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
```

---

## Environment Configuration

### Variable Defaults by Environment

```hcl
locals {
  environment_config = {
    dev = {
      machine_type     = "e2-micro"
      database_tier    = "db-f1-micro"
      min_node_count   = 1
      max_node_count   = 3
      deletion_protection = false
    }
    staging = {
      machine_type     = "e2-small"
      database_tier    = "db-custom-1-3840"
      min_node_count   = 2
      max_node_count   = 5
      deletion_protection = false
    }
    prod = {
      machine_type     = "e2-medium"
      database_tier    = "db-custom-2-7680"
      min_node_count   = 3
      max_node_count   = 10
      deletion_protection = true
    }
  }

  config = local.environment_config[var.environment]
}

# Usage
resource "google_compute_instance" "app" {
  machine_type = local.config.machine_type
  # ...
}

resource "google_sql_database_instance" "main" {
  deletion_protection = local.config.deletion_protection

  settings {
    tier = local.config.database_tier
  }
}
```

### Region Configuration

```hcl
variable "region" {
  description = "GCP region for resources"
  type        = string
  default     = "us-central1"

  validation {
    condition = contains([
      "us-central1", "us-east1", "us-east4", "us-west1", "us-west2",
      "europe-west1", "europe-west2", "europe-west3",
      "asia-east1", "asia-southeast1"
    ], var.region)
    error_message = "Region must be an approved GCP region."
  }
}

locals {
  zone_suffixes = ["a", "b", "c"]
  zones         = [for suffix in local.zone_suffixes : "${var.region}-${suffix}"]
}
```

### Network Configuration

```hcl
locals {
  network_config = {
    dev = {
      vpc_cidr           = "10.0.0.0/16"
      pods_cidr          = "10.100.0.0/14"
      services_cidr      = "10.104.0.0/20"
      master_cidr        = "172.16.0.0/28"
    }
    staging = {
      vpc_cidr           = "10.10.0.0/16"
      pods_cidr          = "10.110.0.0/14"
      services_cidr      = "10.114.0.0/20"
      master_cidr        = "172.16.0.16/28"
    }
    prod = {
      vpc_cidr           = "10.20.0.0/16"
      pods_cidr          = "10.120.0.0/14"
      services_cidr      = "10.124.0.0/20"
      master_cidr        = "172.16.0.32/28"
    }
  }
}
```

---

## Customization Template

Organizations should customize these conventions. Create a `conventions.hcl` file:

```hcl
# Organization-specific conventions
# Copy and modify as needed

locals {
  # Organization prefix for global resources
  org_prefix = "acme"

  # Standard labels - customize as needed
  required_labels = {
    environment  = var.environment
    project      = var.project_name
    managed-by   = "opentofu"
    owner        = var.owner
    cost-center  = var.cost_center

    # Add organization-specific labels
    org          = local.org_prefix
    department   = var.department
  }

  # Resource naming function
  name = {
    # {org}-{project}-{purpose}-{env}
    full   = "${local.org_prefix}-${var.project_name}-${var.purpose}-${var.environment}"
    # {project}-{purpose}-{env}
    medium = "${var.project_name}-${var.purpose}-${var.environment}"
    # {purpose}-{env}
    short  = "${var.purpose}-${var.environment}"
  }
}
```

---

**Back to:** [Main Skill File](../SKILL.md)
