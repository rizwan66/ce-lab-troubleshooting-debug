# Scenario 1 — Cannot SSH to EC2 Instance (Theoretical)

**Type:** Thought Exercise
**Date:** 2026-03-15
**Symptom:** `ssh: connect to host ec2-xx-xx-xx.compute.amazonaws.com port 22: Connection timed out`

---

## Symptoms Observed

- EC2 instance shows **running** in the AWS Console
- Security group appears to have an SSH inbound rule
- SSH connection times out — not refused (meaning packets are being *dropped*, not *rejected*)
- A "connection refused" error would mean the port is reachable but nothing is listening; a *timeout* means traffic is not arriving at all

---

## Investigation Steps

### Step 1 — Confirm your current public IP

**Why first:** Your IP may have changed since you created the security group rule. A `/32` rule that was valid yesterday may not cover your IP today.

```bash
curl -s https://checkip.amazonaws.com
# Example output: 203.0.113.42
```

**What it tells you:** Your current egress IP. Compare this against the SG inbound rule source CIDR.

**Decision:** If your IP is not covered by the SG rule → **go to Fix 1.**

---

### Step 2 — Inspect the Security Group inbound rules

**Why:** A rule may exist for port 22 but target the wrong source IP or CIDR.

```bash
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxxxx \
  --query 'SecurityGroups[*].IpPermissions[?FromPort==`22`]' \
  --output table
```

**What to look for:**
- `IpRanges` — the allowed CIDR blocks
- `UserIdGroupPairs` — if access is granted via another SG (e.g., bastion)
- A rule with `0.0.0.0/0` means all IPs are allowed
- A rule with `x.x.x.x/32` means only that one IP is allowed

**Decision:** If your IP is not in any listed range → **Fix 1.** If the rule looks correct, continue.

---

### Step 3 — Check the Network ACL (NACL)

**Why:** NACLs are stateless and evaluated *before* security groups at the subnet boundary. An explicit DENY rule — or a missing ALLOW rule — drops traffic even if the SG permits it. Unlike SGs, NACLs also require an explicit *outbound* rule for the return traffic.

```bash
# Find which subnet the instance is in
aws ec2 describe-instances --instance-ids i-xxxxxxxxxx \
  --query 'Reservations[*].Instances[*].SubnetId' --output text

# Check the NACL for that subnet
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values=subnet-xxxxxxxxxx \
  --query 'NetworkAcls[*].Entries' \
  --output table
```

**What to look for:**
- Inbound: a rule allowing TCP port 22 from your IP (or `0.0.0.0/0`)
- Outbound: a rule allowing TCP ports 1024–65535 to your IP (ephemeral return ports)
- Rules are evaluated lowest-number-first; a DENY at rule 100 overrides an ALLOW at rule 200

**Decision:** If a DENY rule precedes the ALLOW, or if no ALLOW exists → **Fix 2.**

---

### Step 4 — Verify the subnet has a route to the Internet Gateway

**Why:** If the instance is in a private subnet, outbound routes go to a NAT gateway and there is no inbound path from the internet, regardless of SG or NACL settings.

```bash
# Get the route table for the subnet
aws ec2 describe-route-tables \
  --filters Name=association.subnet-id,Values=subnet-xxxxxxxxxx \
  --query 'RouteTables[*].Routes' \
  --output table
```

**What to look for:**
- A route `0.0.0.0/0 → igw-xxxxxxxxxx` = **public subnet** (SSH should work)
- A route `0.0.0.0/0 → nat-xxxxxxxxxx` = **private subnet** (no direct inbound access)
- No default route at all = completely isolated subnet

**Decision:** If no IGW route → **Fix 3.**

---

### Step 5 — Confirm a public IP is assigned to the instance

**Why:** An instance in a public subnet may still have no public IP if the subnet's auto-assign setting was off at launch, and no Elastic IP was attached.

```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxxxx \
  --query 'Reservations[*].Instances[*].[PublicIpAddress,PublicDnsName]' \
  --output table
```

**What to look for:** `None` or empty value means no public IP — you cannot reach it directly from the internet.

**Decision:** If no public IP → **Fix 4.**

---

### Step 6 — Verify sshd is actually running on the instance

**Why:** All of the above could be correct, but the SSH daemon itself may have crashed, failed to start, or been misconfigured.

