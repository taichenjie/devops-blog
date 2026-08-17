---
title: "What the AWS Provider Actually Tracks With associate_public_ip_address"
description: "A phantom diff that forces NAT instance replacement on every plan, and why the fix is removing the attribute entirely."
pubDate: 2026-08-17
draft: false
---

I am building a portfolio infrastructure project on AWS using Terraform. The network is a standard VPC with public and private subnets across two availability zones. Private subnets reach the internet through a self-managed NAT instance (a t4g.nano running iptables MASQUERADE) instead of a managed NAT Gateway. The NAT instance sits in a public subnet with an Elastic IP attached for a stable outbound address. I run apply/destroy cycles between work sessions to keep costs near zero.

## The unexpected forced replacement

During a module extraction, I ran `terraform plan` against live state and got a forced replacement on the NAT instance. This confused me because I had not changed anything in the resource configuration, and did not expect a destroy on my NAT instance.

Upon reading the `terraform plan` output, I found out that the problem was `associate_public_ip_address = false` in my `aws_instance` resource. I had set it to false because I did not want the instance to get an auto-assigned public IP as I already had an Elastic IP attached to it.

However, the AWS provider does not distinguish between an auto-assigned public IP and an EIP-assigned public IP. When it reads the instance back from the API, it sees a public IP exists and returns `associate_public_ip_address = true`. My code says `false`, and the provider sees a diff. And because `associate_public_ip_address` is a ForceNew attribute, the diff is not an in-place update. It is a destroy and recreate.

## Why this only surfaced now

This was latent in my configuration since the initial build. It only surfaced when I planned against live state during the extraction. Every previous plan had been against empty state (destroy between sessions), so there was no running instance for the provider to read back from.

## The fix

The fix is to remove the attribute entirely. The subnet already has `map_public_ip_on_launch = false`, which controls whether instances automatically get a public IP. That attribute is enough and the correct way to prevent auto-assignment. Setting it again on the instance is redundant, and when an EIP is involved, it creates the phantom diff that forces replacement on every apply, which is what we want to avoid. The NAT instance is the single outbound path for everything in the private subnets. A forced replacement tears that path down and rebuilds it, which means new instance ID, new network interface, and a window where private subnet traffic has no route to the internet. In a production environment, that is an outage. In my case, it meant I could not get a clean plan during the module extraction, which defeated the whole purpose of using moved blocks to prove a non-destructive refactor.

## What I took from this

The lesson I took from this: `terraform plan` output reflects what the provider reads back from the AWS API, not what is written in code. When those two disagree, you get diffs on attributes you never touched. In such cases, always read the diff output, check what the provider actually returns for that attribute, and work backwards from there.
