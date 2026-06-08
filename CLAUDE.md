# CLAUDE.md

## What This Module Provisions

- `aws_dynamodb_table.this` — created when `create_table = true` and `autoscaling_enabled = false`.
- `aws_dynamodb_table.autoscaled` — alternate table used when `autoscaling_enabled = true` (has `lifecycle.ignore_changes` on `read_capacity`/`write_capacity`).
- `aws_appautoscaling_target` / `aws_appautoscaling_policy` — read + write target-tracking autoscaling for the table and per-index (`autoscaling.tf`).
- Inline blocks on the table: `ttl`, `point_in_time_recovery`, `server_side_encryption`, `stream`, `attribute`, `local_secondary_index`, `global_secondary_index`, `replica` (global tables).

> Toggling `autoscaling_enabled` switches between the `this` and `autoscaled` resources and **recreates the table** — use `terraform state mv` to migrate (see README Notes).

## Registry Source

```hcl
module "dynamodb_table" {
  source  = "app.terraform.io/harri/dynamodb-table/aws"
  version = "x.y.z"

  name     = "my-table"
  hash_key = "id"

  attributes = [
    { name = "id", type = "N" }
  ]
}
```

> This is a Harri-vendored copy of the upstream `terraform-aws-modules/dynamodb-table` module. The README still shows the public `terraform-aws-modules/dynamodb-table/aws` source; consumers in Harri pin the private registry source above.

## Required Versions

| Component | Constraint |
|---|---|
| Terraform | `>= 1.0` |
| AWS provider | `>= 4.23` |

Source of truth: `versions.tf`. (Below the Harri module standard of `>= 1.9.0` / `>= 5.0` — see `.tf-init-notes.md`.)

## File Layout

```
terraform-aws-dynamodb-table/
├── variables.tf      # Input variables
├── main.tf           # aws_dynamodb_table (this + autoscaled)
├── autoscaling.tf    # appautoscaling targets + policies
├── outputs.tf        # Values exposed to consumers
├── versions.tf       # required_version + required_providers
├── README.md         # Usage example, inputs/outputs reference
├── examples/         # basic, autoscaling, global-tables
└── wrappers/         # Terragrunt-style wrapper module
```

## Inputs

| Name | Type | Default | Description |
|---|---|---|---|
| `create_table` | `bool` | `true` | Controls if the table and associated resources are created |
| `name` | `string` | `null` | Name of the DynamoDB table |
| `attributes` | `list(map(string))` | `[]` | Attribute definitions (`name`, `type` = S/N/B) |
| `hash_key` | `string` | `null` | Hash (partition) key; must be in `attributes` |
| `range_key` | `string` | `null` | Range (sort) key; must be in `attributes` |
| `billing_mode` | `string` | `"PAY_PER_REQUEST"` | `PROVISIONED` or `PAY_PER_REQUEST` |
| `write_capacity` | `number` | `null` | Write units (required if `PROVISIONED`) |
| `read_capacity` | `number` | `null` | Read units (required if `PROVISIONED`) |
| `point_in_time_recovery_enabled` | `bool` | `false` | Enable PITR |
| `ttl_enabled` | `bool` | `false` | Enable TTL |
| `ttl_attribute_name` | `string` | `""` | Attribute storing the TTL timestamp |
| `global_secondary_indexes` | `any` | `[]` | GSI definitions |
| `local_secondary_indexes` | `any` | `[]` | LSI definitions (creation-time only) |
| `replica_regions` | `any` | `[]` | Replica regions for a global table |
| `stream_enabled` | `bool` | `false` | Enable DynamoDB Streams |
| `stream_view_type` | `string` | `null` | `KEYS_ONLY` / `NEW_IMAGE` / `OLD_IMAGE` / `NEW_AND_OLD_IMAGES` |
| `server_side_encryption_enabled` | `bool` | `false` | Enable SSE with a KMS CMK |
| `server_side_encryption_kms_key_arn` | `string` | `null` | CMK ARN (defaults to `alias/aws/dynamodb`) |
| `tags` | `map(string)` | `{}` | Tags applied to all resources |
| `timeouts` | `map(string)` | `{create=10m, update=60m, delete=10m}` | Resource management timeouts |
| `autoscaling_enabled` | `bool` | `false` | Enable autoscaling (switches resource — recreates table) |
| `autoscaling_defaults` | `map(string)` | `{scale_in/out_cooldown=0, target_value=70}` | Default autoscaling settings |
| `autoscaling_read` | `map(string)` | `{}` | Read autoscaling (`max_capacity` required) |
| `autoscaling_write` | `map(string)` | `{}` | Write autoscaling (`max_capacity` required) |
| `autoscaling_indexes` | `map(map(string))` | `{}` | Per-index autoscaling configs |
| `table_class` | `string` | `null` | `STANDARD` or `STANDARD_INFREQUENT_ACCESS` |

