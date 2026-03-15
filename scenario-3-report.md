# Scenario 3 — Application Crash Loop (Hands-On)

**Instance IP:** `100.31.87.125`
**Date:** 2026-03-15
**Symptom:** Web application configured as a systemd service keeps crashing and will not stay running. Security group is correct (ports 22, 80, 8080 open).

---

## Symptoms Observed

On first connecting via SSH:
- Instance is accessible over SSH — OS is healthy
- Visiting the app in a browser returns "connection refused" or times out intermittently
- The connection briefly succeeds then drops — suggesting the app starts, then crashes

Running `systemctl status webapp` immediately reveals a crash loop:
```
Active: activating (auto-restart) (Result: exit-code) since ...; 3s ago
```

The service is configured with `Restart=always`, so systemd keeps relaunching it, but it exits within milliseconds each time.

---

## Investigation Steps

### Step 1 — Check service status

```bash
systemctl status webapp
```

**Output:**
```
● webapp.service - My Web Application
   Loaded: loaded (/etc/systemd/system/webapp.service; enabled)
   Active: activating (auto-restart) (Result: exit-code) since 2026-03-15 14:22:10 UTC; 3s ago
  Process: 4821 ExecStart=/usr/bin/node app.js (code=exited, status=1/FAILURE)
 Main PID: 4821 (code=exited, status=1/FAILURE)

Mar 15 14:22:10 ip-172-31-42-20 node[4821]: Error: Missing required environment variable: PORT
Mar 15 14:22:10 ip-172-31-42-20 systemd[1]: webapp.service: Main process exited, code=exited, status=1/FAILURE
Mar 15 14:22:10 ip-172-31-42-20 systemd[1]: Failed to start My Web Application.
```

**What it tells us:**
- Status is `activating (auto-restart)` — never reaches `active (running)`
- Exit code is `1/FAILURE` — the app is calling `process.exit(1)` intentionally
- The inline log line already shows the error: `Missing required environment variable: PORT`

---

### Step 2 — Check detailed application logs

```bash
journalctl -u webapp -n 50 --no-pager
```

**Output (repeated pattern):**
```
Mar 15 14:21:50 ... systemd[1]: Started My Web Application.
Mar 15 14:21:50 ... node[4799]: Error: Missing required environment variable: PORT
Mar 15 14:21:50 ... systemd[1]: webapp.service: Main process exited, code=exited, status=1/FAILURE
Mar 15 14:21:50 ... systemd[1]: webapp.service: Scheduled restart job, restart counter is at 4.
Mar 15 14:21:55 ... systemd[1]: Started My Web Application.
Mar 15 14:21:55 ... node[4812]: Error: Missing required environment variable: PORT
Mar 15 14:21:55 ... systemd[1]: webapp.service: Main process exited, code=exited, status=1/FAILURE
Mar 15 14:21:55 ... systemd[1]: webapp.service: Scheduled restart job, restart counter is at 5.
```

**What it tells us:** The same error fires on every attempt with zero variation. This is not a transient failure or race condition — the environment is identically wrong on every launch. No dependency is failing intermittently; the issue is structural and present from the first millisecond of startup.

---

### Step 3 — Inspect the application source code

```bash
cat /home/ec2-user/webapp/app.js
```

**Output:**
```javascript
const express = require('express');
const app = express();

const PORT = process.env.PORT;

if (!PORT) {
  console.error('Error: Missing required environment variable: PORT');
  process.exit(1);
}

app.get('/', (req, res) => {
  res.send('<h1>Hello from the Web App!</h1>');
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server is running on port ${PORT}`);
});
```

**What it tells us:**
- Line 4: `const PORT = process.env.PORT` — the port is read from an environment variable, not hardcoded
- Lines 6–9: An explicit startup guard — if `PORT` is not set, the process prints the error and exits with code `1`
- Line 15: `app.listen(PORT, '0.0.0.0', ...)` — the bind address is already correct (`0.0.0.0`), so Scenario 2's issue does not apply here
- The application logic is sound; the crash is intentional and by design

---

### Step 4 — Inspect the systemd service file

```bash
cat /etc/systemd/system/webapp.service
```

**Output:**
```ini
[Unit]
Description=My Web Application
After=network.target

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/home/ec2-user/webapp
ExecStart=/usr/bin/node app.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**What it tells us:** The `[Service]` section has no `Environment=` directive. When systemd launches a service, it does **not** inherit the shell environment of any user — it starts the process in a minimal, clean environment. Therefore `process.env.PORT` is `undefined` every single time.

