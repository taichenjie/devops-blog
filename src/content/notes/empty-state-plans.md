---
title: "Why a Clean Plan on Empty State Does Not Validate a Moved Block Refactor"
description: "Moved blocks can only be verified against live state. A plan on empty state will always say zero changes, even if the moves are wrong."
pubDate: 2026-08-18
draft: false
---

I am building a portfolio infrastructure project on AWS using Terraform. The codebase is modular, with separate modules for VPC, IAM, and compute resources. I run apply/destroy cycles between work sessions to keep costs near zero, which means most of the time there is nothing live in AWS.

## The refactor

During a module extraction, I used `moved` blocks to relocate resources from one module to another. A moved block tells Terraform that a resource's address in state has changed without the resource itself changing. If the move is correct, the plan shows "has moved to" lines and zero destroys, which proves that the refactor is non-destructive.

## What I assumed

After writing the moved blocks, I ran `terraform plan`. The output said "0 to add, 0 to change, 0 to destroy." I took that as confirmation that the refactor was clean.

However, I had run the plan against empty state. Nothing was deployed, hence there were no resources at the old addresses to move, so Terraform had nothing to evaluate the moved blocks against. The plan was clean because there was nothing to plan, not because the moved blocks were verified and clean.

## What actually validates a moved block

A moved block can only be verified against live state where resources exist at the old address. When Terraform sees a resource at `module.vpc.aws_instance.nat` in state and a moved block saying that address is now `module.compute.aws_instance.nat`, it checks whether the destination configuration matches the existing resource. If it matches, the plan shows "has moved to" and zero destroys. If it does not match (because you also changed an attribute during the extraction), the plan shows a destroy and recreate.

I only got real validation after running `terraform apply` to bring the infrastructure up, then running `terraform plan` with the moved blocks in place. That second plan showed the "has moved to" lines which proved a clean refactor.

## How this works in production

In a production environment, you would not validate moved blocks by applying directly to live infrastructure. The standard practice is to run the refactored plan against a staging environment that mirrors production's state file structure and resource layout. Staging catches the address mismatches and attribute drift before the change reaches production. If staging's plan shows clean "has moved to" lines with zero destroys, then we can move on to applying changes in the production environment.

For this project, a separate staging environment is overkill and unnecessary. I have a single dev environment with 32 resources and the entire stack can be spun up in under three minutes. Applying the infrastructure, running the plan with moved blocks against it, and then verifying the output gives me the same validation loop that staging provides in a larger setup, just compressed into one environment. The cost of a full apply/plan/destroy cycle is under $0.02, so there is no reason to add the overhead of managing a second environment at this scale.

## What I took from this

When performing refactoring, a green plan is not always a meaningful plan. The result depends on what state it runs against. For any refactor that changes resource addresses (moved blocks, module extractions, renames), the only valid test is a plan against live state where those resources actually exist.
