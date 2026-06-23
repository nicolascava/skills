# Code Patterns & Structure

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** Comprehensive patterns for OpenTofu code structure and modern features for GCP

This document provides detailed code patterns, structure guidelines, and modern OpenTofu features for Google Cloud Platform.

---

## Table of Contents

1. [Block Ordering & Structure](#block-ordering--structure)
2. [Comment Styling](#comment-styling)
3. [Count vs For_Each Deep Dive](#count-vs-for_each-deep-dive)
4. [Modern OpenTofu Features](#modern-opentofu-features)
5. [Version Management](#version-management)
6. [Refactoring Patterns](#refactoring-patterns)
7. [Locals for Dependency Management](#locals-for-dependency-management)
8. [Orchestration Approaches](#orchestration-approaches)

---

## Block Ordering & Structure

### Resource Block Structure

**Strict argument ordering:**

1. `count` or `for_each` or `enabled` FIRST (blank line after)
2. Other arguments (alphabetical or logical grouping)
3. `labels` as last real argument (GCP uses labels, not tags)
4. `depends_on` after labels (if needed)
5. `lifecycle` at the very end (if needed)

```hcl
# ✅ GOOD - Correct ordering
resource "google_compute_router_nat" "this" {
  enabled = var.create_nat

  name                               = "${var.name}-nat"
  router                             = google_compute_router.this.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  depends_on = [google_compute_router.this]

  lifecycle {
    create_before_destroy = true
  }
}

# ❌ BAD - Wrong ordering
resource "google_compute_router_nat" "this" {
  nat_ip_allocate_option = "AUTO_ONLY"

  enabled = var.create_nat  # Should be first

  name   = "${var.name}-nat"
  router = google_compute_router.this.name

  lifecycle {
    create_before_destroy = true
  }

  depends_on = [google_compute_router.this]  # Should be after labels
}
```

### Variable Definition Structure

**Variable block ordering:**

1. `description` (ALWAYS required)
2. `type`
3. `default`
4. `sensitive` (when setting to true)
5. `nullable` (when setting to false)
6. `validation`

```hcl
# ✅ GOOD - Correct ordering and structure
variable "environment" {
  description = "Environment name for resource labeling"
  type        = string
  default     = "dev"
  nullable    = false

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}
```

### Variable Type Preferences

- Prefer **simple types** (`string`, `number`, `list()`, `map()`) over `object()` unless strict validation needed
- Use `optional()` for optional object attributes (OpenTofu 1.3+)
- Use `any` to disable validation at certain depths or support multiple types

**Modern variable patterns (OpenTofu 1.3+):**

```hcl
# ✅ GOOD - Using optional() for object attributes
variable "database_config" {
  description = "Database configuration with optional parameters"
  type = object({
    name               = string
    database_version   = string
    tier               = string
    backup_retention   = optional(number, 7)      # Default: 7
    high_availability  = optional(bool, true)     # Default: true
    labels             = optional(map(string), {}) # Default: {}
  })
}

# Usage - only required fields needed
database_config = {
  name             = "mydb"
  database_version = "POSTGRES_15"
  tier             = "db-f1-micro"
  # Optional fields use defaults
}
```

### Output Structure

**Pattern:** `{name}_{type}_{attribute}`

```hcl
# ✅ GOOD
output "network_id" {
  description = "The ID of the VPC network"
  value       = try(google_compute_network.this.id, "")
}

output "private_subnet_ids" {  # Plural for list
  description = "List of private subnet IDs"
  value       = [for s in google_compute_subnetwork.private : s.id]
}

# ❌ BAD
output "this_network_id" {  # Don't prefix with "this_"
  value = google_compute_network.this.id
}

output "subnet_id" {  # Should be plural "subnet_ids"
  value = [for s in google_compute_subnetwork.private : s.id]  # Returns list
}
```

---

## Comment Styling

**Use `#` for all comments:**
```hcl
# ✅ GOOD - Hash comments
# This creates a VPC network
resource "google_compute_network" "this" {
  name = var.network_name
}

# ❌ BAD - Avoid // or /* */ style comments
// This creates a VPC network
resource "google_compute_network" "this" {
  name = var.network_name
}
```

**Use section headers for organization in large files:**
```hcl
# --------------------------------------------------
# VPC Network Configuration
# --------------------------------------------------

resource "google_compute_network" "this" {
  name                    = var.network_name
  auto_create_subnetworks = false
}

# --------------------------------------------------
# Subnet Configuration
# --------------------------------------------------

resource "google_compute_subnetwork" "private" {
  name          = "private-subnet"
  ip_cidr_range = var.subnet_cidr
  region        = var.region
  network       = google_compute_network.this.id
}
```

---

## Count vs For_Each Deep Dive

### When to use count

✓ **Simple numeric replication:**
```hcl
resource "google_compute_instance" "web" {
  count = 3

  name         = "web-${count.index}"
  machine_type = var.machine_type
  zone         = element(var.zones, count.index)
}
```

✓ **Boolean conditions (create or don't):**
```hcl
# ✅ BEST - enabled meta-argument (OpenTofu 1.11+)
resource "google_compute_router_nat" "this" {
  enabled = var.create_nat
}

# ✅ GOOD - Boolean condition (pre-1.11)
resource "google_compute_router_nat" "this" {
  count = var.create_nat ? 1 : 0
}
```

### When to use for_each

✓ **Reference resources by key:**
```hcl
resource "google_compute_subnetwork" "private" {
  for_each = toset(var.zones)

  name          = "private-${each.key}"
  ip_cidr_range = cidrsubnet(var.vpc_cidr, 4, index(var.zones, each.key))
  region        = var.region
  network       = google_compute_network.this.id
}

# Reference by key: google_compute_subnetwork.private["us-central1-a"]
```

✓ **Items may be added/removed from middle:**
```hcl
# ❌ BAD with count - removing middle item recreates all subsequent resources
resource "google_compute_subnetwork" "private" {
  count = length(var.zones)

  name = "private-${var.zones[count.index]}"
  # If var.zones[1] removed, all resources after recreated!
}

# ✅ GOOD with for_each - removal only affects that one resource
resource "google_compute_subnetwork" "private" {
  for_each = toset(var.zones)

  name = "private-${each.key}"
  # Removing one zone only destroys that subnet
}
```

✓ **Creating multiple named resources:**
```hcl
variable "environments" {
  default = {
    dev = {
      machine_type   = "e2-micro"
      instance_count = 1
    }
    prod = {
      machine_type   = "e2-medium"
      instance_count = 3
    }
  }
}

resource "google_compute_instance" "app" {
  for_each = var.environments

  machine_type = each.value.machine_type

  labels = {
    environment = each.key  # "dev" or "prod"
  }
}
```

### Count to For_Each Migration

**When to migrate:** When you need stable resource addressing or items might be added/removed from middle of list.

**Migration steps:**

1. Add `for_each` to resource
2. Use `moved` blocks to preserve existing resources
3. Remove `count` after verifying with `tofu plan`

**Complete example:**

```hcl
# Before (using count)
resource "google_compute_subnetwork" "private" {
  count = length(var.zones)

  name          = "private-${var.zones[count.index]}"
  ip_cidr_range = cidrsubnet(var.vpc_cidr, 8, count.index)
  region        = var.region
  network       = google_compute_network.this.id
}

# After (using for_each)
resource "google_compute_subnetwork" "private" {
  for_each = toset(var.zones)

  name          = "private-${each.key}"
  ip_cidr_range = cidrsubnet(var.vpc_cidr, 8, index(var.zones, each.key))
  region        = var.region
  network       = google_compute_network.this.id
}

# Migration blocks (prevents resource recreation)
moved {
  from = google_compute_subnetwork.private[0]
  to   = google_compute_subnetwork.private["us-central1-a"]
}

moved {
  from = google_compute_subnetwork.private[1]
  to   = google_compute_subnetwork.private["us-central1-b"]
}

moved {
  from = google_compute_subnetwork.private[2]
  to   = google_compute_subnetwork.private["us-central1-c"]
}
```

---

## Modern OpenTofu Features

### try() Function (1.0+)

**Use try() instead of element(concat()):**

```hcl
# ✅ GOOD - Modern try() function
output "network_id" {
  description = "The ID of the VPC network"
  value       = try(google_compute_network.this[0].id, "")
}

output "first_subnet_id" {
  description = "ID of first subnet with multiple fallbacks"
  value       = try(
    google_compute_subnetwork.public[0].id,
    google_compute_subnetwork.private[0].id,
    ""
  )
}

# ❌ BAD - Legacy pattern
output "network_id" {
  value = element(concat(google_compute_network.this.*.id, [""]), 0)
}
```

### nullable = false (1.1+)

**Set nullable = false for non-null variables:**

```hcl
# ✅ GOOD (OpenTofu 1.1+)
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  nullable    = false  # Passing null uses default, not null
  default     = "10.0.0.0/16"
}
```

### optional() with Defaults (1.3+)

**Use optional() for object attributes:**

```hcl
# ✅ GOOD - Using optional() for object attributes
variable "database_config" {
  description = "Database configuration with optional parameters"
  type = object({
    name               = string
    database_version   = string
    tier               = string
    backup_days        = optional(number, 7)
    high_availability  = optional(bool, true)
    labels             = optional(map(string), {})
  })
}
```

### Moved Blocks (1.1+)

**Rename resources without destroy/recreate:**

```hcl
# Rename a resource
moved {
  from = google_compute_instance.web_server
  to   = google_compute_instance.web
}

# Rename a module
moved {
  from = module.old_module_name
  to   = module.new_module_name
}

# Move resource into for_each
moved {
  from = google_compute_subnetwork.private[0]
  to   = google_compute_subnetwork.private["us-central1-a"]
}
```

### State Encryption (1.7+)

**Client-side state encryption with GCP KMS:**

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

### Early Variable Evaluation (1.8+)

**Use variables in backend and module source:**

```hcl
variable "environment" {
  type = string
}

tofu {
  backend "gcs" {
    bucket = "tofu-state-${var.environment}"
    prefix = var.environment
  }
}

module "vpc" {
  source = "github.com/myorg/tofu-google-vpc?ref=${var.module_version}"
}
```

### Cross-Variable Validation (1.9+)

**Reference other variables in validation blocks:**

```hcl
variable "machine_type" {
  description = "GCE machine type"
  type        = string
}

variable "disk_size_gb" {
  description = "Boot disk size in GB"
  type        = number

  validation {
    # Can reference var.machine_type in OpenTofu 1.9+
    condition = !(
      startswith(var.machine_type, "e2-micro") &&
      var.disk_size_gb > 100
    )
    error_message = "Micro instances cannot have disk > 100 GB"
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "backup_retention_days" {
  description = "Backup retention period in days"
  type        = number

  validation {
    # Production requires longer retention
    condition = (
      var.environment == "prod" ? var.backup_retention_days >= 7 : true
    )
    error_message = "Production environment requires backup_retention >= 7 days"
  }
}
```

### Enabled Meta-Argument (1.11+)

**Cleaner conditional resource creation:**

```hcl
# ✅ GOOD - enabled meta-argument (OpenTofu 1.11+)
resource "google_compute_router_nat" "this" {
  enabled = var.create_nat

  name                               = "${var.name}-nat"
  router                             = google_compute_router.this.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Instead of:
resource "google_compute_router_nat" "this" {
  count = var.create_nat ? 1 : 0
  # ...
}
```

### Ephemeral Resources (1.11+)

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

---

## Version Management

### Version Constraint Syntax

```hcl
# Exact version (avoid unless necessary - inflexible)
version = "5.0.0"

# Pessimistic constraint (recommended for stability)
# Allows patch updates only
version = "~> 5.0"      # Allows 5.0.x (any x), but not 5.1.0
version = "~> 5.0.1"    # Allows 5.0.x where x >= 1, but not 5.1.0

# Range constraints
version = ">= 5.0, < 6.0"     # Any 5.x version

# Minimum version
version = ">= 5.0"  # Any version 5.0 or higher (risky - breaking changes)
```

### Versioning Strategy by Component

**OpenTofu itself:**
```hcl
# versions.tf
tofu {
  # Pin to minor version, allow patch updates
  required_version = "~> 1.11"  # Allows 1.11.x
}
```

**Providers:**
```hcl
# versions.tf
tofu {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"  # Pin major version, allow minor/patch updates
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.0"
    }
  }
}
```

**Modules:**
```hcl
# Production - pin exact version
module "vpc" {
  source  = "github.com/myorg/tofu-google-vpc?ref=v5.1.2"
}

# Development - allow flexibility
module "vpc" {
  source  = "github.com/myorg/tofu-google-vpc?ref=v5.1"
}
```

### Example versions.tf Template

```hcl
tofu {
  # OpenTofu version
  required_version = "~> 1.11"

  # Provider versions
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

---

## Refactoring Patterns

### Secrets Remediation

**Pattern:** Move secrets out of OpenTofu state into external secret management.

#### Before - Secrets in State

```hcl
# ❌ BAD - Secret generated and stored in state
resource "random_password" "db" {
  length  = 16
  special = true
}

resource "google_sql_database_instance" "this" {
  database_version = "POSTGRES_15"
  root_password    = random_password.db.result  # In state!
}
```

#### After - External Secret Management

**Option 1: Ephemeral resources (OpenTofu 1.11+)**

```hcl
# ✅ GOOD - Ephemeral resource, never in state
ephemeral "google_secret_manager_secret_version" "db_password" {
  secret = "projects/my-project/secrets/db-password"
}

resource "google_sql_database_instance" "this" {
  database_version = "POSTGRES_15"
  root_password    = ephemeral.google_secret_manager_secret_version.db_password.secret_data
}
```

**Option 2: Separate secret creation (pre-1.11)**

```hcl
# ✅ GOOD - Reference pre-existing secret
data "google_secret_manager_secret_version" "db_password" {
  secret = "prod-database-password"
}

resource "google_sql_database_instance" "this" {
  database_version = "POSTGRES_15"
  root_password    = data.google_secret_manager_secret_version.db_password.secret_data
}
```

---

## Locals for Dependency Management

**Use locals to hint explicit resource deletion order:**

```hcl
# ✅ GOOD - Forces correct deletion order
# Ensures subnets deleted before secondary CIDR blocks

locals {
  # References secondary CIDR first, falling back to VPC
  # This forces OpenTofu to delete subnets before CIDR association
  network_id = try(
    google_compute_network_peering.this[0].network,
    google_compute_network.this.id,
    ""
  )
}

resource "google_compute_network" "this" {
  name                    = var.network_name
  auto_create_subnetworks = false
}

resource "google_compute_network_peering" "this" {
  count = var.enable_peering ? 1 : 0

  name         = "peering"
  network      = google_compute_network.this.id
  peer_network = var.peer_network_id
}

resource "google_compute_subnetwork" "private" {
  # Uses local instead of direct reference
  # Creates implicit dependency on peering
  network    = local.network_id
  name       = "private"
  ip_cidr_range = "10.0.1.0/24"
  region     = var.region
}

# Without local: OpenTofu might try to delete peering before subnets
# With local: Subnets deleted first, then peering, then network
```

**Common use cases:**
- VPC with peering connections
- Resources that depend on optional configurations
- Complex deletion order requirements

---

## Orchestration Approaches

Five primary orchestration patterns for OpenTofu projects:

| Approach | Description | Best For |
|----------|-------------|----------|
| **OpenTofu only** | Direct CLI usage with workspaces | Simple projects, small teams |
| **Terragrunt** | DRY configurations, native module support | Medium-large projects, multiple environments |
| **In-house scripts** | Custom bash/Python wrappers | Specific workflow requirements |
| **Ansible** | Integration with configuration management | Mixed IaC/CM workflows |
| **Crossplane** | Kubernetes-native infrastructure | K8s-centric organizations |

### Choosing an Approach

For most GCP projects, **OpenTofu only** or **Terragrunt** are recommended:

**OpenTofu only** is ideal when:
- Team is small (1-5 people)
- Single environment or simple multi-environment setup
- Workspaces provide sufficient separation

**Terragrunt** is ideal when:
- Managing many environments or regions
- Need DRY configurations across environments
- Complex module dependencies
- Team prefers declarative configuration over scripts

**In-house scripts** work when:
- Existing CI/CD has specific requirements
- Need custom pre/post processing
- Integration with proprietary systems

---

**Back to:** [Main Skill File](../SKILL.md)
