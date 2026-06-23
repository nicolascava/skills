---
name: terraform-test
description: Comprehensive guide for writing and running Terraform tests. Use when creating test files (.tftest.hcl), writing test scenarios with run blocks, validating infrastructure behavior with assertions, mocking providers and data sources, testing module outputs and resource configurations, or troubleshooting Terraform test syntax and execution.
metadata:
  copyright: Copyright IBM Corp. 2026
  version: "0.0.1"
---

# Terraform Test

Terraform's built-in testing framework enables module authors to validate that configuration updates don't introduce breaking changes. Tests execute against temporary resources, protecting existing infrastructure and state files.

## Core Concepts

**Test File**: A `.tftest.hcl` or `.tftest.json` file containing test configuration and run blocks that validate your Terraform configuration.

**Test Block**: Optional configuration block that defines test-wide settings (available since Terraform 1.6.0).

**Run Block**: Defines a single test scenario with optional variables, provider configurations, and assertions. Each test file requires at least one run block.

**Assert Block**: Contains conditions that must evaluate to true for the test to pass. Failed assertions cause the test to fail.

**Mock Provider**: Simulates provider behavior without creating real infrastructure (available since Terraform 1.7.0).

**Test Modes**: Tests run in apply mode (default, creates real infrastructure) or plan mode (validates logic without creating resources).

## File Structure

Terraform test files use the `.tftest.hcl` or `.tftest.json` extension and are typically organized in a `tests/` directory. Use clear naming conventions to distinguish between unit tests (plan mode) and integration tests (apply mode):

```
my-module/
├── main.tf
├── variables.tf
├── outputs.tf
└── tests/
    ├── validation_unit_test.tftest.hcl      # Unit test (plan mode)
    ├── edge_cases_unit_test.tftest.hcl      # Unit test (plan mode)
    └── full_stack_integration_test.tftest.hcl  # Integration test (apply mode - creates real resources)
```

### Test File Components

A test file contains:
- **Zero to one** `test` block (configuration settings)
- **One to many** `run` blocks (test executions)
- **Zero to one** `variables` block (input values)
- **Zero to many** `provider` blocks (provider configuration)
- **Zero to many** `mock_provider` blocks (mock provider data, since v1.7.0)

**Important**: The order of `variables` and `provider` blocks doesn't matter. Terraform processes all values within these blocks at the beginning of the test operation.

## Test Configuration (`.tftest.hcl`)

### Test Block

The optional `test` block configures test-wide settings:

```hcl
test {
  parallel = true  # Enable parallel execution for all run blocks (default: false)
}
```

**Test Block Attributes:**
- `parallel` - Boolean, when set to `true`, enables parallel execution for all run blocks by default (default: `false`). Individual run blocks can override this setting.

### Run Block

Each `run` block executes a command against your configuration. Run blocks execute **sequentially by default**.

**Basic Integration Test (Apply Mode - Default):**

```hcl
run "test_instance_creation" {
  command = apply

  assert {
    condition     = aws_instance.example.id != ""
    error_message = "Instance should be created with a valid ID"
  }

  assert {
    condition     = output.instance_public_ip != ""
    error_message = "Instance should have a public IP"
  }
}
```

**Unit Test (Plan Mode):**

```hcl
run "test_default_configuration" {
  command = plan

  assert {
    condition     = aws_instance.example.instance_type == "t2.micro"
    error_message = "Instance type should be t2.micro by default"
  }

  assert {
    condition     = aws_instance.example.tags["Environment"] == "test"
    error_message = "Environment tag should be 'test'"
  }
}
```

**Run Block Attributes:**

- `command` - Either `apply` (default) or `plan`
- `plan_options` - Configure plan behavior (see below)
- `variables` - Override test-level variable values
- `module` - Reference alternate modules for testing
- `providers` - Customize provider availability
- `assert` - Validation conditions (multiple allowed)
- `expect_failures` - Specify expected validation failures
- `state_key` - Manage state file isolation (since v1.9.0)
- `parallel` - Enable parallel execution when set to `true` (since v1.9.0)

### Plan Options

The `plan_options` block configures plan command behavior:

```hcl
run "test_refresh_only" {
  command = plan

  plan_options {
    mode    = refresh-only  # "normal" (default) or "refresh-only"
    refresh = true           # boolean, defaults to true
    replace = [
      aws_instance.example
    ]
    target = [
      aws_instance.example
    ]
  }

  assert {
    condition     = aws_instance.example.instance_type == "t2.micro"
    error_message = "Instance type should be t2.micro"
  }
}
```

