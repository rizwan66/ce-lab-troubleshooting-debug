# Personal Troubleshooting Methodology

A structured guide for diagnosing and resolving infrastructure and application issues in cloud environments.

---

## 1. Define the Problem

Before touching anything, get precise about what is actually broken.

**Questions to answer first:**
- What is the exact symptom? (e.g., "connection timeout", "HTTP 502", "service exits with code 1")
- Who is affected — all users, some users, one user, one region?
- When did it start? Did anything change around that time (deploy, config update, instance restart)?
- What is the expected behaviour vs. the observed behaviour?
- Is it consistent or intermittent?

**Write it down in one sentence:**
> "The webapp service on instance i-xxxx exits immediately after systemd starts it, with exit code 1/FAILURE."

A vague problem statement leads to vague investigation. A precise one tells you where to start.

---

## 2. Gather Information First — Don't Touch Yet

Resist the urge to restart things or change config immediately. Gather a snapshot of the current state first, because:
- Restarting may clear logs or change state you needed
- You may fix a symptom without finding the root cause
- Other people may be using the same environment

**Standard first-pass commands for any EC2 issue:**

```bash
# OS and uptime
uptime
uname -r

# Disk — full disk causes silent app failures
df -h

# Memory
free -h

# Is the service running?
systemctl status <servicename>

# What's actually listening on the network?
sudo ss -tuln

# Recent system errors
sudo journalctl -p err -n 30 --no-pager

# Service-specific logs
journalctl -u <servicename> -n 50 --no-pager

# Running processes
ps aux | grep <appname>
```

**For network issues specifically:**
```bash
curl -s https://checkip.amazonaws.com   # your current IP
sudo ss -tuln                           # what's listening and on which interface
ping <host>                             # is the target reachable at all?
nc -zv <host> <port>                    # is a specific port open?
curl -v http://localhost:<port>         # can the app serve locally?
curl -v http://<private-ip>:<port>      # can it serve on the LAN interface?
```

---

## 3. Form and Test Hypotheses

Once you have the symptoms and initial data, generate possible causes — from most likely to least likely.

**Framework: work from the outside in (or bottom up)**

For a web app that isn't loading externally:
```
Layer 1 (AWS)      → Is the Security Group allowing the port?
Layer 2 (OS/net)   → Is the app bound to the right interface (0.0.0.0 vs 127.0.0.1)?
Layer 3 (Process)  → Is the process running at all?
Layer 4 (App)      → Is the app healthy when accessed locally?
Layer 5 (Config)   → Does the app have all the environment/config it needs?
```

**For each hypothesis:**
1. State it clearly: *"I think the SG is blocking port 8080"*
2. Identify the single command that will confirm or deny it: `aws ec2 describe-security-groups ...`
3. Run it — look at the output with fresh eyes
4. Confirm or eliminate: *"Port 8080 is not in the inbound rules — hypothesis confirmed"*
5. Move to the next hypothesis if denied

**Don't fix multiple things at once.** If you change two things simultaneously, you won't know which one resolved the issue — and you might accidentally mask a second problem.

---

## 4. Use Logs as Your Primary Evidence

Logs are the most reliable source of truth. Always check logs before guessing.

**Log hierarchy — check in this order:**

```bash
# 1. Service status (last few lines of logs, current state)
systemctl status <service>

# 2. Full service journal (50-100 lines, look for first error)
journalctl -u <service> -n 100 --no-pager

# 3. System-level errors (kernel, OOM killer, disk errors)
sudo journalctl -p err -n 50 --no-pager

# 4. Application-specific log files (if the app writes its own)
sudo tail -50 /var/log/<app>/error.log
sudo tail -50 /var/log/nginx/error.log
sudo tail -50 /var/log/httpd/error_log

# 5. Auth/SSH issues
sudo tail -30 /var/log/secure        # Amazon Linux / RHEL
sudo tail -30 /var/log/auth.log      # Ubuntu / Debian
```

**What to look for in logs:**
- The *first* error in a sequence — not the most recent one. The first error is usually the root cause; subsequent ones are often cascading side effects.
- Exit codes: `exit-code 1` = app chose to exit; `killed` = OOM or signal; `core-dump` = unhandled exception
- Timestamps: did errors start at a specific event (deploy, midnight cron, reboot)?
- Repeated identical messages = structural problem (like a missing env var); varying messages = intermittent/resource problem

---

## 5. Common Root Cause Patterns (and Their Signatures)

