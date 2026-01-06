# Migration Guide: Monolithic `account` to Single-Resource `aws-organizational-unit`

This document outlines the migration from the monolithic `account` component to the new single-resource `aws-organizational-unit` component.

## Overview

The previous `account` component created all AWS Organizations resources (organization, OUs, accounts, SCPs) in a single Terraform state. The new `aws-organizational-unit` component follows the single-resource pattern - it manages only a single OU.

### Why Migrate?

| Aspect | Old `account` Component | New `aws-organizational-unit` Component |
|--------|-------------------------|----------------------------------------|
| **Scope** | Entire organization hierarchy | Single OU |
| **State** | All resources in one state | Independent state per OU |
| **Lifecycle** | Changes affect all OUs | Changes isolated to one OU |
| **Risk** | High blast radius | Minimal blast radius |

### New Component Suite

The monolithic `account` component is replaced by these single-resource components:

| Component | Purpose |
|-----------|---------|
| `aws-organization` | Creates/imports the AWS Organization |
| `aws-organizational-unit` | Creates/imports a single OU (this component) |
| `aws-account` | Creates/imports a single AWS Account |
| `aws-account-settings` | Configures account settings |
| `aws-scp` | Creates/imports Service Control Policies |

---

## Migration Steps

### Phase 1: Get OU IDs

```bash
# Get the root ID first
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)

# List all top-level OUs
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --query 'OrganizationalUnits[*].[Name,Id]' --output table

# Example output:
# |   Name   |        Id           |
# |----------|---------------------|
# |  core    | ou-xxxx-11111111    |
# |  plat    | ou-xxxx-22222222    |
```

### Phase 2: Create Stack Configuration

Create a component instance for each OU:

```yaml
# stacks/orgs/<namespace>/core/root/global-region.yaml
components:
  terraform:
    # Core OU
    aws-organizational-unit/core:
      metadata:
        component: aws-organizational-unit
      vars:
        name: core
        parent_id: "r-xxxx"  # Organization root ID
        import_resource_id: "ou-xxxx-11111111"

    # Platform OU
    aws-organizational-unit/plat:
      metadata:
        component: aws-organizational-unit
      vars:
        name: plat
        parent_id: "r-xxxx"
        import_resource_id: "ou-xxxx-22222222"
```

### Phase 3: Import OUs

```bash
# Import each OU
atmos terraform apply aws-organizational-unit/core -s <namespace>-gbl-root
atmos terraform apply aws-organizational-unit/plat -s <namespace>-gbl-root
```

### Phase 4: Remove from Old Component State

> [!CAUTION]
> Use `terraform state rm` to remove resources from state without destroying them.

```bash
# Remove OUs from old state
atmos terraform state rm account -s <namespace>-gbl-root \
  'aws_organizations_organizational_unit.this["core"]'

atmos terraform state rm account -s <namespace>-gbl-root \
  'aws_organizations_organizational_unit.this["plat"]'
```

### Phase 5: Clean Up

After successful import, remove `import_resource_id` from each configuration:

```yaml
components:
  terraform:
    aws-organizational-unit/core:
      metadata:
        component: aws-organizational-unit
      vars:
        name: core
        parent_id: "r-xxxx"
        # Remove after import:
        # import_resource_id: "ou-xxxx-11111111"
```

---

## Nested OUs

For nested OUs, reference the parent OU's ID:

```yaml
components:
  terraform:
    # Parent OU
    aws-organizational-unit/core:
      vars:
        name: core
        parent_id: "r-xxxx"

    # Child OU (nested under core)
    aws-organizational-unit/core-security:
      vars:
        name: security
        parent_id: !terraform.output aws-organizational-unit/core organizational_unit_id
```

---

## Troubleshooting

### Import Block Not Working

Ensure you're using OpenTofu >= 1.7.0 (required for `for_each` in `import` blocks).

If you excluded `imports.tf` when vendoring, use manual import:

```bash
atmos terraform import aws-organizational-unit/core -s <namespace>-gbl-root \
  'aws_organizations_organizational_unit.this[0]' 'ou-xxxx-11111111'
```

### OU Already Managed Error

This means the OU is being managed in both states. Complete Phase 4 first.

---

## References

- [OpenTofu Import Blocks](https://opentofu.org/docs/language/import/)
- [AWS Organizations OUs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_ous.html)