**Plan Options Attributes:**
- `mode` - `normal` (default) or `refresh-only`
- `refresh` - Boolean, defaults to `true`
- `replace` - List of resource addresses to replace
- `target` - List of resource addresses to target

### Variables Block

Define variables at the test file level (applied to all run blocks) or within individual run blocks.

**Important**: Variables defined in test files take the **highest precedence**, overriding environment variables, variables files, or command-line input.

**File-Level Variables:**

```hcl
# Applied to all run blocks
variables {
  aws_region    = "us-west-2"
  instance_type = "t2.micro"
  environment   = "test"
}

run "test_with_file_variables" {
  command = plan

  assert {
    condition     = var.aws_region == "us-west-2"
    error_message = "Region should be us-west-2"
  }
}
```

**Run Block Variables (Override File-Level):**

```hcl
variables {
  instance_type = "t2.small"
  environment   = "test"
}

run "test_with_override_variables" {
  command = plan

  # Override file-level variables
  variables {
    instance_type = "t3.large"
  }

  assert {
    condition     = var.instance_type == "t3.large"
    error_message = "Instance type should be overridden to t3.large"
  }
}
```

**Variables Referencing Prior Run Blocks:**

```hcl
run "setup_vpc" {
  command = apply
}

run "test_with_vpc_output" {
  command = plan

  variables {
    vpc_id = run.setup_vpc.vpc_id
  }

  assert {
    condition     = var.vpc_id == run.setup_vpc.vpc_id
    error_message = "VPC ID should match setup_vpc output"
  }
}
```

### Assert Block

Assert blocks validate conditions within run blocks. All assertions must pass for the test to succeed.

**Syntax:**

```hcl
assert {
  condition     = <expression>
  error_message = "failure description"
}
```

**Resource Attribute Assertions:**

```hcl
run "test_resource_configuration" {
  command = plan

  assert {
    condition     = aws_s3_bucket.example.bucket == "my-test-bucket"
    error_message = "Bucket name should match expected value"
  }

  assert {
    condition     = aws_s3_bucket.example.versioning[0].enabled == true
    error_message = "Bucket versioning should be enabled"
  }

  assert {
    condition     = length(aws_s3_bucket.example.tags) > 0
    error_message = "Bucket should have at least one tag"
  }
}
```

**Output Validation:**

```hcl
run "test_outputs" {
  command = plan

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID output should not be empty"
  }

  assert {
    condition     = length(output.subnet_ids) == 3
    error_message = "Should create exactly 3 subnets"
  }
}
```

**Referencing Prior Run Block Outputs:**

```hcl
run "create_vpc" {
  command = apply
}

run "validate_vpc_output" {
  command = plan

  assert {
    condition     = run.create_vpc.vpc_id != ""
    error_message = "VPC from previous run should have an ID"
  }
}
```

**Complex Conditions:**

```hcl
run "test_complex_validation" {
  command = plan

  assert {
    condition = alltrue([
      for subnet in aws_subnet.private :
      can(regex("^10\\.0\\.", subnet.cidr_block))
    ])
    error_message = "All private subnets should use 10.0.0.0/8 CIDR range"
  }

  assert {
    condition = alltrue([
      for instance in aws_instance.workers :
      contains(["t2.micro", "t2.small", "t3.micro"], instance.instance_type)
    ])
    error_message = "Worker instances should use approved instance types"
  }
}
```

### Expect Failures Block

Test that certain conditions intentionally fail. The test **passes** if the specified checkable objects report an issue, and **fails** if they do not.

**Checkable objects include**: Input variables, output values, check blocks, and managed resources or data sources.

```hcl
run "test_invalid_input_rejected" {
  command = plan

  variables {
    instance_count = -1
  }

  expect_failures = [
    var.instance_count
  ]
}
```

**Testing Custom Conditions:**

```hcl
run "test_custom_condition_failure" {
  command = plan

  variables {
    instance_type = "t2.nano"  # Invalid type
  }

  expect_failures = [
    var.instance_type
  ]
}
```

### Module Block

Test a specific module rather than the root configuration.

