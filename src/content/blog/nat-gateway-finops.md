---
title: "Replacing the $32/Month NAT Gateway with a $4 NAT Instance"
description: "I needed private subnet egress on AWS. The managed NAT Gateway costs $32/month before any traffic. I built a self-managed NAT instance for $4 instead, and documented exactly what I gave up."
pubDate: 2026-08-14
draft: true
---

I needed private subnet internet access. Every instance in my VPC sits in a private subnet with no public IP, which is the security design of the architecture. Nothing is publicly addressable. But private instances still need outbound access for package updates, SSM agent registration, and pulling container images.

AWS offers a managed NAT Gateway for this. In ap-southeast-1, it costs $0.059 per hour plus $0.059 per GB of data processed. That is about $43 per month just for the gateway to exist, before a single byte of real traffic moves through it. US East is cheaper at $0.045/hr, which comes to about $32/month. Either way, this is the single most expensive line item for a simple private network.

AWS recommends the managed NAT Gateway for production use, and for good reason. It is fully managed, scales automatically, and requires no maintenance. But I am running this project on a hard budget constraint of under $10 per month on standard AWS pay-as-you-go rates, no free tier, no credits. A $32-43 NAT Gateway blows that immediately. So I built a self-managed NAT instance instead.

## What I built

The NAT runs on a t4g.nano. At ap-southeast-1 on-demand rates, that is $3.87/month for compute and $0.77/month for 8GB of gp3 EBS. Under $5 total, compared to $43 for the managed gateway.

The instance sits in a public subnet with an Elastic IP. Private subnets route their 0.0.0.0/0 traffic to the NAT instance's primary network interface. The NAT rewrites the source IP of outgoing packets to its own, forwards them to the internet gateway, and tracks the connections so return traffic gets routed back.

Three things make a plain EC2 instance work as a NAT.

**source_dest_check = false.** By default, AWS drops any packet whose source or destination IP does not match the instance's own IP. This is a safety feature, but a NAT forwards packets belonging to other hosts by definition. I needed to turn this check off to make the instance usable as a NAT. Without it, every forwarded packet gets silently dropped.

```hcl
resource "aws_instance" "nat" {
  ami               = data.aws_ssm_parameter.al2023_arm64.value
  instance_type     = "t4g.nano"
  source_dest_check = false
  # ...
}
```

**Kernel IP forwarding.** The Linux kernel does not forward packets between interfaces by default. The user data script sets net.ipv4.ip_forward = 1 via a drop-in file in /etc/sysctl.d so the setting survives reboot.

**iptables MASQUERADE.** A MASQUERADE rule on the primary interface rewrites the source IP of outbound VPC traffic to the NAT instance's own private IP. Without this, the internet gateway would not know where to send return traffic. The rule is scoped to the VPC CIDR so only VPC traffic gets rewritten.

```bash
iptables -t nat -A POSTROUTING -s "${vpc_cidr}" -o "$PRIMARY_INTERFACE" -j MASQUERADE
```

The user data script handles all three. It gives instructions to the NAT instance on how to behave. It runs on first boot with set -euo pipefail so any failure stops the script, logs to /var/log/user-data.log for debugging, and detects the primary network interface from the default route rather than hardcoding ens5.

## The routing

Two route tables connect everything.

The public route table sends 0.0.0.0/0 to the internet gateway. Both public subnets are associated with it.

The private route table sends 0.0.0.0/0 to the NAT instance's primary ENI. Both private subnets are associated with it. I route to the ENI rather than the instance ID because routing via the ENI survives instance replacement cleanly.

```hcl
resource "aws_route" "private_default" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  network_interface_id   = var.nat_network_interface_id
}
```

The full outbound path for a packet leaving a private instance goes through the private subnet, private route table, NAT ENI, kernel MASQUERADE, public route table, internet gateway (rewrites source to the EIP), then out to the internet. Return traffic follows the reverse path via connection tracking. No explicit inbound route is needed.

## Verifying it works

I applied the full stack (31 resources, about 3 minutes) and connected to the NAT instance via SSM Session Manager. No SSH, no bastion, no public IP on the instance I connected to.

Inside the NAT, I checked the following.

cloud-init status showed done. The user data log confirmed "NAT setup complete."

net.ipv4.ip_forward was 1.

The iptables MASQUERADE rule was present. 238 packets and 18,819 bytes had already been forwarded during cloud-init's own dnf update calls. The NAT was working before I even looked at it.

curl ifconfig.me returned the Elastic IP address, confirming outbound traffic was exiting through the internet gateway.

A ping to the SSM endpoint returned in 26ms, which means the request went to the VPC interface endpoint via private DNS, not out to the public internet.

The SSM session connecting at all is its own proof. It means the IAM role, permission boundary, instance profile, SSM endpoints, endpoint security group, and SSM agent registration are all correct at the same time. If any one of those is wrong, the session fails.

After verification, I destroyed everything. 31 resources destroyed, zero orphans on the post-destroy audit. The EIP and NAT instance were both confirmed gone. This matters because an unattached EIP costs about $0.005/hour, so a forgotten one is a billing leak I am trying to avoid.

## What I gave up

A single NAT instance is a single point of failure. It runs in one availability zone. If the instance dies or the AZ has an outage, all private subnet egress stops until the instance is replaced.

The managed NAT Gateway is highly available and does not have this problem. AWS runs it across multiple AZs with automatic failover.

I documented this trade-off in ADR-001. I am trading high availability for cost savings, and I know exactly what the production path looks like if I needed to undo it.

There are three options to make this highly available.

Run two NAT instances, one per AZ, each in an Auto Scaling Group of size 1 with a launch template. If an instance dies, the ASG replaces it automatically.

Switch to [fck-nat](https://fck-nat.dev/), a maintained NAT AMI with HA mode and a Terraform module. This is the correct production choice and I evaluated it before deciding on the self-managed approach.

Accept the cost and use the managed NAT Gateway.

I chose the self-managed instance specifically to understand the underlying mechanics. fck-nat abstracts the same work I did by hand, the source_dest_check, the IP forwarding, the MASQUERADE rule. As a learning objective, I chose to build my own self-managed NAT instance.

## The actual cost

Infracost reports the worst-case 24/7 cost for the entire Q1 network at $52.09/month. The NAT instance is about $4.64 of that. The five interface VPC endpoints for SSM and ECR dominate at about $47/month.

With apply/destroy discipline (spin up, verify, tear down), a single cycle costs about $0.02 and takes 20-25 minutes. My real monthly spend is well under $2.

The Infracost baseline is committed to the repo at docs/cost/m1-baseline.json, and every pull request shows the cost delta against that baseline before any change is applied. This is not a report I generate after the fact. It is a gate in the CI pipeline.

---

*All infrastructure is Terraform, applied and verified against real AWS, then destroyed. Source code at [platform10](https://github.com/taichenjie/platform10). NAT trade-off documented in ADR-001.*