**How to access without SSH** — use AWS Systems Manager Session Manager (requires SSM agent, which comes pre-installed on Amazon Linux 2):

```bash
aws ssm start-session --target i-xxxxxxxxxx
```

Then inside the session:
```bash
sudo systemctl status sshd
sudo journalctl -u sshd -n 30 --no-pager
sudo sshd -t   # test sshd config syntax
```

**What to look for:**
- `active (running)` — sshd is fine; the problem is network-layer
- `failed` or `inactive (dead)` — sshd is not running → **Fix 5**
- Config syntax errors in `journalctl` output → fix `/etc/ssh/sshd_config`

---

## Decision Tree

```
Connection times out on port 22?
│
├─ [1] Is your current IP covered by the SG inbound rule?
│        NO  →  Fix 1: Update SG inbound rule to allow your IP on TCP/22
│        YES → continue
│
├─ [2] Does the subnet NACL allow TCP/22 inbound AND TCP/1024-65535 outbound?
│        NO  →  Fix 2: Add NACL allow rules for both directions
│        YES → continue
│
├─ [3] Does the subnet route table have 0.0.0.0/0 → igw-xxx?
│        NO  →  Fix 3: Use bastion host, SSM Session Manager, or move instance to public subnet
│        YES → continue
│
├─ [4] Does the instance have a public IP or Elastic IP?
│        NO  →  Fix 4: Allocate and associate an Elastic IP
│        YES → continue
│
└─ [5] Is sshd running on the instance? (check via SSM)
         NO  →  Fix 5: sudo systemctl start sshd
         YES → Investigate further: key pair mismatch, wrong username, sshd_config AllowUsers
```

---

## Proposed Fixes

### Fix 1 — Update Security Group inbound rule

```bash
# Get your current IP
MY_IP=$(curl -s https://checkip.amazonaws.com)

# Remove the old stale rule (if it was a /32)
aws ec2 revoke-security-group-ingress \
  --group-id sg-xxxxxxxxxx \
  --protocol tcp --port 22 \
  --cidr 203.0.113.0/32   # old IP

# Add rule for your current IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxx \
  --protocol tcp --port 22 \
  --cidr ${MY_IP}/32
```

Or in the **AWS Console:** EC2 → Security Groups → Inbound rules → Edit → change source to "My IP".

---

### Fix 2 — Add NACL rules

```bash
# Inbound: allow SSH from anywhere (or restrict to your IP)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-xxxxxxxxxx \
  --ingress --rule-number 100 \
  --protocol tcp --port-range From=22,To=22 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Outbound: allow ephemeral return ports
aws ec2 create-network-acl-entry \
  --network-acl-id acl-xxxxxxxxxx \
  --egress --rule-number 100 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow
```

---

### Fix 3 — Instance in private subnet

Option A — Use SSM Session Manager (no SSH needed, no public IP needed):
```bash
aws ssm start-session --target i-xxxxxxxxxx
```

Option B — Use a bastion host in the public subnet:
```bash
ssh -i key.pem -J ec2-user@BASTION_PUBLIC_IP ec2-user@INSTANCE_PRIVATE_IP
```

Option C — Attach an Internet Gateway and update the route table in the AWS Console.

---

### Fix 4 — Associate an Elastic IP

```bash
# Allocate a new EIP
ALLOC_ID=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)

# Associate it with the instance
aws ec2 associate-address \
  --instance-id i-xxxxxxxxxx \
  --allocation-id $ALLOC_ID
```

---

### Fix 5 — Start sshd

```bash
# Via SSM Session Manager
sudo systemctl start sshd
sudo systemctl enable sshd   # ensure it starts on reboot
sudo systemctl status sshd
```

---

## Prevention

| Risk | Prevention |
|---|---|
| SG rule scoped to dynamic IP | Use a VPN with a static IP, or use SSM Session Manager instead of SSH |
| Accidental NACL DENY rules | Keep NACLs simple (allow all) and use SGs for fine-grained control |
| Instance in private subnet | Tag subnets clearly (`public`/`private`); use SSM as the standard access method |
| sshd crashes on bad config | Test config before applying: `sudo sshd -t`; use `cloud-init` to validate on boot |
| No public IP | Enable "Auto-assign public IP" on public subnets, or always use Elastic IPs |
