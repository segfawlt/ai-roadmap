# AWS Core Services Quick Reference

Every other AWS service composes these four primitives. Understand them well, and the rest of AWS becomes approachable.

## IAM (Identity and Access Management)

Learn this first. Almost every AWS outage or security incident has an IAM misconfiguration somewhere in the chain.

### Core concepts

- **User** — a human or long-lived credential. Keep these rare.

- **Role** — a short-lived identity that an AWS service assumes. Prefer roles over users for everything machine-to-machine.

- **Policy** — a JSON document listing allowed actions on resources.

- **Principal** — the entity that the policy applies to.

### Least privilege policy template

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSpecificBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

### Key rules

1.  **Never attach policies directly to users** — use groups or roles.

2.  **Never use root credentials** — only for initial account setup.

3.  **Never commit access keys to Git** — use IAM Roles for EC2, ECS, Lambda; use `aws-vault` or SSO for developers.

4.  **Rotate any key that touches a repo**, even a private one.

## EC2 (Elastic Compute Cloud)

### Instance type naming

`t3.medium` → `t` (family) + `3` (generation) + `.medium` (size).

- `t` — burstable, cheap default for dev/small prod

- `m` — general purpose, balanced CPU/memory

- `c` — compute-optimized

- `r` — memory-optimized

- `i` / `d` — storage-optimized (NVMe SSD / HDD)

- `g` / `p` — GPU

- `a` — ARM (Graviton) — ~20% cheaper, often faster for web workloads

### Launch essentials

``` bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --security-group-ids sg-01234567 \
  --subnet-id subnet-01234567 \
  --iam-instance-profile Name=MyInstanceProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-1}]'
```

### Gotchas

- **Free tier is t2.micro, not t3.micro** — double-check if you’re trying to stay free.

- **Stopping an instance releases the ephemeral IP** — use an Elastic IP for stable addressing.

- **Instance Store vs EBS** — instance store disappears on stop. Use EBS for anything you want to keep.

## S3 (Simple Storage Service)

### Anatomy

- **Bucket** — global namespace, DNS-compatible name, single region.

- **Object** — a file with a key, up to 5 TB.

- **Prefix** — a path-like grouping (`logs/2026/04/`), not a directory.

- **Storage class** — Standard, Intelligent-Tiering, Glacier, etc.

### Common CLI

``` bash
# Upload
aws s3 cp file.txt s3://my-bucket/path/file.txt

# Recursive upload
aws s3 sync ./dist s3://my-bucket/static --delete

# Pre-signed URL (expires in 1 hour)
aws s3 presign s3://my-bucket/path/file.txt --expires-in 3600

# Bucket policy to enforce HTTPS
aws s3api put-bucket-policy --bucket my-bucket --policy file://deny-http.json
```

### Must-do checklist for every new bucket

1.  **Block Public Access** on (unless it’s a static site).

2.  **Default encryption** with SSE-S3 or SSE-KMS.

3.  **Versioning** for anything mutable you don’t want to lose.

4.  **Lifecycle rules** to move old objects to cheaper storage classes.

5.  **Access logging** to a separate bucket.

### Cost gotchas

- **Request costs** — millions of `LIST` calls can rival storage cost.

- **Data transfer** — egress to the internet is ~\$0.09/GB. Cross-AZ is not free either.

- **Incomplete multipart uploads** — add a lifecycle rule to abort them after 7 days, otherwise they pile up silently.

## VPC (Virtual Private Cloud)

### Component map

```
Region
└── VPC (CIDR: 10.0.0.0/16)
    ├── Public Subnet (10.0.1.0/24, AZ-a) ──── Internet Gateway (IGW)
    ├── Public Subnet (10.0.2.0/24, AZ-b)
    ├── Private Subnet (10.0.11.0/24, AZ-a) ── NAT Gateway → IGW
    └── Private Subnet (10.0.12.0/24, AZ-b)
```

### Minimum viable VPC

1.  **One VPC per environment** (dev, staging, prod).

2.  **Two AZs minimum** for any production workload.

3.  **Public subnets for load balancers only.** Your app servers and databases go in private subnets.

4.  **Security groups are stateful firewalls.** Default allows nothing inbound, everything outbound. Open only what you need.

5.  **NACLs are stateless.** Use them for a coarse second layer.

### The single most common VPC mistake

Putting your database in a **public subnet** “just for development”. Don’t. Use a bastion host or SSM Session Manager instead — no inbound ports required, audited, and free.

## Where to go next

- IAM deep dive → [AWS IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/)

- EC2 pricing model → see [Basic AWS Services Essential](../applied/aws-services.md)

- VPC patterns → [AWS Well-Architected: Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)

## Practice

All exercises assume a free-tier account. Stay in one region (e.g., `us-east-1`) to avoid cross-region confusion.

### 1. IAM least privilege

Create an IAM role named `AppReadS3` with a policy that grants **read-only** access to a single bucket `my-app-data` and its contents. Verify via the AWS CLI with `--profile`` ``AppReadS3` that:

- `aws`` ``s3`` ``ls`` ``s3://my-app-data` succeeds

- `aws`` ``s3`` ``cp`` ``test.txt`` ``s3://my-app-data/test.txt` fails with `AccessDenied`

- `aws`` ``s3`` ``ls`` ``s3://some-other-bucket` fails with `AccessDenied`

