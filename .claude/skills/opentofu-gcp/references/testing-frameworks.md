# Testing Frameworks Guide

> **Part of:** [opentofu-skill-gcp](../SKILL.md)
> **Purpose:** Deep dive into testing approaches for OpenTofu/GCP

This document provides comprehensive testing strategies from static analysis to full integration testing with GCP resources.

---

## Table of Contents

1. [Testing Pyramid](#testing-pyramid)
2. [Static Analysis](#static-analysis)
3. [Native Test Framework](#native-test-framework)
4. [Terratest for GCP](#terratest-for-gcp)
5. [Mock Providers](#mock-providers)
6. [Cost Optimization](#cost-optimization)

---

## Testing Pyramid

```
        /\
       /  \          End-to-End Tests (Expensive)
      /____\         - Full environment deployment
     /      \        - Production-like setup in GCP
    /________\
   /          \      Integration Tests (Moderate)
  /____________\     - Module testing in isolation
 /              \    - Real GCP resources in test project
/________________\   Static Analysis (Cheap)
                     - tofu validate, fmt, lint
                     - trivy, checkov security scans
```

### Testing Strategy by Cost

| Level | Tools | GCP Cost | When to Run |
|-------|-------|----------|-------------|
| Static | validate, fmt, tflint, trivy, checkov | Free | Every commit |
| Unit (Mocked) | tofu test with mocks | Free | Every PR |
| Integration | tofu test, Terratest | $ | Main branch |
| End-to-End | Full deployment | $$ | Scheduled/Release |

---

## Static Analysis

### Format and Validate

```bash
# Check formatting
tofu fmt -check -recursive

# Validate configuration
tofu validate
```

### TFLint with GCP Plugin

```bash
# Install tflint
brew install tflint  # macOS
# OR
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Initialize with GCP plugin
tflint --init

# Run with GCP rules
tflint --enable-plugin=google
```

**.tflint.hcl configuration:**

```hcl
# .tflint.hcl
plugin "google" {
  enabled = true
  version = "0.28.0"
  source  = "github.com/terraform-linters/tflint-ruleset-google"
}

rule "google_compute_instance_invalid_machine_type" {
  enabled = true
}

rule "google_container_cluster_node_version" {
  enabled = true
}
```

### Security Scanning

```bash
# Trivy - IaC misconfiguration
trivy config . --severity CRITICAL,HIGH

# Checkov - GCP policy compliance
checkov -d . --framework terraform --check CKV_GCP*
```

---

## Native Test Framework

### Overview (OpenTofu 1.6+)

The native test framework provides built-in testing without external tools.

**File naming:** `*.tftest.hcl` or `tests/*.tftest.hcl`

### Basic Test Structure

```hcl
# tests/vpc_test.tftest.hcl

# Variables for tests
variables {
  project_id   = "test-project"
  region       = "us-central1"
  network_name = "test-vpc"
}

# Test: VPC creation with plan mode (fast, no resources created)
run "vpc_creates_correctly" {
  command = plan

  assert {
    condition     = google_compute_network.this.name == "test-vpc"
    error_message = "VPC name should be 'test-vpc'"
  }

  assert {
    condition     = google_compute_network.this.auto_create_subnetworks == false
    error_message = "Auto-create subnetworks should be disabled"
  }
}

# Test: Validation rules work
run "invalid_region_fails" {
  command = plan

  variables {
    region = "invalid-region"
  }

  expect_failures = [var.region]
}
```

### Command Modes

| Mode | Use Case | Creates Resources | Speed |
|------|----------|-------------------|-------|
| `command = plan` | Input validation, logic testing | No | Fast |
| `command = apply` | Integration tests, computed values | Yes | Slow |

### Testing with Mocks (OpenTofu 1.7+)

```hcl
# tests/compute_test.tftest.hcl

# Mock the Google provider
mock_provider "google" {
  mock_data "google_compute_zones" {
    defaults = {
      names = ["us-central1-a", "us-central1-b", "us-central1-c"]
    }
  }

  mock_resource "google_compute_instance" {
    defaults = {
      id = "test-instance-id"
    }
  }
}

run "compute_instance_creates" {
  command = plan

  assert {
    condition     = google_compute_instance.web.machine_type == "e2-medium"
    error_message = "Instance should use e2-medium"
  }
}
```

### Testing Modules

```hcl
# tests/module_test.tftest.hcl

# Test the VPC module
run "vpc_module_test" {
  command = plan

  module {
    source = "./modules/vpc"
  }

  variables {
    network_name = "test-vpc"
    subnets = {
      "private-us-central1" = {
        cidr   = "10.0.1.0/24"
        region = "us-central1"
      }
    }
  }

  assert {
    condition     = length(google_compute_subnetwork.this) == 1
    error_message = "Should create 1 subnet"
  }
}
```

### GCP-Specific Test Patterns

**Testing GKE Cluster:**

```hcl
# tests/gke_test.tftest.hcl

variables {
  project_id   = "test-project"
  region       = "us-central1"
  cluster_name = "test-cluster"
}

run "gke_cluster_is_private" {
  command = plan

  assert {
    condition     = google_container_cluster.primary.private_cluster_config[0].enable_private_nodes == true
    error_message = "GKE cluster must use private nodes"
  }

  assert {
    condition     = google_container_cluster.primary.remove_default_node_pool == true
    error_message = "Default node pool should be removed"
  }
}

run "gke_has_required_labels" {
  command = plan

  assert {
    condition     = google_container_cluster.primary.resource_labels["managed-by"] == "opentofu"
    error_message = "Cluster must have managed-by label"
  }
}
```

**Testing Cloud SQL:**

```hcl
# tests/cloudsql_test.tftest.hcl

run "cloudsql_is_private" {
  command = plan

  assert {
    condition     = google_sql_database_instance.main.settings[0].ip_configuration[0].ipv4_enabled == false
    error_message = "Cloud SQL must not have public IP"
  }

  assert {
    condition     = google_sql_database_instance.main.settings[0].ip_configuration[0].private_network != null
    error_message = "Cloud SQL must use private network"
  }
}

run "cloudsql_has_backups" {
  command = plan

  assert {
    condition     = google_sql_database_instance.main.settings[0].backup_configuration[0].enabled == true
    error_message = "Backups must be enabled"
  }
}
```

**Testing Pub/Sub:**

```hcl
# tests/pubsub_test.tftest.hcl

run "pubsub_has_labels" {
  command = plan

  assert {
    condition     = google_pubsub_topic.events.labels["environment"] != null
    error_message = "Topic must have environment label"
  }
}

run "subscription_has_retention" {
  command = plan

  assert {
    condition     = google_pubsub_subscription.events.message_retention_duration == "604800s"
    error_message = "Subscription should retain messages for 7 days"
  }
}
```

### command = plan vs command = apply

**Critical decision:** When to use each command mode

#### Use `command = plan`

**When:**
- Checking input validation
- Verifying resource will be created
- Testing variable defaults
- Checking resource attributes that are **input-derived** (not computed)

**Example:**
```hcl
run "test_input_validation" {
  command = plan  # Fast, no resource creation

  variables {
    network_name = "test-vpc"
  }

  assert {
    # network name is an input, known at plan time
    condition     = google_compute_network.this.name == "test-vpc"
    error_message = "Network name should match input"
  }
}
```

#### Use `command = apply`

**When:**
- Checking computed attributes (IDs, self_links, generated names)
- Accessing set-type blocks
- Verifying actual resource behavior
- Testing with real/mocked provider responses

**Example:**
```hcl
run "test_computed_values" {
  command = apply  # Executes and gets computed values

  variables {
    network_name = "test-vpc"
  }

  assert {
    # self_link is computed, only known after apply
    condition     = length(google_compute_network.this.self_link) > 0
    error_message = "Network should have self_link"
  }
}
```

---

## Terratest for GCP

### Setup

```go
// go.mod
module github.com/myorg/infra-tests

go 1.21

require (
    github.com/gruntwork-io/terratest v0.46.0
    github.com/stretchr/testify v1.8.4
    google.golang.org/api v0.150.0
)
```

### Basic GCP Test

```go
// tests/vpc_test.go
package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/gcp"
    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()

    projectID := gcp.GetGoogleProjectIDFromEnvVar(t)
    region := "us-central1"
    uniqueID := random.UniqueId()
    networkName := fmt.Sprintf("test-vpc-%s", uniqueID)

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/simple",
        Vars: map[string]interface{}{
            "project_id":   projectID,
            "region":       region,
            "network_name": networkName,
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    vpcName := terraform.Output(t, terraformOptions, "vpc_name")

    // Verify outputs
    assert.NotEmpty(t, vpcID)
    assert.Equal(t, networkName, vpcName)

    // Verify VPC exists in GCP
    network := gcp.GetNetwork(t, projectID, networkName)
    assert.NotNil(t, network)
    assert.False(t, network.AutoCreateSubnetworks)
}
```

### Testing GKE Cluster

```go
// tests/gke_test.go
package test

import (
    "fmt"
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/gcp"
    "github.com/gruntwork-io/terratest/modules/k8s"
    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestGKECluster(t *testing.T) {
    t.Parallel()

    projectID := gcp.GetGoogleProjectIDFromEnvVar(t)
    region := "us-central1"
    uniqueID := random.UniqueId()
    clusterName := fmt.Sprintf("test-gke-%s", uniqueID)

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/gke",
        Vars: map[string]interface{}{
            "project_id":   projectID,
            "region":       region,
            "cluster_name": clusterName,
            "environment":  "test",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Get cluster endpoint
    clusterEndpoint := terraform.Output(t, terraformOptions, "cluster_endpoint")
    assert.NotEmpty(t, clusterEndpoint)

    // Get kubeconfig
    kubeconfig := terraform.Output(t, terraformOptions, "kubeconfig")

    // Test cluster connectivity
    kubectlOptions := k8s.NewKubectlOptions("", kubeconfig, "default")
    k8s.WaitUntilAllNodesReady(t, kubectlOptions, 10, 30*time.Second)

    // Verify nodes are running
    nodes := k8s.GetNodes(t, kubectlOptions)
    assert.GreaterOrEqual(t, len(nodes), 1)
}
```

### Testing Cloud SQL

```go
// tests/cloudsql_test.go
package test

import (
    "fmt"
    "testing"

    "github.com/gruntwork-io/terratest/modules/gcp"
    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestCloudSQL(t *testing.T) {
    t.Parallel()

    projectID := gcp.GetGoogleProjectIDFromEnvVar(t)
    region := "us-central1"
    uniqueID := random.UniqueId()
    instanceName := fmt.Sprintf("test-db-%s", uniqueID)

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/cloudsql",
        Vars: map[string]interface{}{
            "project_id":    projectID,
            "region":        region,
            "instance_name": instanceName,
            "environment":   "test",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Get connection info
    connectionName := terraform.Output(t, terraformOptions, "connection_name")
    assert.NotEmpty(t, connectionName)

    // Verify instance exists and is RUNNABLE
    instance := gcp.GetCloudSQLInstance(t, projectID, instanceName)
    assert.Equal(t, "RUNNABLE", instance.State)
}
```

---

## Mock Providers

### When to Use Mocks (OpenTofu 1.7+)

| Scenario | Use Mocks? | Why |
|----------|------------|-----|
| Input validation | Yes | No cloud resources needed |
| Logic testing | Yes | Testing conditionals, loops |
| Computed values | No | Need real provider |
| Integration tests | No | Testing real behavior |
| Cost-sensitive CI | Yes | Reduce cloud spend |

### Mock Provider Example

```hcl
# tests/mocked_test.tftest.hcl

mock_provider "google" {
  # Mock zone data source
  mock_data "google_compute_zones" {
    defaults = {
      names = ["us-central1-a", "us-central1-b", "us-central1-c"]
    }
  }

  # Mock instance creation
  mock_resource "google_compute_instance" {
    defaults = {
      id           = "projects/test/zones/us-central1-a/instances/test"
      instance_id  = "1234567890"
      self_link    = "https://compute.googleapis.com/..."
    }
  }

  # Mock network
  mock_resource "google_compute_network" {
    defaults = {
      id        = "projects/test/global/networks/test-vpc"
      self_link = "https://compute.googleapis.com/..."
    }
  }
}

run "instance_uses_correct_zone" {
  command = plan

  assert {
    condition     = google_compute_instance.web.zone == "us-central1-a"
    error_message = "Instance should be in us-central1-a"
  }
}
```

---

## Cost Optimization

### Strategy Overview

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| Use mocks for unit tests | ~90% | OpenTofu 1.7+ mocks |
| Smallest instance types | ~70% | e2-micro, db-f1-micro |
| Run integration on main only | ~80% | CI/CD conditional |
| TTL labels + cleanup | 100% of orphans | Scheduled cleanup job |

### Implementing Cost Controls

**1. Use test-specific machine types:**

```hcl
variable "environment" {
  type = string
}

locals {
  machine_types = {
    test = "e2-micro"
    dev  = "e2-small"
    prod = "e2-medium"
  }
}

resource "google_compute_instance" "app" {
  machine_type = local.machine_types[var.environment]
  # ...
}
```

**2. Add TTL labels:**

```hcl
locals {
  test_labels = {
    environment = "test"
    ttl         = "2h"
    created-at  = formatdate("YYYY-MM-DD", timestamp())
  }
}
```

**3. Conditional integration tests:**

```yaml
# GitHub Actions
integration-test:
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - run: go test -v ./tests/integration/...
```

---

## Best Practices

### Test Organization

```
tests/
├── unit/                       # Native tests with mocks
│   ├── validation_test.tftest.hcl
│   └── logic_test.tftest.hcl
├── integration/                # Native tests with real resources
│   ├── vpc_test.tftest.hcl
│   └── gke_test.tftest.hcl
└── e2e/                        # Terratest (if needed)
    ├── complete_test.go
    └── fixtures/
```

### Naming Conventions

```hcl
# Test file: <module>_test.tftest.hcl
# Test run: <action>_<expected_result>

run "vpc_creates_with_correct_name" { }
run "invalid_cidr_fails_validation" { }
run "gke_cluster_is_private" { }
```

### Cleanup Patterns

**Terratest with defer:**

```go
defer terraform.Destroy(t, terraformOptions)
terraform.InitAndApply(t, terraformOptions)
```

**Native tests auto-cleanup:**

Native tests with `command = apply` automatically destroy resources after the test run.

---

**Back to:** [Main Skill File](../SKILL.md)
