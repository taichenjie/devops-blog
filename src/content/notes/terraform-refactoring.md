---
title: "A Step-by-Step Guide to Refactoring Terraform Modules Without Destroying Infrastructure"
description: "The rules and sequence for extracting resources into modules using moved blocks, based on real refactoring across two PRs."
pubDate: 2026-08-28
draft: true
---

I am building a portfolio infrastructure project on AWS using Terraform. During M2 I extracted IAM and compute resources into their own modules using moved blocks across two separate PRs. These are the rules I follow for any Terraform refactoring that changes resource addresses.

## The rules

1. **Never change resource addresses and attributes in the same commit.** A moved block tells Terraform the resource at the old address is the same resource at the new address. If you also change an attribute during the move, Terraform sees a diff on the new address and may force a replacement. Refactor first, modify second, in separate commits.

2. **Never validate moved blocks against empty state.** A plan on empty state will always say "0 to add, 0 to change, 0 to destroy" because there are no resources at the old addresses to evaluate. That green plan proves nothing. See [Why a Clean Plan on Empty State Does Not Validate a Moved Block Refactor](/notes/empty-state-plans) for the full explanation.

3. **Some attributes are immutable.** `aws_iam_policy` description, for example, forces replacement if changed. If you edit an immutable attribute during extraction, the moved block is defeated. Know which attributes are immutable before you start.

## The sequence

### 1. Write the destination module

Create the new module with variables, resources, and outputs. Every resource attribute must be byte-for-byte identical to what the current code produces. Do not rename anything yet.

If the original resource looks like this:

```hcl
resource "aws_iam_role" "ec2_ssm" {
  name                 = "platform10-dev-ec2-ssm-role"
  assume_role_policy   = data.aws_iam_policy_document.ec2_assume_role.json
  permissions_boundary = aws_iam_policy.permission_boundary.arn
  max_session_duration = 3600

  tags = {
    Name = "platform10-dev-ec2-ssm-role"
  }
}
```

The resource in the destination module must produce the exact same values. A single character difference is a diff. A diff on an immutable attribute is a forced replacement.

### 2. Write the moved blocks

One per resource. Each maps the old address to the new address. Place them in the environment directory alongside the module call.

```hcl
moved {
  from = aws_iam_role.ec2_ssm
  to   = module.iam.aws_iam_role.ec2_ssm
}
moved {
  from = aws_iam_policy.permission_boundary
  to   = module.iam.aws_iam_policy.permission_boundary
}
moved {
  from = aws_iam_instance_profile.ec2_ssm
  to   = module.iam.aws_iam_instance_profile.ec2_ssm
}
```

### 3. Apply the original code to bring infrastructure up

If you run apply/destroy cycles like I do, the infrastructure must be live before you can test the moved blocks.

```bash
terraform plan -var-file=dev.tfvars -out=tfplan.binary
terraform apply "tfplan.binary"
```

### 4. Switch to the refactored code and plan against live state

Checkout the branch with the moved blocks and run a plan. This is the real test.

```bash
git checkout feat/extract-iam-module
terraform plan -var-file=dev.tfvars -out=tfplan.binary
```

The plan must show "has moved to" lines and zero destroys:
```
aws_iam_role.ec2_ssm has moved to module.iam.aws_iam_role.ec2_ssm
aws_iam_policy.permission_boundary has moved to module.iam.aws_iam_policy.permission_boundary
aws_iam_instance_profile.ec2_ssm has moved to module.iam.aws_iam_instance_profile.ec2_ssm

Plan: 0 to add, 0 to change, 0 to destroy.
```

If the plan shows any destroy or recreate, an attribute does not match between old and new. Fix it before proceeding.

### 5. Apply the refactored code

```bash
terraform apply "tfplan.binary"
```

Terraform updates state pointers only. No infrastructure is created or destroyed.

### 6. Modify attributes in a follow-up commit

Now that extraction is complete and state is migrated, make any improvements as separate changes. For example, updating a description or adding a new tag. Each change is its own commit with its own plan.

### 7. Remove moved blocks in a later commit

They are safe to delete once the migrating apply has completed. They do nothing after state has been updated.

```bash
rm environments/dev/compute-moved.tf
git add -A
git commit -m "chore: remove stale moved blocks after state migration"
```

## Common failures

- **Phantom diffs from provider read-back.** The provider may read back a different value than what your code says, creating a diff that forces replacement. See [What the AWS Provider Actually Tracks With associate_public_ip_address](/notes/nat-eip-drift) for an example.

- **Checkov skips on the wrong block.** If you add inline skips during extraction, place them inside the block where the finding is raised, not above it. Misplaced skips are silently ignored.

- **Deleting moved blocks too early.** If you remove a moved block before the migrating apply, Terraform loses the address mapping and falls back to destroy and recreate.