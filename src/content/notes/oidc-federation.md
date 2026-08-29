---
title: "How OIDC Federation Works Between GitHub Actions and AWS"
description: "The full token flow from GitHub to STS, why each stage exists, and why you need more sub claims than most guides show."
pubDate: 2026-08-27
draft: false
---

I am building a portfolio infrastructure project on AWS using Terraform. The CI pipeline authenticates to AWS via OIDC federation instead of static IAM access keys. This note documents how the token flow works and what I learned setting it up.

## Why OIDC over static keys

Static IAM keys stored in GitHub Secrets are standing credentials. They work from any IP, any machine, and never expire unless manually rotated. A single leak gives an attacker persistent access. OIDC eliminates the stored secret entirely. Each workflow run gets short-lived credentials that expire in minutes and are bound to a specific repo, branch, and workflow context.

## The token flow in four stages

**Stage 1: GitHub issues a JWT.** When a workflow job has `id-token: write` permission, the GitHub Actions runner requests a token from GitHub's OIDC endpoint. GitHub signs the token cryptographically and includes three key claims: `iss` (issuer, always `https://token.actions.githubusercontent.com`), `sub` (subject, identifies the repo, branch, and trigger context), and `aud` (audience, set to `sts.amazonaws.com` for AWS).

**Stage 2: The runner presents the token to AWS STS.** The `configure-aws-credentials` action calls `sts:AssumeRoleWithWebIdentity`, passing the JWT and the ARN of the IAM role to assume. The token is proof of identity, not authorization. STS is the bridge that converts a verified identity into usable AWS credentials.

**Stage 3: STS validates the token.** STS checks four things in order. Is the issuer a registered OIDC provider in this AWS account? Is the JWT signature valid (verified against GitHub's public signing keys fetched via OIDC discovery)? Do the token's claims match the conditions in the IAM role's trust policy (audience and subject)? Is the token still within its expiry window (GitHub tokens last about five minutes)? If any check fails, the request is rejected.

**Stage 4: STS issues temporary credentials.** All checks pass. STS returns an access key ID, secret access key, and session token. These are injected into the runner's environment variables. Every subsequent step (terraform plan, terraform apply) uses them transparently. When they expire, they become invalid. There are no secrets to be stored or rotated.

## What the Terraform code builds

The implementation lives in `modules/iam/oidc.tf` and creates four resources:

1. `aws_iam_openid_connect_provider` registers GitHub as a trusted identity issuer. Without this, STS does not know what `token.actions.githubusercontent.com` is and rejects every request immediately.

2. `aws_iam_role` is the role the pipeline assumes. Its trust policy (the `assume_role_policy` argument) defines the conditions STS checks in Stage 3. Its `permissions_boundary` caps the maximum permissions the role can ever have.

3. `aws_iam_policy` wraps the permission policy document into a managed policy object. A policy document alone is just JSON. It needs to be wrapped in a policy resource before it can be attached.

4. `aws_iam_role_policy_attachment` connects the permission policy to the role, scoping what the role can actually do once assumed.

The trust policy controls WHO can assume the role. The permission policy controls WHAT the role can do. The permission boundary sets the ceiling on WHAT, regardless of what the permission policy allows.

## The sub claim is not one value

Most OIDC guides pin the trust policy to a single sub claim like `repo:owner/repo:ref:refs/heads/main`. This works if your pipeline only triggers on push to main. Mine has three trigger contexts, and each produces a different sub claim:

| Workflow trigger | Sub claim produced |
|---|---|
| Push to main | `repo:taichenjie/platform10:ref:refs/heads/main` |
| Pull request targeting main | `repo:taichenjie/platform10:pull_request` |
| Manual dispatch with `environment: production-apply` | `repo:taichenjie/platform10:environment:production-apply` |

I discovered the second and third values through failures. The PR plan workflow failed because `pull_request` did not match the ref-based entry. After fixing that, the gated apply workflow failed because GitHub Actions replaces the ref-based sub claim entirely when a job uses a named environment. Both failures returned the same error: `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`. The error does not tell you which claim failed. You have to reason about what sub claim the trigger context would produce and check it against your trust policy.

## What I took from this

The sub claim changes based on how the workflow is triggered and whether the job uses a named environment. If your pipeline has PR checks, branch pushes, and environment-gated deploys, you need a separate trust policy entry for each context. The OIDC flow itself is well documented, but the sub claim behavior across different GitHub Actions trigger types is not. Test every workflow path, not just the main branch push.
