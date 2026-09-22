---
title: "Building OIDC Federation for CI from Scratch: What Broke and What I Learned"
description: "Replacing static IAM keys with OIDC federation in a Terraform CI pipeline. 3 Failures And The 2 Identity Problem"
pubDate: 2026-09-22
draft: false
---

I am building a portfolio infrastructure project on AWS using Terraform. The CI pipeline runs on GitHub Actions with two workflows: a plan workflow that triggers on pull requests, and a gated apply workflow that triggers by manual dispatch with an environment approval step. Both workflows authenticate to AWS using static IAM access keys stored in GitHub Secrets. This article covers replacing that authentication method with OIDC federation.

## The problem with static keys

Static keys are standing credentials, which could potentially be exposed. They work from any IP and machine, and never expire unless I rotate them. Although the keys were encrypted at rest in GitHub Secrets, that only protects them from being read through the GitHub UI. The keys were still vulnerable to a compromised runner, a malicious action dependency, or a log spill that accidentally prints environment variables. The consequence would be persistent access to my AWS account. I moved to OIDC federation to remove this risk entirely. There is nothing to leak.

## What OIDC federation does

OIDC federation replaces static keys with short-lived STS credentials generated per workflow run. Each time a workflow job runs, GitHub issues a signed JWT token. The workflow presents that token to AWS STS, which validates it against the conditions in an IAM role's trust policy. If the conditions match, STS returns temporary credentials that expire in minutes. Nothing is stored in GitHub Secrets. Nothing needs rotation.

I wrote a detailed breakdown of the four-stage token flow in a separate note: [How OIDC Federation Works Between GitHub Actions and AWS](/notes/oidc-federation). This article focuses on what happened when I built it and what I learned from the failures.

## What I built

The implementation lives in a single file, `modules/iam/oidc.tf`, and consists of four resources. All four are free AWS resources. The security improvement costs nothing on the invoice.

An `aws_iam_openid_connect_provider` that registers GitHub's token service as a trusted identity issuer in the AWS account. Without this, STS does not recognize tokens from GitHub and rejects every request.

An `aws_iam_role` with a trust policy that controls who can assume the role. The trust policy checks two conditions on the JWT: the `aud` claim must be `sts.amazonaws.com`, and the `sub` claim must match one of three allowed values (main branch push, pull request, or the `production-apply` environment context).

An `aws_iam_policy` containing a least-privilege permission policy that lists the exact AWS actions Terraform needs for the current 36-resource stack: EC2 and VPC management, IAM management, S3 access scoped to the state bucket, SSM parameter reads for AMI lookups, and STS caller identity checks.

An `aws_iam_role_policy_attachment` that connects the permission policy to the role. The trust policy controls WHO. The permission policy controls WHAT. The permission boundary (applied to every principal in the account) sets the ceiling on WHAT, regardless of what the permission policy allows.

## What broke and what I learned

The implementation took three rounds of CI failures before all workflow paths passed.

### Failure 1: missing SSM permission from least-privilege permission policy

The first CI run failed at the Terraform plan step. The error was `AccessDeniedException: User is not authorized to perform ssm:GetParameter`. The compute module uses a data source that looks up the latest Amazon Linux 2023 AMI via an SSM public parameter. My permission policy did not include `ssm:GetParameter` because I had focused on the resources Terraform creates and destroys, not the data sources it reads during planning.

The local plan passed because my static IAM access keys `cj-admin` has broader permissions. The gap only surfaced when the OIDC role, with its self-written scoped-down permission policy, ran the same code. The fix was adding `ssm:GetParameter` and `ssm:GetParameters` to the policy, applying the update locally, and pushing again.

The lesson: least-privilege policies cannot be written accurately in one pass by reading your own code. Data sources, provider internals, and backend operations all make API calls that are not visible in the resource definitions. The correct workflow is to write your best guess, run it in CI, read the existing errors, add the missing permission, and repeat.

### Failure 2: environment sub claim

After fixing the SSM permission, both the plan workflow (on PRs) and the plan on push to main passed. The apply workflow failed. The error was `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`.

