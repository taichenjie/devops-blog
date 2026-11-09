---
title: "Security groups don't filter the Amazon DNS resolver"
description: "My DNS egress rules did nothing. A security group never sees traffic to the VPC resolver."
pubDate: 2026-11-09
draft: false
---

My K3s node's security group had two egress rules for DNS, one for UDP and one for TCP, both on port 53 to the VPC CIDR. I wrote them in Terraform like any other rule.

~~~hcl
resource "aws_vpc_security_group_egress_rule" "node_dns_udp" {
  security_group_id = aws_security_group.node.id
  description       = "DNS over UDP to the VPC resolver"
  cidr_ipv4         = var.vpc_cidr
  ip_protocol       = "udp"
  from_port         = 53
  to_port           = 53
}
~~~

I added them because I had locked egress down to port 443 only. I assumed DNS would break without an explicit allow.

## What the reviewer pointed out

An external reviewer on the pull request pointed out that these rules did nothing. AWS documents that security groups do not filter traffic to and from Amazon DNS, Amazon DHCP, instance metadata, ECS task metadata, Windows license activation, Amazon Time Sync, and the reserved addresses of the default VPC router.

It is not that the allow was unneeded. The security group never sees this traffic, so no rule can allow it or block it. Instances send DNS queries to the VPC resolver at the VPC base address plus two, because the VPC's DHCP options tell them to, and that traffic sits outside the security group entirely.

## How I proved it

I removed both rules and applied again. The plan went from 40 resources to 38. On the new node, `getent hosts github.com` still returned an address, and the K3s installer still downloaded from GitHub by hostname. DNS worked exactly as before.

## Why it matters

A rule that does nothing still looks like a control. Anyone reading my security group would think DNS egress was restricted to the VPC. It was not restricted at all, because it never passed through the security group.

That has a security cost. A compromised pod can send DNS queries for any domain, whatever the security group says, and DNS queries can carry data out. Security groups cannot stop that. They only have allow rules, and they match on IP addresses and ports, not domain names.

## The exception

The exemption covers only the Amazon-provided resolver. If I ran my own DNS server on an EC2 instance, traffic to it would go through security groups like any other traffic, and I would need the port 53 rule.

## What I would use instead

To control DNS, AWS offers Route 53 Resolver DNS Firewall. It filters queries by domain name at the resolver itself, with allow lists and block lists. That is the layer that can block lookups of malicious domains or stop DNS being used to exfiltrate data. Security groups were never the right tool for it.

## What I took from it

Before trusting a control, I check what it actually governs. The fastest test was removing the rule and seeing that nothing changed.

Sources:
- AWS VPC user guide, security groups: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html
- Route 53 Resolver DNS Firewall: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-dns-firewall.html