**Supported Module Sources:**
- ✅ **Local modules**: `./modules/vpc`, `../shared/networking`
- ✅ **Public Terraform Registry**: `terraform-aws-modules/vpc/aws`
- ✅ **Private Registry (HCP Terraform)**: `app.terraform.io/org/module/provider`

**Unsupported Module Sources:**
- ❌ Git repositories: `git::https://github.com/...`
- ❌ HTTP URLs: `https://example.com/module.zip`
- ❌ Other remote sources (S3, GCS, etc.)

**Module Block Attributes:**
- `source` - Module source (local path or registry address)
- `version` - Version constraint (only for registry modules)

**Testing Local Modules:**

```hcl
run "test_vpc_module" {
  command = plan

  module {
    source = "./modules/vpc"
  }

  variables {
    cidr_block = "10.0.0.0/16"
    name       = "test-vpc"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR should match input variable"
  }
}
```

**Testing Public Registry Modules:**

```hcl
run "test_registry_module" {
  command = plan

  module {
    source  = "terraform-aws-modules/vpc/aws"
    version = "5.0.0"
  }

  variables {
    name = "test-vpc"
    cidr = "10.0.0.0/16"
  }

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC should be created"
  }
}
```

### Provider Configuration

Override or configure providers for tests. Since Terraform 1.7.0, provider blocks can reference test variables and prior run block outputs.

**Basic Provider Configuration:**

```hcl
provider "aws" {
  region = "us-west-2"
}

run "test_with_provider" {
  command = plan

  assert {
    condition     = aws_instance.example.availability_zone == "us-west-2a"
    error_message = "Instance should be in us-west-2 region"
  }
}
```

**Multiple Provider Configurations:**

```hcl
provider "aws" {
  alias  = "primary"
  region = "us-west-2"
}

provider "aws" {
  alias  = "secondary"
  region = "us-east-1"
}

run "test_with_specific_provider" {
  command = plan

  providers = {
    aws = provider.aws.secondary
  }

  assert {
    condition     = aws_instance.example.availability_zone == "us-east-1a"
    error_message = "Instance should be in us-east-1 region"
  }
}
```

**Provider with Test Variables:**

```hcl
variables {
  aws_region = "eu-west-1"
}

provider "aws" {
  region = var.aws_region
}
```

### State Key Management

The `state_key` attribute controls which state file a run block uses. By default:
- The main configuration shares a state file across all run blocks
- Each alternate module (referenced via `module` block) gets its own state file

**Force Run Blocks to Share State:**

```hcl
run "create_vpc" {
  command = apply

  module {
    source = "./modules/vpc"
  }

  state_key = "shared_state"
}

run "create_subnet" {
  command = apply

  module {
    source = "./modules/subnet"
  }

  state_key = "shared_state"  # Shares state with create_vpc
}
```

### Parallel Execution

Run blocks execute **sequentially by default**. Enable parallel execution with `parallel = true`.

**Requirements for Parallel Execution:**
- No inter-run output references (run blocks cannot reference outputs from parallel runs)
- Different state files (via different modules or state keys)
- Explicit `parallel = true` attribute

```hcl
run "test_module_a" {
  command  = plan
  parallel = true

  module {
    source = "./modules/module-a"
  }

  assert {
    condition     = output.result != ""
    error_message = "Module A should produce output"
  }
}

run "test_module_b" {
  command  = plan
  parallel = true

  module {
    source = "./modules/module-b"
  }

  assert {
    condition     = output.result != ""
    error_message = "Module B should produce output"
  }
}

# This creates a synchronization point
run "test_integration" {
  command = plan

  # Waits for parallel runs above to complete
  assert {
    condition     = output.combined != ""
    error_message = "Integration should work"
  }
}
```

## Mock Providers

Mock providers simulate provider behavior without creating real infrastructure (available since Terraform 1.7.0).

**Basic Mock Provider:**

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id            = "i-1234567890abcdef0"
      instance_type = "t2.micro"
      ami           = "ami-12345678"
    }
  }

  mock_data "aws_ami" {
    defaults = {
      id = "ami-12345678"
    }
  }
}

run "test_with_mocks" {
  command = plan

  assert {
    condition     = aws_instance.example.id == "i-1234567890abcdef0"
    error_message = "Mock instance ID should match"
  }
}
```

**Advanced Mock with Custom Values:**

```hcl
mock_provider "aws" {
  alias = "mocked"

  mock_resource "aws_s3_bucket" {
    defaults = {
      id     = "test-bucket-12345"
      bucket = "test-bucket"
      arn    = "arn:aws:s3:::test-bucket"
    }
  }

  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-west-2a", "us-west-2b", "us-west-2c"]
    }
  }
}

