# CLAUDE.md

`terraform-aws-dynamodb-table` — a reusable Harri Terraform **module**. No Harri
stack pins it today, so it is consumed by git source until it is published to the
HCP private registry.

Harri's fork of `terraform-aws-modules/dynamodb-table`. No Harri stack pins it
today — `TerraformDynamoDB` calls the upstream public module directly — so its
HCP registry address is unverified; use the git source below until a registry
consumer exists.

## What This Module Provisions

- `aws_dynamodb_table` — one table, created by one of two mutually exclusive
  resources depending on `autoscaling_enabled` (switching the flag re-creates
  the table unless state is moved).
- Attributes, hash/range keys, global and local secondary indexes, TTL,
  point-in-time recovery, streams, server-side encryption, table class, and
  deletion protection from the corresponding inputs.
- Application Auto Scaling for provisioned tables — `aws_appautoscaling_target`
  and `aws_appautoscaling_policy` for table read/write and per-index read/write.
- Global-table replicas via `replica_regions`.

## Registry Source

```hcl
module "dynamodb_table" {
  source = "git::ssh://git@github.com/HarriLLC/terraform-aws-dynamodb-table.git?ref=vX.Y.Z"

  # no required inputs — set at least these in practice
  name     = "prep_my_service_table"
  hash_key = "id"
  attributes = [{ name = "id", type = "S" }]
}
```

> Take the current published version from the HCP private registry, or from the
> `version` pinned beside this `source` in a consumer's `common/resources.tf`.
> Consumers pin an exact version; ranges are never allowed.

## Required Versions

Read them from **`versions.tf`**: `required_version` for Terraform core, and each
`required_providers` entry for the providers. They are not repeated here, so this
guide cannot go stale when they change. A module's provider constraint is
intentionally a range, so consumers can pin an exact version themselves.

## File Layout

```
terraform-aws-dynamodb-table/
├── variables.tf     # 26 inputs, all optional (upstream shape)
├── main.tf          # aws_dynamodb_table (this / autoscaled) — legacy name; resources.tf is the standard
├── autoscaling.tf   # appautoscaling targets and policies
├── outputs.tf       # table arn/id, stream arn/label
├── versions.tf      # required_version + required_providers
├── wrappers/        # upstream for_each wrapper
├── examples/        # autoscaling, basic, global-tables
├── CHANGELOG.md     # upstream changelog (ends at 2.0.0)
└── README.md        # upstream README (source line names the public module)
```

## Inputs

| Name | Type | Default | Description |
|---|---|---|---|
| `create_table` | `bool` | `true` | Create the table and associated resources. |
| `name` | `string` | `null` | Table name. |
| `attributes` | `list(map(string))` | `[]` | Attribute definitions (`name`, `type` S/N/B) for keys and indexes. |
| `hash_key` | `string` | `null` | Partition key attribute. |
| `range_key` | `string` | `null` | Sort key attribute. |
| `billing_mode` | `string` | `"PAY_PER_REQUEST"` | `PROVISIONED` or `PAY_PER_REQUEST`. |
| `write_capacity` | `number` | `null` | Write units (PROVISIONED). |
| `read_capacity` | `number` | `null` | Read units (PROVISIONED). |
| `point_in_time_recovery_enabled` | `bool` | `false` | Enable PITR. |
| `ttl_enabled` | `bool` | `false` | Enable TTL. |
| `ttl_attribute_name` | `string` | `""` | TTL timestamp attribute. |
| `global_secondary_indexes` | `any` | `[]` | GSI definitions. |
| `local_secondary_indexes` | `any` | `[]` | LSI definitions (creation-time only). |
| `replica_regions` | `any` | `[]` | Regions for global-table replicas. |
| `stream_enabled` | `bool` | `false` | Enable streams. |
| `stream_view_type` | `string` | `null` | `KEYS_ONLY`, `NEW_IMAGE`, `OLD_IMAGE`, `NEW_AND_OLD_IMAGES`. |
| `server_side_encryption_enabled` | `bool` | `false` | Encrypt with a KMS CMK. |
| `server_side_encryption_kms_key_arn` | `string` | `null` | CMK ARN (defaults to `alias/aws/dynamodb`). |
| `tags` | `map(string)` | `{}` | Tags for all resources. |
| `timeouts` | `map(string)` | `{ create/update/delete = "10m" }` | Resource timeouts. |
| `autoscaling_enabled` | `bool` | `false` | Enable autoscaling (selects the autoscaled table resource). |
| `autoscaling_defaults` | `map(string)` | `{ scale_in_cooldown = 0, scale_out_cooldown = 0, target_value = 70 }` | Default autoscaling settings. |
| `autoscaling_read` | `map(string)` | `{}` | Read autoscaling (`max_capacity` required). |
| `autoscaling_write` | `map(string)` | `{}` | Write autoscaling (`max_capacity` required). |
| `autoscaling_indexes` | `map(map(string))` | `{}` | Per-index autoscaling. |
| `table_class` | `string` | `null` | `STANDARD` or `STANDARD_INFREQUENT_ACCESS`. |

## Outputs

| Name | Description |
|---|---|
| `dynamodb_table_arn` | Table ARN. |
| `dynamodb_table_id` | Table name/id. |
| `dynamodb_table_stream_arn` | Stream ARN (when streams are enabled). |
| `dynamodb_table_stream_label` | Stream label. |

## Examples

- `examples/basic` — a PAY_PER_REQUEST table with a GSI and TTL.
- `examples/autoscaling` — a PROVISIONED table with read/write and index autoscaling.
- `examples/global-tables` — replicas in additional regions.

## Publishing a New Version

1. Make the changes; run `terraform fmt` + `terraform validate` (and
   `terraform test` only if a `tests/` directory exists).
2. Pick the version by semver: adding a **required** input, or renaming or
   removing an output, is a **breaking change → major bump**; a new optional
   input → minor; a fix → patch.
3. Update `README.md` (and `CHANGELOG.md` if the module keeps one).
4. Tag and push **that one tag**: `git tag vX.Y.Z && git push origin vX.Y.Z`.
5. HCP Registry auto-publishes from the tag; bump consumers' `version` pins to
   the exact, `v`-stripped value (`X.Y.Z`, not `vX.Y.Z`).

## Conventions & Naming

House style, module conventions, the local validate flow and the publish rules
all load automatically from the `harri-tf-house-style` and `harri-tf-modules`
rules whenever a `.tf` file in this module is edited — they are not repeated here.

### Project naming patterns

- Harri table names are `snake_case` with the env prefix
  (`prep_my_service_table`), matching `TerraformDynamoDB`.
- Every attribute named by `hash_key`, `range_key`, or an index must also appear
  in `attributes`.
- Toggling `autoscaling_enabled` moves the table between two resource addresses
  (`aws_dynamodb_table.this` ↔ `.autoscaled`); add a `moved` block or state move
  first, or the table is destroyed and re-created.
- Registry address if it is ever published: `app.terraform.io/harri/dynamodb-table/aws`
  by Harri convention — unverified; confirm in the HCP private registry.