> No variable carries a `validation` block; constrained values (`billing_mode`, `stream_view_type`, `table_class`) are validated by the AWS provider at plan/apply time.

## Outputs

| Name | Description |
|---|---|
| `dynamodb_table_arn` | ARN of the DynamoDB table |
| `dynamodb_table_id` | ID of the DynamoDB table |
| `dynamodb_table_stream_arn` | Stream ARN (only when `stream_enabled`) |
| `dynamodb_table_stream_label` | Stream label, ISO 8601 (only when `stream_enabled`) |

## Examples

- `examples/basic` — minimal table with a hash key.
- `examples/autoscaling` — provisioned table with read/write/index autoscaling.
- `examples/global-tables` — multi-region global table via `replica_regions`.

## Validate Changes Locally

- Run `terraform fmt` before opening a PR (always — CI may fail otherwise).
- From the module dir: `terraform init -upgrade=false`.
- Run `terraform validate`.
- **Do not run `terraform plan` or `terraform apply` locally** — modules don't hold state on their own; that happens in the consumer stack.

## Publishing a New Version

1. Make the changes and run `terraform validate` (no `terraform test` suite present).
2. Update `CHANGELOG.md` / `README.md` if behaviour changes.
3. Tag the commit: `git tag vX.Y.Z && git push --tags`.
4. HCP Registry auto-publishes from the tag; bump consumers' `version` pins.

## Conventions & Naming

### Project naming patterns

- Table name comes straight from `var.name`; a `Name` tag is merged onto every table via `merge(var.tags, { Name = var.name })`.
- Two mutually-exclusive table resources (`this` vs `autoscaled`) gated by `autoscaling_enabled`; keep both in sync when editing table arguments.
- Autoscaling policy names follow `DynamoDB{Read|Write}CapacityUtilization:<resource_id>`.
- Nested index/replica/attribute config is expressed via `dynamic` blocks over `any`-typed list/map variables.

### Universal house style

- Variables: `type` + `description` required; snake_case; `nullable` declared explicitly. Use `optional(type, default)` for nested object fields.
- Module / component versions pinned to exact semver (e.g. `version = "4.1.1"`), not ranges.
- **Comments**: use `#` for both single-line and multi-line comments. Only comment to clarify non-obvious intent.
- **No hardcoded secrets**: never put credentials, tokens, or keys in Terraform files. Source them from env vars, TFC variable sets, or Vault (for modules, the consumer supplies them). Mark secret-holding variables with `sensitive = true`.
- **Indentation**: two spaces per nesting level (`terraform fmt` enforces — required before every PR).
- **Variable block field order**: `type`, `description`, `default`, `sensitive`, `validation`.
- **Output block field order**: `type`, `description`, `value`, `sensitive`.
- **Resource argument order**: `count`/`for_each` first, then a blank line, then non-block arguments, then block arguments, then `lifecycle`, then `depends_on`.
- **Blank lines within blocks**: separate logical groups of arguments with empty lines.
- **`count` vs `for_each`**: use `count` for nearly identical instances; use `for_each` when arguments differ per instance.
- **Tags — don't duplicate `default_tags`**: shared/common tags are applied once at the provider level via `default_tags { tags = local.common_tags }`, so they already land on every resource. **Never re-declare those same tags on individual resources or map entries** — only add `tags` to a resource for values that are genuinely resource-specific and not already in `common_tags`. Duplicating the common tags per-resource is redundant and drifts.
- **Data sources**: live in a separate `data.tf` file, logically positioned before the resources that reference them.
- **`.gitignore`**: exclude `*.tfstate`, `*.tfstate.backup`, `.terraform/`, `*.tfplan`, and any `.tfvars` holding secrets. Keep `.terraform.lock.hcl` committed.

### Module-specific

- File set: `variables.tf`, `data.tf`, `locals.tf`, `resources.tf`, `outputs.tf`, `versions.tf`, `README.md`, `tests/`. (This module uses `main.tf` + `autoscaling.tf` rather than `resources.tf`.)
- Required versions standard: `terraform >= 1.9.0`, `aws >= 5.0`.
- Registry source convention: `app.terraform.io/harri/{name}/aws`.
- Publish: `git tag vX.Y.Z` → HCP Registry auto-publish.
- Tests: `terraform test` with `mock_provider "aws" {}`.