run "test_with_mock_provider" {
  command = plan

  providers = {
    aws = provider.aws.mocked
  }

  assert {
    condition     = length(data.aws_availability_zones.available.names) == 3
    error_message = "Should return 3 availability zones"
  }
}
```

## Test Execution

### Running Tests

**Run all tests:**

```bash
terraform test
```

**Run specific test file:**

```bash
terraform test tests/defaults.tftest.hcl
```

**Run with verbose output:**

```bash
terraform test -verbose
```

**Run tests in a specific directory:**

```bash
terraform test -test-directory=integration-tests
```

**Filter tests by name:**

```bash
terraform test -filter=test_vpc_configuration
```

**Run tests without cleanup (for debugging):**

```bash
terraform test -no-cleanup
```

### Test Output

**Successful test output:**

```
tests/defaults.tftest.hcl... in progress
  run "test_default_configuration"... pass
  run "test_outputs"... pass
tests/defaults.tftest.hcl... tearing down
tests/defaults.tftest.hcl... pass

Success! 2 passed, 0 failed.
```

**Failed test output:**

```
tests/defaults.tftest.hcl... in progress
  run "test_default_configuration"... fail
    Error: Test assertion failed
    Instance type should be t2.micro by default

Success! 0 passed, 1 failed.
```

## Common Test Patterns (Unit Tests - Plan Mode)

The following examples demonstrate common unit test patterns using `command = plan`. These tests validate Terraform logic without creating real infrastructure, making them fast and cost-free.

### Testing Module Outputs

```hcl
run "test_module_outputs" {
  command = plan

  assert {
    condition     = output.vpc_id != null
    error_message = "VPC ID output must be defined"
  }

  assert {
    condition     = can(regex("^vpc-", output.vpc_id))
    error_message = "VPC ID should start with 'vpc-'"
  }

  assert {
    condition     = length(output.subnet_ids) >= 2
    error_message = "Should output at least 2 subnet IDs"
  }
}
```

### Testing Resource Counts

```hcl
run "test_resource_count" {
  command = plan

  variables {
    instance_count = 3
  }

  assert {
    condition     = length(aws_instance.workers) == 3
    error_message = "Should create exactly 3 worker instances"
  }
}
```

### Testing Tags

```hcl
run "test_resource_tags" {
  command = plan

  variables {
    common_tags = {
      Environment = "production"
      ManagedBy   = "Terraform"
    }
  }

  assert {
    condition     = aws_instance.example.tags["Environment"] == "production"
    error_message = "Environment tag should be set correctly"
  }

  assert {
    condition     = aws_instance.example.tags["ManagedBy"] == "Terraform"
    error_message = "ManagedBy tag should be set correctly"
  }
}
```

### Sequential Tests with Dependencies

```hcl
run "setup_vpc" {
  # command defaults to apply

  variables {
    vpc_cidr = "10.0.0.0/16"
  }

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC should be created"
  }
}

run "test_subnet_in_vpc" {
  command = plan

  variables {
    vpc_id = run.setup_vpc.vpc_id
  }

  assert {
    condition     = aws_subnet.example.vpc_id == run.setup_vpc.vpc_id
    error_message = "Subnet should be created in the VPC from setup_vpc"
  }
}
```

## Integration Testing

For tests that create real infrastructure (default behavior with `command = apply`):

```hcl
run "integration_test_full_stack" {
  # command defaults to apply

  variables {
    environment = "integration-test"
    vpc_cidr    = "10.100.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.id != ""
    error_message = "VPC should be created"
  }

  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Should create 2 private subnets"
  }

  assert {
    condition     = aws_instance.bastion.public_ip != ""
    error_message = "Bastion instance should have a public IP"
  }
}

# Cleanup happens automatically after test completes
```

## Cleanup and Destruction

**Important**: Resources are destroyed in **reverse run block order** after test completion. This is critical for configurations with dependencies.

**Example**: For S3 buckets containing objects, the bucket must be emptied before deletion:

```hcl
run "create_bucket_with_objects" {
  command = apply

  assert {
    condition     = aws_s3_bucket.example.id != ""
    error_message = "Bucket should be created"
  }
}

