# AWS Networking Troubleshooting Lab

**Project:** Learn to Cloud — Phase 2 Networking Lab  
**Environment:** AWS, Linux, Terraform, AWS CLI  
**Result:** Resolved 4 of 4 simulated incidents and completed Phase 2 verification

## Overview

In this hands-on lab, I investigated and fixed four simulated networking and security incidents in a multi-tier AWS environment. Rather than rebuilding the environment, I used Linux networking tools and the AWS CLI to identify configuration problems, make targeted changes, and retest connectivity.

The environment included a public-facing web server, a private API server, a database server, and a bastion host used for SSH access. Networking components included a VPC, public/private/database subnets, route tables, an internet gateway, NAT gateway, Route 53 private hosted zones, security groups, and network ACLs (NACLs).

**Note:** This was a guided educational lab with intentionally broken infrastructure, not a production incident. The write-up describes the troubleshooting and fixes I performed. IP addresses and resource identifiers below are historical lab values; resources were destroyed after verification.

## Architecture and traffic flow

```text
My computer
    |
    | SSH (22)
    v
Bastion host (public subnet)
    | SSH (22) to internal instances
    |
Internet --> Web server (public subnet)
                 |
                 | TCP 8080
                 v
            API server (private subnet)
                 |
                 | TCP 5432
                 v
            PostgreSQL (database subnet)

API outbound HTTPS --> NAT gateway (public subnet) --> Internet gateway --> Internet
Internal hostnames --> Route 53 private hosted zone (internal.test)
```

The diagram is conceptual; the security groups and NACLs controlled which connections were actually permitted.

## INC-4521 — Private API outbound internet access

**Symptoms:** The API server could not complete an outbound HTTPS request. A connectivity test timed out despite the NAT gateway being available.

**Investigation:** I verified that the NAT gateway existed and inspected the VPC route tables using the AWS CLI. The API subnet's route table had its local VPC route but lacked a default route for internet-bound traffic.

**Fix:** Added a default IPv4 route (`0.0.0.0/0`) in the API/private subnet's route table, targeting the NAT gateway.

```bash
aws ec2 create-route \
  --route-table-id <private-route-table-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id <nat-gateway-id>
```

**Validation:** Re-ran an outbound HTTPS test from the API instance; it returned HTTP `200`. Confirmed the request used the NAT gateway's public egress IP.

**What I learned:** A route table chooses where traffic goes; a `0.0.0.0/0` route is not an authorization rule. A NAT gateway allows instances without public IPs to initiate outbound internet connections while remaining unreachable by unsolicited inbound internet connections.

## INC-4522 — Missing private DNS records

**Symptoms:** Internal service names under `internal.test` did not resolve to the expected private IP addresses.

**Investigation:** Used Route 53 CLI commands to inspect private hosted zones, their VPC associations, and record sets. The lab VPC was associated with a private hosted zone, but the necessary service A records were absent.

**Fix:** Created/updated three private IPv4 A records using a Route 53 change batch with `UPSERT`:

| DNS name | Lab private IPv4 |
|---|---|
| `web.internal.test` | `10.0.1.68` |
| `api.internal.test` | `10.0.2.211` |
| `db.internal.test` | `10.0.3.218` |

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <private-hosted-zone-id> \
  --change-batch file://dns-records.json
```

**Validation:** Queried AWS's VPC DNS resolver from an EC2 instance:

```bash
dig +short @169.254.169.253 api.internal.test A
# Result: 10.0.2.211
```

The lab validator subsequently confirmed the service names resolved correctly using cloud and system DNS on the checked VMs.

**What I learned:** An A record maps a hostname to an IPv4 address. A correctly associated private hosted zone still needs the records for applications to resolve internal services.

## INC-4523 — Web-to-API and API-to-database failures

**Symptoms:** The web server timed out when connecting to the API on TCP `8080`; the API could not confirm PostgreSQL readiness on TCP `5432`.

**Investigation and fix, part A — Web → API:**

1. From the web instance, reproduced the TCP timeout with `curl` against `http://10.0.2.211:8080/health`.
2. Verified the API service was running locally.
3. Inspected the API security group's inbound permissions and found TCP `8080` missing.
4. Allowed inbound TCP `8080` on the API security group **from the web security group**.
5. The request still timed out. Checked the web security group's outbound rules and found only HTTP/HTTPS ports allowed.
6. Allowed outbound TCP `8080` **to the API security group**.

**Result:** The web instance successfully reached the API health endpoint with HTTP `200` and a healthy status response.

**Investigation and fix, part B — API → PostgreSQL:**

