The pkaranth-hub-branch-lab Skill
Problem it solves
Testing SD-WAN hub/branch behavior (routing, BGP, tunnel establishment, failover) needs a real, isolated network topology to poke at — not a diagram. Standing that up by hand every time is repetitive and error-prone: a fresh VPC, three subnets in the right AZ, an IGW and route tables, two security groups with specific port sets (IPsec/IKE ports, BGP, ICMP, SSH), a dedicated SSH key, and four EC2 instances wired with the right ENIs, source/dest-check settings, and Elastic IPs. Getting any one piece wrong (e.g., forgetting source_dest_check=false on the gateway's private interface) silently breaks the thing you're trying to test.

This skill packages that whole build as a repeatable, parameterized procedure so it can be reproduced in any region on demand, without re-deriving the topology from scratch each time.

The final prompt (skill body)
The skill itself is the prompt — it's not a one-liner but a full runbook, structured as:

Variables — asks only for AWS region; derives AZ, VPC/subnet CIDRs, and key pair name from it, with an escape hatch for overrides.
Provisioning method — an explicit standing instruction to use direct boto3 calls via the AWS MCP sandbox rather than Terraform, even though the rest of the repo is Terraform-based (this matches a memory I hold: feedback_aws_provisioning_method.md).
Pre-flight checks — collision checks (existing VPC/key pair names) and AMI availability checks before creating anything, plus a mandatory one-line cost confirmation before the mutating calls (2×c5.xlarge + 2×t3.small + 2 EIPs is ongoing spend).
Build order — a strict 12-step sequence: VPC → subnets → IGW/routing → two security groups → key pair → four instances (hub gateway, branch gateway, hub VyOS client, branch VyOS client) → wait for running state → write a descriptive (but not applied) Terraform file documenting what was built.
Output contract — report the 4 instances (name/ID/public IP/private IPs) and where the generated .pem key was saved.
My approach
When invoked, I treat it as a checklist to execute literally and in order, not as inspiration for my own design:

Ask for region first (the only required input) via a clarifying question — this is a costly, real-money action, so I don't guess.
Run idempotency/availability checks before touching anything mutating.
Surface the cost/consent checkpoint explicitly rather than silently proceeding, per the skill's own instruction.
Execute the AWS calls through mcp__aws-mcp__aws___run_script (boto3), not terraform apply, honoring the standing project preference even though it cuts against the repo's usual IaC pattern.
After the resources exist, write the Terraform files as as-built documentation only — never terraform apply them.
Finish with the two required deliverables: the instance table and the key-pair/key-path info.
