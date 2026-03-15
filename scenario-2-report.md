# Scenario 2 — Website Not Loading (Hands-On)

**Instance IP:** `54.91.100.17`
**Date:** 2026-03-15
**Symptom:** Web application is reported running, SSH works, but users cannot load the site in a browser

---

## Symptoms Observed

On first connecting via SSH:
- Instance is up and accessible over SSH — the OS is healthy
- No obvious errors on login banner
- The application process (`node`) appears in `ps` output — something *is* running

From the browser (external):
- Connection times out — the browser never receives a response
- No "connection refused" (which would mean the port is reachable but nothing is listening)
- The timeout happens on both HTTP (port 80) and port 8080

---

## Investigation Steps

### Step 1 — Check if the application process is running

```bash
ps aux | grep node
```

**Output:**
```
ec2-user  1234  0.0  2.1  560312  21432 ?  Ssl  14:01  0:00 node /home/ec2-user/webapp/app.js
```

**What it tells us:** The Node.js process is alive with PID 1234. The OS did not crash it. The problem is not a dead process.

```bash
systemctl status webapp
```

**Output:**
```
● webapp.service - My Web Application
   Active: active (running) since 2026-03-15 14:01:33 UTC; 22min ago
 Main PID: 1234 (node)
```

**Conclusion:** The service is running and stable. The bug is not a crash.

---

### Step 2 — Find what port the app is listening on, and on which address

```bash
sudo ss -tuln
```

**Output:**
```
Netid  State   Local Address:Port  Peer Address:Port
tcp    LISTEN  127.0.0.1:8080      0.0.0.0:*
tcp    LISTEN  0.0.0.0:22          0.0.0.0:*
```

**Critical finding:** The application is listening on `127.0.0.1:8080`.

- Port `22` (SSH) listens on `0.0.0.0` — all interfaces — which is why SSH works.
- Port `8080` (the app) listens on `127.0.0.1` only — the loopback interface — meaning the OS will accept connections on this port **only from the same machine**, and will drop all packets arriving on any other network interface.

```bash
cat /home/ec2-user/webapp/app.js
```

**Output (relevant section):**
```javascript
const PORT = 8080;
const HOST = '127.0.0.1';   // ← bound to loopback only

app.listen(PORT, HOST, () => {
  console.log(`Server running at http://${HOST}:${PORT}`);
});
```

**What it tells us:** The developer hardcoded `127.0.0.1` as the bind address. Any connection from outside the machine — even from the instance's own private IP — is rejected at the network stack before the app ever sees it.

---

### Step 3 — Test the app locally from inside the instance

```bash
curl http://localhost:8080
curl http://127.0.0.1:8080
```

**Output:**
```html
<h1>Hello from the Web App!</h1>
```

**Status: 200 OK.** The application is healthy and responding correctly when accessed via loopback.

```bash
hostname -I
# Returns: 172.31.42.15

curl http://172.31.42.15:8080
```

**Output:**
```
curl: (7) Failed to connect to 172.31.42.15 port 8080: Connection refused
```

**What it tells us:** Even the private IP cannot reach the app. The loopback binding is confirmed — all non-loopback interfaces are refused. This means external traffic has no path to the application regardless of firewall rules.

---

### Step 4 — Can external traffic reach the app?

**Two independent blockers:**

**Blocker A — Application bind address (software layer):**

The app is bound to `127.0.0.1`. The kernel discards any packet destined for port 8080 that arrives on the `eth0` interface (public/private IP). The application process never sees these packets.

**Blocker B — Security Group (AWS network layer):**

```bash
# AWS Console: EC2 → Instance → Security tab → Inbound rules
```

**Security group inbound rules observed:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | 0.0.0.0/0 |

**Port 8080 is not listed.** AWS drops all packets destined for port 8080 at the security group boundary — before they even reach the instance's network interface.

**Answer:** No. External traffic cannot reach the app for two compounding reasons:
1. The security group has no inbound rule for TCP/8080 — AWS discards the packets
2. Even if the SG allowed it, the app is bound to `127.0.0.1` — the OS would also refuse the connection

Both issues must be fixed.

---

## Root Cause

| # | Layer | Root Cause | Effect |
|---|---|---|---|
| 1 | Application | `app.js` binds to `127.0.0.1` instead of `0.0.0.0` | App only accepts loopback connections; all external interfaces are refused |
| 2 | AWS | Security group has no inbound rule for TCP/8080 | AWS drops all external packets on port 8080 before they reach the instance |

The application itself works correctly — it serves valid responses. The failure is entirely in how and where it is listening, and what AWS permits through at the network boundary.

---

## Proposed Fix

### Fix 1 — Change the app to bind on all interfaces

```bash
sudo nano /home/ec2-user/webapp/app.js
```

Change:
```javascript
const HOST = '127.0.0.1';
```

To:
```javascript
const HOST = '0.0.0.0';
```

Restart the service:
```bash
sudo systemctl restart webapp
sudo systemctl status webapp
```

Verify the new bind address:
```bash
sudo ss -tuln | grep 8080
# Expected: tcp  LISTEN  0.0.0.0:8080   ← was 127.0.0.1:8080
```

Verify the private IP now works:
```bash
curl http://172.31.42.15:8080
# Expected: <h1>Hello from the Web App!</h1>
```

---

### Fix 2 — Open port 8080 in the Security Group

**AWS Console:**
```
EC2 → Instances → Select instance → Security tab
→ Click the Security Group link
→ Inbound rules → Edit inbound rules → Add rule:
  Type:        Custom TCP
  Protocol:    TCP
  Port range:  8080
  Source:      0.0.0.0/0
→ Save rules
```

**AWS CLI:**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxxxx \
  --protocol tcp \
  --port 8080 \
  --cidr 0.0.0.0/0
```

---

### Verify end-to-end after both fixes

```bash
# From inside the instance — via private IP
curl http://172.31.42.15:8080
# Expected: 200 OK

# From your local machine or browser
curl http://54.91.100.17:8080
# Expected: <h1>Hello from the Web App!</h1>
```

---

## Prevention

| Risk | Prevention |
|---|---|
| App bound to loopback | Default to `HOST = '0.0.0.0'` in templates; use `process.env.HOST` so it's configurable per environment |
| SG missing app port | Use Infrastructure as Code (Terraform/CloudFormation) to define SG rules alongside the app; include port checks in deployment smoke tests |
| Both issues together undetected | Add a post-deploy health check: `curl http://PUBLIC_IP:PORT` from outside — fail the deploy if it doesn't return 200 |
| Developer testing locally only | Require `curl http://PRIVATE_IP:PORT` (not just localhost) as part of the deployment checklist |