1. Reproduced the failure from the API instance using `pg_isready`.
2. Identified missing API security group egress permission for TCP `5432` and added egress to the database security group.
3. PostgreSQL was still unreachable. Inspected the database subnet's NACL.
4. Found inbound NACL rule `100` explicitly denying TCP `5432`, ahead of a broader allow rule `200`.
5. Replaced rule `100` with an inbound allow rule for TCP `5432` from the API subnet (`10.0.2.0/24`). The NACL already permitted outbound return traffic.

```bash
# Representative connectivity checks, run on the respective EC2 instances
curl --noproxy '*' -v --max-time 5 http://10.0.2.211:8080/health
pg_isready -h 10.0.3.218 -p 5432 -U labuser -d labdb -t 3
```

**Validation:** Web → API returned HTTP `200`. API → Database returned `accepting connections`. The lab validator marked INC-4523 resolved.

**What I learned:** Both the source's outbound and destination's inbound security group rules must allow a new connection. Security groups are stateful and automatically allow response traffic. NACLs operate at the subnet boundary, evaluate rules in order, support explicit denies, and are stateless, so both directions matter.

## INC-4524 — Least-privilege security hardening

**Symptoms:** The security audit identified overly permissive inbound rules: SSH access from `0.0.0.0/0` on the bastion, web, API, and database hosts; PostgreSQL TCP `5432` from `0.0.0.0/0`; and unrestricted ICMP to the web instance.

**Fixes:**

- **Database access:** Added an inbound TCP `5432` rule referencing the API security group, then removed the `0.0.0.0/0` rule. Confirmed PostgreSQL still accepted connections from the API.
- **Bastion SSH:** Restricted TCP `22` to my current public IPv4 address with a `/32` CIDR. Tested a fresh connection before and after removing unrestricted access.
- **Internal SSH:** Restricted inbound TCP `22` on the web, API, and database security groups to the bastion security group. Added the replacement rules before revoking the broad ones, then tested SSH through the bastion.
- **Web ICMP:** Added an ICMP rule referencing the bastion security group and revoked unrestricted ICMP.

**Important troubleshooting lesson:** I briefly lost new SSH access to the bastion because I confused the bastion's destination public IP with my computer's public source IP. I recovered by using `curl -4 https://api.ipify.org` from my local machine, authorizing the correct `/32` source in the bastion security group via the AWS CLI, verifying a fresh SSH connection, and removing the incorrect rule.

**Validation:** Final lab checks confirmed the expected allowed/denied network traffic and least-privilege SSH sources. INC-4524 passed.

**What I learned:** Use security group references where possible instead of broad subnet or internet CIDRs. To avoid outages or lockouts, add the replacement permission, test, remove the old permission, and retest. Public web ports `80` and `443` remained intentionally reachable; not every `0.0.0.0/0` rule is inappropriate.

## Commands and tools used

| Tool | Purpose |
|---|---|
| `aws ec2 describe-route-tables` | Inspect subnet routes and NAT targets |
| `aws ec2 describe-nat-gateways` | Check NAT gateway availability |
| `aws route53 list-hosted-zones`, `get-hosted-zone`, `list-resource-record-sets` | Investigate private DNS configuration |
| `aws ec2 describe-security-groups` | Inspect inbound and outbound rules |
| `aws ec2 authorize-security-group-ingress/egress` | Add narrowly scoped security group rules |
| `aws ec2 revoke-security-group-ingress` | Remove overly broad inbound access |
| `aws ec2 describe-network-acls`, `replace-network-acl-entry` | Investigate and correct subnet ACL rules |
| `curl`, `dig`, `pg_isready`, `ssh`, `ip route` | Reproduce failures and verify connectivity |
| `terraform plan`, `terraform state list`, lab cleanup script | Inspect managed infrastructure and destroy lab resources |

## Results and takeaways

- Resolved **4 of 4 simulated AWS networking and security incidents**.
- Confirmed web-to-API HTTP health and API-to-PostgreSQL connectivity.
- Configured private DNS resolution and restored NAT-based outbound HTTPS.
- Replaced broadly scoped administrative and database access with least-privilege rules.
- Passed Learn to Cloud Phase 2 verification.
- Ran the supplied cleanup workflow successfully (`Destroy complete! Resources: 18 destroyed`), including the current deployment's NAT gateway, hosted zone, and VPC.

The lab reinforced a repeatable process: **reproduce the problem → identify the traffic path → inspect routing and security controls → make a focused change → retest → validate**.

### Cleanup note

An interrupted earlier deployment left an AWS NAT gateway outside Terraform's recorded state; I investigated and manually deleted it before completing that deployment's cleanup. After the successful final lab, the supplied destroy script removed the current deployment's 18 managed resources. Because older or manually created resources may not be tracked by the current Terraform state, I would also verify remaining resources and charges in the AWS console.

---

*Hands-on learning project completed as part of [Learn to Cloud](https://learntocloud.guide/).*