run "add_objects_to_bucket" {
  command = apply

  assert {
    condition     = length(aws_s3_object.files) > 0
    error_message = "Objects should be added"
  }
}

# Cleanup occurs in reverse order:
# 1. Destroys objects (run "add_objects_to_bucket")
# 2. Destroys bucket (run "create_bucket_with_objects")
```

**Disable Cleanup for Debugging:**

```bash
terraform test -no-cleanup
```

## Best Practices

1. **Test isolation**: Each run block should be independent when possible. Use sequential runs only when testing dependencies.

2. **Mock vs real tradeoffs**: Use `command = plan` with mock providers for fast unit tests; use `command = apply` (default) for integration tests that validate real provider behavior. Mocks require Terraform 1.7.0+.

3. **Cleanup guarantees**: Integration tests destroy resources in reverse run block order automatically. Order run blocks so dependencies are destroyed last. Use `-no-cleanup` only for debugging.

4. **Assertion specificity**: Write assertions that check exact values, not just non-null. Include error messages that name the expected value to speed up diagnosis.

5. **Parallel test safety**: Only use `parallel = true` for run blocks with no inter-run output references and separate state files (different modules or `state_key` values).

6. **CI integration**: Run unit tests (plan mode) on every PR; run integration tests on merge or nightly. Use `terraform test -filter=` to target specific scenarios.

## Advanced Features

### Testing with Refresh-Only Mode

```hcl
run "test_refresh_only" {
  command = plan

  plan_options {
    mode = refresh-only
  }

  assert {
    condition     = aws_instance.example.tags["Environment"] == "production"
    error_message = "Tags should be refreshed correctly"
  }
}
```

### Testing with Targeted Resources

```hcl
run "test_specific_resource" {
  command = plan

  plan_options {
    target = [
      aws_instance.example
    ]
  }

  assert {
    condition     = aws_instance.example.instance_type == "t2.micro"
    error_message = "Targeted resource should be planned"
  }
}
```

### Testing Multiple Modules in Parallel

```hcl
run "test_networking_module" {
  command  = plan
  parallel = true

  module {
    source = "./modules/networking"
  }

  variables {
    cidr_block = "10.0.0.0/16"
  }

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC should be created"
  }
}

run "test_compute_module" {
  command  = plan
  parallel = true

  module {
    source = "./modules/compute"
  }

  variables {
    instance_type = "t2.micro"
  }

  assert {
    condition     = output.instance_id != ""
    error_message = "Instance should be created"
  }
}
```

### Custom State Management

```hcl
run "create_foundation" {
  command   = apply
  state_key = "foundation"

  assert {
    condition     = aws_vpc.main.id != ""
    error_message = "Foundation VPC should be created"
  }
}

run "create_application" {
  command   = apply
  state_key = "foundation"  # Share state with foundation

  variables {
    vpc_id = run.create_foundation.vpc_id
  }

  assert {
    condition     = aws_instance.app.vpc_id == run.create_foundation.vpc_id
    error_message = "Application should use foundation VPC"
  }
}
```

## Troubleshooting

- **Provider auth errors**: Configure credentials before running apply-mode tests, or switch to mock providers (`mock_provider`) for credential-free unit testing (requires Terraform 1.7.0+).
- **State conflicts**: Run blocks share state by default when using the root module. Use separate `module` blocks or the `state_key` attribute to isolate state between tests.
- **Mock limitations**: Mocks only work with `command = plan`. Mock defaults may not match real computed attributes; update mock `defaults` when provider schemas change.
- **Parallel test failures**: Run blocks using `parallel = true` must not reference each other's outputs and must operate on separate state files. Drop `parallel = true` to diagnose race conditions.

## Example test suite

See the "Common test patterns" section above for complete run block and assertion examples.

## CI/CD Integration

### GitHub Actions Example

```yaml
name: Terraform Tests

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  terraform-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Run Terraform Tests
        run: terraform test -verbose
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### GitLab CI Example

```yaml
terraform-test:
  image: hashicorp/terraform:1.9
  stage: test
  before_script:
    - terraform init
  script:
    - terraform fmt -check -recursive
    - terraform validate
    - terraform test -verbose
  only:
    - merge_requests
    - main
```

## References

For more information:
- [Terraform Testing Documentation](https://developer.hashicorp.com/terraform/language/tests)
- [Terraform Test Command Reference](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Testing Best Practices](https://developer.hashicorp.com/terraform/language/tests/best-practices)
