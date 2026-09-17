---
name: pkaranth-hub-branch-lab
description: Provision the pkaranth hub/branch SD-WAN test lab (dedicated VPC, public+private subnets, hub/branch instances with dual ENIs and EIPs, VyOS client routers, new SSH key) in a given AWS region. Use when the user asks to stand up "the hub/branch lab", "pkaranth-hub and pkaranth-branch", or references this skill by name.
---

# pkaranth hub/branch lab

Builds a self-contained SD-WAN test topology: one hub site and one branch site,
each with a public-facing gateway instance (dual-homed, public+private ENI) and
a private-side VyOS "client" router, all inside one new dedicated VPC.

## Variables

Ask the user for, before doing anything else:

1. **AWS region** (e.g. `ap-south-1` for Mumbai). This is the only required input.

Derive everything else from the region unless the user overrides it:

- **Availability zone**: `<region>a` (first AZ in the region), used for every subnet/instance.
- **VPC CIDR**: `10.0.0.0/16`
- **Subnet CIDRs**: `pkaranth-public` = `10.0.1.0/24`, `pkaranth-hub-private` = `10.0.2.0/24`, `pkaranth-branch-private` = `10.0.3.0/24`
- **Key pair name**: `pkaranth-hub-branch-<region>-key`

If the user gives their own CIDRs/AZ/key name, use those instead — don't ask unless something conflicts with existing resources.

## Provisioning method (important)

Use **direct AWS calls via the AWS MCP sandbox** (`mcp__aws-mcp__aws___run_script`, boto3), **not Terraform**, to actually create these resources — this is a standing preference for this repo (see memory `feedback-aws-provisioning-method`), even though the repo otherwise consists of Terraform projects.

Before creating anything:
- Verify the target region doesn't already have a VPC/key pair named the same thing (avoid collisions).
- Verify both AMIs are available/subscribed in the target region (`DescribeImages`) — `ami-01b34066afaff1f73` (hub/branch gateway image) and `ami-08c3193580309607f` (VyOS router image) are region-specific and may need a marketplace subscription per account/region.
- Confirm with the user before the actual mutating/costly calls (2×`c5.xlarge` + 2×`t3.small` + 2 Elastic IPs running continuously is real, ongoing spend), even though this skill's whole point is to build the thing — a one-line "about to create X in region Y, proceed?" is enough, don't re-litigate the design.

## Build order

1. **VPC** `pkaranth-vpc` — the CIDR above, DNS support + DNS hostnames enabled.
2. **Subnets** (all in the derived AZ):
   - `pkaranth-public` — public CIDR, `MapPublicIpOnLaunch=true`
   - `pkaranth-hub-private` — hub-private CIDR
   - `pkaranth-branch-private` — branch-private CIDR
3. **Internet gateway** — create, attach to `pkaranth-vpc`. Create a route table with `0.0.0.0/0 -> igw`, associate it with `pkaranth-public`. Private subnets stay on the main route table (local-only, no NAT — nothing in this spec asks for internet egress from the private side).
4. **Security group `pkaranth-public-sg`** (in `pkaranth-vpc`), attached to interfaces in `pkaranth-public`:
   - ingress: udp/4500 from 0.0.0.0/0, tcp/443 from 0.0.0.0/0, udp/443 from 0.0.0.0/0, tcp/80 from 0.0.0.0/0, tcp/22 from 0.0.0.0/0
   - egress: all
5. **Security group `pkaranth-private-sg`** (in `pkaranth-vpc`), attached to interfaces in both private subnets:
   - ingress: tcp/22 from 0.0.0.0/0, icmp (all types) from 0.0.0.0/0, tcp/179 (BGP) from 0.0.0.0/0
   - egress: all
6. **Key pair** — create new, named per the derived key name. Save the returned private key material to a local `.pem` file in the current project directory and `chmod 400` it. This one key is used for all 4 instances.
7. **Instance `pkaranth-hub`** — AMI `ami-01b34066afaff1f73`, `c5.xlarge`, the key pair above:
   - eth0 (device index 0): `pkaranth-public`, `pkaranth-public-sg`
   - eth1 (device index 1): `pkaranth-hub-private`, `pkaranth-private-sg`, `source_dest_check=false`
   - Allocate an Elastic IP and associate it to eth0.
8. **Instance `pkaranth-branch`** — same as hub but eth1 on `pkaranth-branch-private`; its own Elastic IP on eth0.
9. **Instance `pkaranth-hub-client`** — AMI `ami-08c3193580309607f` (VyOS), `t3.small`, single interface on `pkaranth-hub-private` with `pkaranth-private-sg`. No EIP.
10. **Instance `pkaranth-branch-client`** — same as above but on `pkaranth-branch-private`.
11. Wait for all 4 instances to reach `running`, then collect their public/private IPs.
12. **Write a descriptive Terraform file** (`main.tf`/`variables.tf`/`network.tf`/`instances.tf`, matching the style already used elsewhere in this repo, e.g. `bgptracking/`) capturing the actual resource IDs/CIDRs/names created above, as a record of the environment — this file documents the build, it is not applied (the resources already exist from steps 1–11).

## Output

Report back:

1. A table of the 4 instances: **name, instance ID, public IP (if any), private IP(s)**.
2. The **key pair name** created and the local path where its private key was saved.