| Symptom | First place to look | Common root cause |
|---|---|---|
| SSH connection timeout | Security group inbound rules | Port 22 blocked or your IP changed |
| SSH connection refused | `systemctl status sshd` | sshd not running or wrong port |
| Browser can't reach app | `sudo ss -tuln` + SG rules | App bound to 127.0.0.1, or SG missing port |
| App crashes immediately | `journalctl -u <svc>` | Missing environment variable or missing dependency |
| App crashes after a few hours | `df -h` then logs | Disk full, memory exhausted, or connection pool exhausted |
| App works locally, not externally | `ss -tuln` (look at Local Address) | Bound to 127.0.0.1 instead of 0.0.0.0 |
| Service keeps restarting | `journalctl` — find the *first* error | Missing config, bad env var, missing file/dependency |
| Works in shell, fails as service | `systemctl show <svc> \| grep Environ` | Env var present in shell but not in systemd unit |

---

## 6. Verify a Fix Works — Don't Assume

After applying a fix, confirm each layer independently.

**Verification checklist:**

```bash
# 1. Service is up and stable (not in restart loop)
systemctl status <service>
# Look for: Active: active (running)

# 2. Listening on the right interface and port
sudo ss -tuln | grep <port>
# Look for: 0.0.0.0:<port>  not  127.0.0.1:<port>

# 3. App responds locally
curl http://localhost:<port>
# Look for: HTTP 200 and expected content

# 4. App responds via private IP (rules out loopback-only binding)
curl http://$(hostname -I | awk '{print $1}'):<port>
# Must also return HTTP 200

# 5. App responds from outside (the real user test)
curl http://<public-ip>:<port>
# Must return HTTP 200

# 6. Logs show no new errors
journalctl -u <service> -n 20 --no-pager
# Should show clean startup, no error lines

# 7. Fix survives a restart (prevents regression on next reboot)
sudo systemctl restart <service>
systemctl status <service>
```

Only when all six checks pass is the fix verified. A fix that only passes check #3 (localhost) but not #5 (external) means you've solved half the problem.

---

## 7. Document for Next Time

Good documentation means you — or anyone on your team — can resolve the same issue in minutes next time instead of hours.

**Minimum documentation for any incident:**

```
Issue:       [one sentence describing the symptom]
Environment: [instance ID, service name, date]
Root Cause:  [what was specifically wrong and why it caused the symptom]
Discovery:   [the exact command(s) that revealed the root cause]
Fix Applied: [exact commands or config changes made]
Verified by: [the command(s) confirming the fix worked]
Prevention:  [what to change in process/tooling to prevent recurrence]
```

**Where to document:**
- Post-incident: in a shared runbook or wiki, linked from the relevant ticket
- Recurring issues: convert to a monitoring alert so it's caught automatically next time
- Infrastructure fixes: commit the fixed config to version control with a clear commit message explaining *why* the change was made

---

## 8. Escalation Triggers

Know when to stop investigating alone and get help.

**Escalate when:**
- You've exhausted your decision tree and still can't identify the root cause
- The issue affects production and is ongoing (stop investigating, start mitigating first)
- You're about to make a change you can't easily reverse
- You've been on the same hypothesis for more than 30 minutes with no progress

**Before escalating, prepare:**
- The one-sentence problem statement
- What you've already checked and what each check showed
- Your current best hypothesis
- What access/permissions you need that you don't have

---

## Quick Reference: Diagnostic Command Cheatsheet

```bash
# ── SERVICE STATE ──────────────────────────────────────────
systemctl status <svc>                    # current state + last few log lines
systemctl list-units --failed             # all failed services
journalctl -u <svc> -n 100 --no-pager    # last 100 lines of service logs
journalctl -u <svc> -f                   # follow logs in real time
journalctl -p err -n 50 --no-pager       # all system-level errors

# ── NETWORK ────────────────────────────────────────────────
sudo ss -tuln                             # what's listening and on which address
curl -s https://checkip.amazonaws.com    # your current public IP
curl -v http://localhost:<port>           # test app via loopback
curl -v http://$(hostname -I | awk '{print $1}'):<port>  # test via private IP
nc -zv <host> <port>                      # is a remote port open?

# ── RESOURCES ──────────────────────────────────────────────
df -h                                     # disk usage (full disk = silent failures)
free -h                                   # memory usage
top -bn1 | head -20                       # CPU and process snapshot

# ── APPLICATION ────────────────────────────────────────────
cat /etc/systemd/system/<svc>.service     # service unit file
systemctl show <svc> | grep Environ       # env vars set on the service
cat /home/ec2-user/webapp/app.js          # application source
ps aux | grep <process>                   # is it running?

# ── AWS (from CLI) ─────────────────────────────────────────
aws ec2 describe-security-groups --group-ids sg-xxx \
  --query 'SecurityGroups[*].IpPermissions' --output table

aws ec2 describe-instances --instance-ids i-xxx \
  --query 'Reservations[*].Instances[*].[PublicIpAddress,SubnetId,State.Name]' \
  --output table
```