The apply workflow uses `environment: production-apply` in the job definition for the approval gate. When a GitHub Actions job runs in a named environment, GitHub replaces the ref-based sub claim with an environment-based one. Instead of `repo:taichenjie/platform10:ref:refs/heads/main`, the token's sub claim becomes `repo:taichenjie/platform10:environment:production-apply`. That value was not in my trust policy's allowed list, so STS rejected the token.

The fix was adding a third entry to the allowed sub claims. But the deeper lesson is that the sub claim is not a fixed value. It changes based on how the workflow is triggered and whether the job uses a named environment. Most OIDC setup guides show a single ref-based claim and stop there. If your pipeline uses GitHub environments for approval gates, you need to account for this.

### Failure 3: Checkov skip placement

The second CI run also failed, but for a different reason. Checkov flagged five findings on the permission policy document (privilege escalation, data exfiltration, write access without constraints). These are expected findings for a policy that grants IAM and EC2 management actions with `resources = ["*"]`. Most of these actions do not support resource-level restrictions.

I added five inline skips with rationale comments. The next run failed on the same findings. The lesson: this was a simple yet repeatable syntax error. The skips were placed above the `data` block as standalone comments, not inside it. The correct method is to write them inside the block. Checkov evaluates blocks independently.

## The two-identity problem

After OIDC migration, local and CI use different identities:

| | Local | CI |
|---|---|---|
| Identity | `cj-admin` (static keys) | `platform10-github-actions-ci` (OIDC) |
| Permissions | Broad | Scoped to 36-resource stack |
| Credential lifetime | Permanent until rotated | Minutes |

A plan that passes locally can fail in CI for permission reasons. This is a feature, not a bug. CI runs the same code against the same state but with the actual scoped credentials. It is the real validation of whether the permission policy is complete. Local is convenient for fast feedback, not a guarantee.

This means every new resource type added in future months (K3s in Q3, for example) will require a deliberate update to the OIDC permission policy. If I add an ELB resource and forget to add `elasticloadbalancing:*` permissions, the local plan will pass and the CI plan will fail.

## How engineers write this in production

Most teams do not write the OIDC provider, trust policy, and permission policy from scratch. The common approach is to use a pre-built Terraform module like `terraform-aws-modules/iam` which abstracts the provider registration and role creation into a few input variables. The permission policy is generated after the workload has been running, using IAM Access Analyzer. Access Analyzer reads CloudTrail logs of actual API calls and produces a policy containing only the actions that were used. This is more accurate than manual enumeration because it captures data source calls, provider internals, and backend operations that are easy to miss.

Before deploying a new policy, engineers use IAM Policy Simulator to test whether specific actions would be allowed or denied under the proposed policy, without making real API calls.

I wrote everything from scratch because understanding what each resource does, why each permission is needed, and where each failure comes from was the objective. In a team setting, the pre-built module saves time and reduces human error. But the engineer configuring it still needs to understand the trust policy conditions (which sub claims to allow), the permission scoping (what the workload actually calls), and the failure mode (CI failing on a missing permission is the system working correctly, not a bug to work around).

## What production would look like

In a production system with multiple teams and repositories:

The OIDC provider would be shared across repositories, but each repo would have its own IAM role with its own trust policy pinned to that repo's sub claims.

The trust policy would pin sub claims to specific workflow filenames using `job_workflow_ref` in addition to repo and branch. This prevents a renamed or modified workflow from assuming the role without review.

A break-glass path would exist: a separate IAM role with broader permissions, assumable only by named humans via MFA, for incident response when the OIDC path is broken or insufficient.

Static keys for local development would be replaced with `aws sso login` or similar federated access, eliminating long-lived keys from developer machines and closing the last static-credential gap.

## The mental model shift

Static keys mean I am responsible for protecting a secret that never expires. Rotation, access audits, leak detection, revocation. All of those are processes that have to be maintained and can fail silently.

OIDC removes the secret entirely. Each workflow run proves its identity, gets scoped credentials for that run, and the credentials expire on their own. There is nothing to rotate and nothing to revoke. The security work moves from ongoing operational habits to a one-time architectural decision, which is easier to get right and harder to quietly break.