### 2. S3 bucket hardening

Create a new bucket. Enable:

- Block Public Access (all four toggles on)

- Default encryption (SSE-S3)

- Versioning

- Lifecycle rule: transition objects to `Standard-IA` after 30 days, delete non-current versions after 90 days, abort incomplete multipart uploads after 7 days

Verify each setting via `aws`` ``s3api`` ``get-*` commands.

### 3. EC2 with Instance Profile

Launch a `t3.micro` EC2 instance in a **private subnet** with:

- An IAM Instance Profile that allows read from your bucket (reuse exercise 1’s role).

- An SSH key pair (you won’t actually SSH in — we’ll use SSM).

- A security group with **no inbound rules**.

Connect via AWS Systems Manager Session Manager (`aws`` ``ssm`` ``start-session`` ``--target`` ``i-...`). From inside the instance, run `aws`` ``s3`` ``ls`` ``s3://my-app-data` — it should work without any credentials set, proving the instance profile is injected.

### 4. VPC with two AZs

Build a VPC (10.1.0.0/16) with:

- One public subnet in each of two AZs

- One private subnet in each of two AZs

- One Internet Gateway attached to the VPC

- One NAT Gateway in a public subnet with an Elastic IP

- Route tables such that private subnets egress via the NAT Gateway

Launch a `t3.micro` in a **private** subnet. Verify it can reach `curl`` ``https://api.github.com` (egress through NAT) but cannot be reached from your laptop (no inbound path).

### 5. Pre-signed URL

Upload a PDF to S3. Generate a pre-signed URL that expires in 10 minutes. Share it with a colleague (or a second browser session). Verify access works before expiry and returns `403` after expiry.

### 6. Cost cleanup

At the end of the lab, run:

``` bash
aws s3 ls
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name]'
aws ec2 describe-addresses
```

Terminate instances, release Elastic IPs, and empty+delete buckets. **Elastic IPs are free while attached to a running instance, but cost money when unattached** — a very common surprise bill.

## Review Questions

1.  **What is the recommended way to give an EC2 instance permission to read from S3?**

    - A. Copy an IAM user’s access keys onto the instance

    - B. Attach an IAM Role (via Instance Profile) to the instance

    - C. Make the bucket public

    - D. Store credentials in `/etc/environment`

2.  **A new S3 bucket should have which four things enabled by default?**

    - A. Public read, versioning, logging, lifecycle

    - B. Block Public Access, default encryption, versioning, and a lifecycle rule to abort incomplete multipart uploads

    - C. Static website hosting, CORS, public ACLs, acceleration

    - D. No defaults — configure only when you need them

3.  **What is the primary difference between a Security Group and a NACL in AWS VPC?**

    - A. Security Groups are free, NACLs cost money

    - B. Security Groups are stateful (return traffic is automatically allowed); NACLs are stateless and require explicit return rules

    - C. NACLs are stateful; Security Groups are stateless

    - D. They are identical

4.  **Where should your production database live in a VPC?**

    - A. Public subnet so developers can SSH in

    - B. Private subnet with no direct internet path; access via SSM or a bastion

    - C. Outside the VPC entirely

    - D. In the Internet Gateway

5.  **You need SSH-like access to an EC2 instance in a private subnet without opening inbound ports. What do you use?**

    - A. A VPN

    - B. Port 22 open to 0.0.0.0/0

    - C. AWS Systems Manager Session Manager (no inbound port, audited)

    - D. A public Elastic IP

6.  **An Elastic IP attached to a running EC2 instance is free. When does it start costing money?**

    - A. Never

    - B. When the instance is stopped or the EIP is unattached

    - C. Only in `us-east-1`

    - D. Only at night

7.  **What does “instance store” mean for EC2?**

    - A. A persistent EBS volume

    - B. Ephemeral local NVMe storage that is wiped when the instance stops

    - C. S3 storage mounted as a drive

    - D. RAM

8.  **The `t3` instance family is designed for what?**

    - A. GPU workloads

    - B. Burstable general-purpose workloads with baseline CPU credits

    - C. Storage-optimized databases

    - D. Specialized ML inference

9.  **Which action should NEVER be taken with the AWS root account credentials?**

    - A. Use them for day-to-day deployments

    - B. Enable MFA on the root account

    - C. Lock them in a safe after initial setup

    - D. Rotate them

10. **What is the most common cost surprise when uploading many small files to S3?**

    - A. The storage cost

    - B. The request cost — `PUT`, `LIST`, and `GET` requests are billed per-request and can dominate for small files

    - C. Data transfer in (which is free)

    - D. The bucket creation cost (there isn’t one)

**View Answer Key**

1.  **B** — IAM Roles via Instance Profiles are the secure, auditable way.

2.  **B** — The four hardening defaults every bucket should have.

3.  **B** — Security Groups = stateful firewalls; NACLs = stateless, second layer.

4.  **B** — Private subnet is the only correct answer.

5.  **C** — Session Manager is the modern, no-inbound-port way.

6.  **B** — EIPs cost money whenever they are not serving a running instance.

7.  **B** — Ephemeral storage; use EBS if you want persistence.

8.  **B** — `t` family is burstable, good default for dev and small prod.

9.  **A** — Root should be locked away after setup; use IAM users/roles for everything else.

10. **B** — Request costs dominate for workloads with lots of small files.
