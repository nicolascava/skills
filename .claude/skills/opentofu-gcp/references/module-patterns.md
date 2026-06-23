# Module Development Patterns

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** Best practices for OpenTofu module development for GCP

This document provides detailed guidance on creating reusable, maintainable OpenTofu modules for Google Cloud Platform.

---

## Table of Contents

1. [Module Hierarchy](#module-hierarchy)
2. [Architecture Principles](#architecture-principles)
3. [Module Structure](#module-structure)
4. [Variable Best Practices](#variable-best-practices)
5. [Output Best Practices](#output-best-practices)
6. [Common Patterns](#common-patterns)
7. [Anti-patterns to Avoid](#anti-patterns-to-avoid)
8. [GCP Module Examples](#gcp-module-examples)

---

## Module Hierarchy

### Module Type Classification

OpenTofu modules can be organized into three distinct types, each serving a specific purpose:

| Type | When to Use | Scope | Example |
|------|-------------|-------|---------|
| **Resource Module** | Single logical group of connected resources | Tightly coupled resources | VPC + subnets, Firewall + rules |
| **Infrastructure Module** | Collection of resource modules for a purpose | Multiple resource modules in one region | Complete networking stack |
| **Composition** | Complete infrastructure | Spans multiple regions/projects | Production environment |

**Hierarchy:** Resource → Resource Module → Infrastructure Module → Composition

### Resource Module

**Characteristics:**
- Smallest building block
- Single logical group of resources
- Highly reusable across projects
- Minimal external dependencies
- Clear, focused purpose

**Examples:**
```
modules/
├── vpc/                    # Resource module
│   ├── main.tf            # VPC + subnets + routes
│   ├── variables.tf
│   └── outputs.tf
├── firewall/               # Resource module
│   ├── main.tf            # Firewall rules
│   ├── variables.tf
│   └── outputs.tf
└── cloudsql/               # Resource module
    ├── main.tf            # Cloud SQL instance
    ├── variables.tf
    └── outputs.tf
```

### Infrastructure Module

**Characteristics:**
- Combines multiple resource modules
- Purpose-specific (e.g., "web application infrastructure")
- May span multiple services
- Region-specific
- Moderate reusability

**Examples:**
```
modules/
└── web-application/        # Infrastructure module
    ├── main.tf            # Orchestrates multiple resource modules
    ├── variables.tf
    ├── outputs.tf
    └── README.md

# main.tf contents:
module "vpc" {
  source = "../vpc"
}

module "gke" {
  source     = "../gke"
  network_id = module.vpc.network_id
}

module "cloudsql" {
  source     = "../cloudsql"
  network_id = module.vpc.network_id
}
```

### Composition

**Characteristics:**
- Highest level of abstraction
- Complete environment or application
- Combines infrastructure modules
- Environment-specific (dev, staging, prod)
- Not reusable (environment-specific values)

**Examples:**
```
environments/
├── prod/                   # Composition
│   ├── main.tf            # Complete production environment
│   ├── backend.tf         # Remote state configuration
│   ├── terraform.tfvars   # Production-specific values
│   └── variables.tf
├── staging/                # Composition
│   └── ...
└── dev/                    # Composition
    └── ...
```

---

## Architecture Principles

### 1. Smaller Scopes = Better Performance + Reduced Blast Radius

```hcl
# ❌ BAD - One massive composition with everything
environments/prod/
  main.tf  # 2000 lines, manages everything

# ✅ GOOD - Separated by concern
environments/prod/
  networking/     # VPC, subnets, Cloud NAT
  compute/        # GCE, Instance Groups
  data/           # Cloud SQL, Memorystore
  gke/            # GKE cluster
  iam/            # IAM roles, service accounts
```

### 2. Always Use Remote State with GCS

```hcl
# ✅ GOOD - Remote state with GCS
tofu {
  backend "gcs" {
    bucket = "my-project-tofu-state"
    prefix = "prod/networking"
  }
}
```

### 3. Use terraform_remote_state as Glue

```hcl
# environments/prod/networking/outputs.tf
output "network_id" {
  description = "ID of the production VPC"
  value       = google_compute_network.this.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = [for s in google_compute_subnetwork.private : s.id]
}

# environments/prod/gke/main.tf
data "terraform_remote_state" "networking" {
  backend = "gcs"
  config = {
    bucket = "my-project-tofu-state"
    prefix = "prod/networking"
  }
}

module "gke" {
  source = "../../modules/gke"

  network_id = data.terraform_remote_state.networking.outputs.network_id
  subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
}
```

### 4. Keep Resource Modules Simple

```hcl
# ❌ BAD - Hardcoded values in resource module
resource "google_compute_instance" "web" {
  machine_type = "e2-medium"              # Hardcoded
  zone         = "us-central1-a"          # Hardcoded
  project      = "my-project"             # Hardcoded
}

# ✅ GOOD - Parameterized resource module
resource "google_compute_instance" "web" {
  machine_type = var.machine_type
  zone         = var.zone
  project      = var.project_id

  labels = var.labels
}
```

---

## Module Structure

### Standard Layout

```
my-module/
├── README.md                # Usage documentation
├── LICENSE                  # Apache 2.0 (for public modules)
├── .pre-commit-config.yaml  # Pre-commit hooks
├── main.tf                  # Primary resources
├── variables.tf             # Input variables with descriptions
├── outputs.tf               # Output values
├── versions.tf              # Provider version constraints
├── examples/
│   ├── simple/              # Minimal working example
│   └── complete/            # Full-featured example
└── tests/                   # Test files
    └── module_test.tftest.hcl
```

### Module Naming Convention

```
tofu-google-<NAME>

Examples:
tofu-google-vpc
tofu-google-gke
tofu-google-cloudsql
tofu-google-pubsub
```

---

## Variable Best Practices

### Complete Example

```hcl
variable "machine_type" {
  description = "GCE machine type for the application server"
  type        = string
  default     = "e2-micro"

  validation {
    condition     = can(regex("^(e2|n2|n2d|c2|c2d|m1|m2)-", var.machine_type))
    error_message = "Machine type must be a valid GCE machine type."
  }
}

variable "labels" {
  description = "Labels to apply to all resources"
  type        = map(string)
  default     = {}
}

variable "enable_monitoring" {
  description = "Enable Cloud Monitoring for resources"
  type        = bool
  default     = true
}
```

### Key Principles

- ✅ **Always include `description`** - Helps users understand the variable
- ✅ **Use explicit `type` constraints** - Catches errors early
- ✅ **Provide sensible `default` values** - Where appropriate
- ✅ **Add `validation` blocks** - For complex constraints
- ✅ **Use `sensitive = true`** - For secrets

### Variable Naming

```hcl
# ✅ Good: Context-specific
var.vpc_cidr_range          # Not just "cidr"
var.database_tier           # Not just "tier"
var.gke_node_count          # Not just "count"

# ❌ Bad: Generic names
var.name
var.type
var.value
```

---

## Output Best Practices

### Complete Example

```hcl
output "instance_id" {
  description = "ID of the created GCE instance"
  value       = google_compute_instance.this.id
}

output "instance_self_link" {
  description = "Self-link of the created GCE instance"
  value       = google_compute_instance.this.self_link
}

output "private_ip" {
  description = "Private IP address of the instance"
  value       = google_compute_instance.this.network_interface[0].network_ip
}

output "connection_info" {
  description = "Connection information for the instance"
  value = {
    id         = google_compute_instance.this.id
    private_ip = google_compute_instance.this.network_interface[0].network_ip
    zone       = google_compute_instance.this.zone
  }
}
```

### Key Principles

- ✅ **Always include `description`** - Explain what the output is for
- ✅ **Mark sensitive outputs** - Use `sensitive = true`
- ✅ **Return objects for related values** - Groups logically related data
- ✅ **Document intended use** - What should consumers do with this?

---

## Common Patterns

### ✅ DO: Use `for_each` for Resources

```hcl
# Good: Maintain stable resource addresses
resource "google_compute_subnetwork" "private" {
  for_each = toset(var.zones)

  name          = "private-${each.key}"
  ip_cidr_range = cidrsubnet(var.vpc_cidr, 8, index(var.zones, each.key))
  region        = var.region
  network       = google_compute_network.this.id
}
```

### ❌ DON'T: Use `count` When Order Matters

```hcl
# Bad: Removing middle item reshuffles all subsequent resources
resource "google_compute_subnetwork" "private" {
  count = length(var.zones)

  name          = "private-${var.zones[count.index]}"
  ip_cidr_range = cidrsubnet(var.vpc_cidr, 8, count.index)
}
```

### ✅ DO: Use Locals for Computed Values

```hcl
locals {
  required_labels = merge(
    var.labels,
    {
      environment = var.environment
      managed-by  = "opentofu"
    }
  )

  instance_name = "${var.project_name}-${var.environment}-instance"
}

resource "google_compute_instance" "app" {
  name   = local.instance_name
  labels = local.required_labels
}
```

### ✅ DO: Version Your Modules

```hcl
# In consuming code
module "vpc" {
  source  = "github.com/myorg/tofu-google-vpc?ref=v1.2.0"

  # module inputs...
}
```

---

## Anti-patterns to Avoid

### ❌ DON'T: Hard-code Environment-Specific Values

```hcl
# Bad: Module is locked to production
resource "google_compute_instance" "app" {
  machine_type = "e2-medium"  # Should be variable
  labels = {
    environment = "production" # Should be variable
  }
}
```

### ❌ DON'T: Create God Modules

```hcl
# Bad: One module does everything
module "everything" {
  source = "./modules/app-infrastructure"
  # Creates VPC, GCE, Cloud SQL, GCS, IAM, everything
}

# Good: Break into focused modules
module "networking" {
  source = "./modules/vpc"
}

module "compute" {
  source     = "./modules/gce"
  network_id = module.networking.network_id
}

module "database" {
  source     = "./modules/cloudsql"
  network_id = module.networking.network_id
}
```

### ❌ DON'T: Use `count` for Different Environments

```hcl
# Bad: All environments in one module
resource "google_compute_instance" "app" {
  for_each = toset(["dev", "staging", "prod"])

  machine_type = each.key == "prod" ? "e2-medium" : "e2-micro"
}

# Good: Use separate compositions
environments/
  dev/
    main.tf
  staging/
    main.tf
  prod/
    main.tf
```

---

## GCP Module Examples

### VPC Module

```hcl
# modules/vpc/main.tf
resource "google_compute_network" "this" {
  name                    = var.network_name
  project                 = var.project_id
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "private" {
  for_each = var.subnets

  name          = each.key
  ip_cidr_range = each.value.cidr
  region        = each.value.region
  network       = google_compute_network.this.id
  project       = var.project_id

  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = each.value.pods_cidr
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = each.value.services_cidr
  }
}
```

### GKE Module

```hcl
# modules/gke/main.tf
resource "google_container_cluster" "primary" {
  name     = var.cluster_name
  location = var.region
  project  = var.project_id

  network    = var.network_id
  subnetwork = var.subnet_id

  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1

  # Private cluster
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_cidr
  }

  # IP allocation
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  resource_labels = var.labels
}

resource "google_container_node_pool" "primary" {
  name       = "primary"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  project    = var.project_id

  node_count = var.node_count

  node_config {
    machine_type = var.machine_type
    disk_size_gb = var.disk_size_gb

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    labels = var.labels
  }
}
```

### Pub/Sub Module

```hcl
# modules/pubsub/main.tf
resource "google_pubsub_topic" "this" {
  name    = var.topic_name
  project = var.project_id

  message_retention_duration = var.retention_duration

  labels = var.labels
}

resource "google_pubsub_subscription" "this" {
  for_each = var.subscriptions

  name    = each.key
  topic   = google_pubsub_topic.this.name
  project = var.project_id

  message_retention_duration = each.value.retention_duration
  ack_deadline_seconds       = each.value.ack_deadline

  expiration_policy {
    ttl = each.value.expiration_ttl
  }

  labels = var.labels
}
```

---

## Artifact Registry Patterns

### Docker Repository

```hcl
# modules/artifact-registry/main.tf
resource "google_artifact_registry_repository" "docker" {
  location      = var.region
  repository_id = "${var.project_name}-docker"
  format        = "DOCKER"
  description   = "Docker repository for ${var.project_name}"

  docker_config {
    immutable_tags = var.immutable_tags
  }

  labels = var.labels
}

# IAM binding for read access
resource "google_artifact_registry_repository_iam_member" "reader" {
  for_each = toset(var.reader_members)

  location   = google_artifact_registry_repository.docker.location
  repository = google_artifact_registry_repository.docker.name
  role       = "roles/artifactregistry.reader"
  member     = each.value
}

# IAM binding for write access
resource "google_artifact_registry_repository_iam_member" "writer" {
  for_each = toset(var.writer_members)

  location   = google_artifact_registry_repository.docker.location
  repository = google_artifact_registry_repository.docker.name
  role       = "roles/artifactregistry.writer"
  member     = each.value
}
```

### Multi-Format Repositories

```hcl
variable "repository_formats" {
  description = "Repository formats to create"
  type = map(object({
    format      = string
    description = string
  }))
  default = {
    docker = {
      format      = "DOCKER"
      description = "Docker container images"
    }
    npm = {
      format      = "NPM"
      description = "NPM packages"
    }
    python = {
      format      = "PYTHON"
      description = "Python packages"
    }
    maven = {
      format      = "MAVEN"
      description = "Maven artifacts"
    }
  }
}

resource "google_artifact_registry_repository" "repos" {
  for_each = var.repository_formats

  location      = var.region
  repository_id = "${var.project_name}-${each.key}"
  format        = each.value.format
  description   = each.value.description

  labels = var.labels
}
```

### Remote Repository (Proxy)

```hcl
# Proxy for Docker Hub to reduce rate limits and improve performance
resource "google_artifact_registry_repository" "docker_hub_proxy" {
  location      = var.region
  repository_id = "${var.project_name}-docker-hub-proxy"
  format        = "DOCKER"
  mode          = "REMOTE_REPOSITORY"
  description   = "Docker Hub proxy repository"

  remote_repository_config {
    description = "Docker Hub proxy"
    docker_repository {
      public_repository = "DOCKER_HUB"
    }
  }

  labels = var.labels
}

# Proxy for npm registry
resource "google_artifact_registry_repository" "npm_proxy" {
  location      = var.region
  repository_id = "${var.project_name}-npm-proxy"
  format        = "NPM"
  mode          = "REMOTE_REPOSITORY"
  description   = "npm registry proxy"

  remote_repository_config {
    description = "npm registry proxy"
    npm_repository {
      public_repository = "NPMJS"
    }
  }

  labels = var.labels
}
```

### Cloud Build Integration

```hcl
# Service account for Cloud Build with Artifact Registry access
resource "google_service_account" "cloud_build" {
  account_id   = "${var.project_name}-cloud-build"
  display_name = "Cloud Build Service Account"
}

# Grant Cloud Build access to push images
resource "google_artifact_registry_repository_iam_member" "cloud_build_writer" {
  location   = google_artifact_registry_repository.docker.location
  repository = google_artifact_registry_repository.docker.name
  role       = "roles/artifactregistry.writer"
  member     = "serviceAccount:${google_service_account.cloud_build.email}"
}

# Grant Cloud Build access to pull base images from proxy
resource "google_artifact_registry_repository_iam_member" "cloud_build_reader" {
  location   = google_artifact_registry_repository.docker_hub_proxy.location
  repository = google_artifact_registry_repository.docker_hub_proxy.name
  role       = "roles/artifactregistry.reader"
  member     = "serviceAccount:${google_service_account.cloud_build.email}"
}
```

### GKE Workload Identity Integration

```hcl
# Service account for GKE workloads to pull images
resource "google_service_account" "gke_workload" {
  account_id   = "${var.project_name}-gke-workload"
  display_name = "GKE Workload Service Account"
}

# Allow GKE workloads to pull from Artifact Registry
resource "google_artifact_registry_repository_iam_member" "gke_reader" {
  location   = google_artifact_registry_repository.docker.location
  repository = google_artifact_registry_repository.docker.name
  role       = "roles/artifactregistry.reader"
  member     = "serviceAccount:${google_service_account.gke_workload.email}"
}

# Workload Identity binding
resource "google_service_account_iam_member" "gke_workload_identity" {
  service_account_id = google_service_account.gke_workload.id
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[${var.namespace}/${var.k8s_service_account}]"
}
```

### Outputs

```hcl
output "docker_repository_url" {
  description = "URL for Docker repository"
  value       = "${google_artifact_registry_repository.docker.location}-docker.pkg.dev/${var.project_id}/${google_artifact_registry_repository.docker.repository_id}"
}

output "repository_urls" {
  description = "Map of repository URLs by format"
  value = {
    for key, repo in google_artifact_registry_repository.repos :
    key => "${repo.location}-${lower(repo.format)}.pkg.dev/${var.project_id}/${repo.repository_id}"
  }
}
```

---

## Pre-commit Configuration

```yaml
# .pre-commit-config.yaml
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
```

---

## .gitignore Template

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

# Plan files
*.tfplan
*.tfplan.json

# Secrets
.env
secrets/
*.pem
*.key
credentials.json
```

---

**Back to:** [Main Skill File](../SKILL.md)