Confirm no environment variables are set on the service:
```bash
systemctl show webapp | grep Environment
# Returns nothing
```

**Root cause confirmed:** The service definition is missing `Environment=PORT=8080`.

---

## Root Cause

The `webapp.service` systemd unit file does not define a `PORT` environment variable. Systemd launches processes in an isolated environment that does not inherit shell variables from any user session. The application code correctly validates that `PORT` is set at startup and exits if it is not — which is good defensive coding practice. The fault is in the service configuration, not the application logic.

**Why this is easy to miss:** Running the app manually in an SSH session works fine if you have `PORT` exported in your shell:
```bash
export PORT=8080
node app.js   # works perfectly
```
This creates the false impression the app is fine, because in a developer's shell the variable exists. But systemd has no knowledge of that shell state.

---

## Proposed Fix

### Fix — Add `Environment=` to the service unit file

**File to edit:** `/etc/systemd/system/webapp.service`

```bash
sudo nano /etc/systemd/system/webapp.service
```

Add `Environment=PORT=8080` to the `[Service]` section:

```ini
[Unit]
Description=My Web Application
After=network.target

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/home/ec2-user/webapp
ExecStart=/usr/bin/node app.js
Environment=PORT=8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

If the app requires multiple environment variables (e.g., a database connection), add them as separate lines:
```ini
Environment=PORT=8080
Environment=DB_HOST=localhost
Environment=DB_NAME=myapp
```

Or use an environment file for cleaner management:
```ini
EnvironmentFile=/etc/webapp/webapp.env
```

```bash
# Create the env file
sudo mkdir -p /etc/webapp
echo "PORT=8080" | sudo tee /etc/webapp/webapp.env
```

---

### Apply the fix

```bash
# Reload systemd to pick up the unit file change
sudo systemctl daemon-reload

# Restart the service
sudo systemctl restart webapp

# Verify it stays running (wait a few seconds, check again)
sudo systemctl status webapp
```

**Expected output after fix:**
```
● webapp.service - My Web Application
   Active: active (running) since 2026-03-15 14:30:01 UTC; 8s ago
 Main PID: 5102 (node)

Mar 15 14:30:01 ... node[5102]: Server is running on port 8080
```

---

### Verify end-to-end

```bash
# Confirm the app is now listening on the correct port and interface
sudo ss -tuln | grep 8080
# Expected: tcp  LISTEN  0.0.0.0:8080

# Test locally
curl http://localhost:8080
# Expected: <h1>Hello from the Web App!</h1>

# Test from external machine (SG already allows 8080)
curl http://100.31.87.125:8080
# Expected: <h1>Hello from the Web App!</h1>
```

---

## Prevention

| Risk | Prevention |
|---|---|
| Missing env vars in systemd service | Always run `systemctl show <service> \| grep Environ` after writing a new service file to confirm vars are present |
| Developer tests in shell but not as service | Add a smoke test after deployment that checks `systemctl status` *and* `curl http://localhost:PORT` — never just one |
| Silently wrong env in production | Log all required env vars at startup (with values masked for secrets) so any future crash shows exactly what was and wasn't set |
| `Restart=always` masking the real error | Always check `journalctl -u <service>` when a service won't stay up — `systemctl status` only shows the last few lines |
| Config drift between dev and prod | Store service files in version control alongside the application code; use Ansible or Terraform to deploy them consistently |
