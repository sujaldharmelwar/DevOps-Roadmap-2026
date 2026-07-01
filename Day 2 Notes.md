# 🛠️ Day 2 — Linux Troubleshooting, Nginx & AWS Networking

> **Goal:** Understand how to diagnose production issues across Linux servers, Nginx, and AWS networking — and speak confidently about them in interviews.

---

## 📁 Structure

```
Day-2/
├── Linux-Troubleshooting/
├── Nginx-Troubleshooting/
└── AWS-Networking/
```

---

# 📌 Task 1 — Linux Log Analysis

## 🔍 Key Commands

| Command | What it does |
|---|---|
| `tail -f /var/log/messages` | Streams the system log in real time |
| `journalctl -xe` | Shows recent journal logs with explanations (`-x`) and jumps to end (`-e`) |
| `journalctl -u nginx` | Shows logs for a specific service (e.g., nginx) |
| `dmesg` | Kernel ring buffer — hardware, boot, driver errors |
| `uptime` | Shows how long server has been up + load averages |
| `vmstat 1 5` | Snapshot of CPU, memory, I/O every 1 second, 5 times |

---

## 📝 Notes

### ❓ What is the difference between `journalctl` and log files?

| | `journalctl` | Log Files (`/var/log/`) |
|---|---|---|
| **Format** | Binary (structured, searchable) | Plain text |
| **Source** | systemd journal | Written by apps/daemons directly |
| **Filtering** | By unit, time, priority, boot | Manual `grep` needed |
| **Persistence** | Configurable (volatile or persistent) | Always on disk |
| **Examples** | `journalctl -u nginx` | `tail -f /var/log/nginx/error.log` |

> **Simple rule:** `journalctl` is for systemd-managed services. Log files are for everything else (apps, Apache, custom scripts).

---

### ❓ How do you identify application failures?

```bash
# Check if the service is running
systemctl status <service-name>

# See last 100 lines of service logs
journalctl -u <service-name> -n 100

# Check the app's own error log (example: nginx)
tail -f /var/log/nginx/error.log

# Look for ERROR / FATAL / Exception keywords
grep -i "error\|fatal\|exception" /var/log/messages
```

**Signs of application failure:**
- Service state shows `failed` or `inactive`
- Logs show `segfault`, `exit code 1`, `OOM killed`
- Port not listening (`ss -tulpn | grep <port>` shows nothing)

---

### ❓ How do you identify server crashes?

```bash
# Check kernel messages for OOM (Out of Memory) killer
dmesg | grep -i "oom\|killed\|panic"

# Check when the system last booted (crash = unexpected reboot)
who -b
last reboot

# Look for kernel panics in journal
journalctl -b -1   # logs from the PREVIOUS boot (before crash)

# Check for hardware errors
dmesg | grep -i "error\|fail\|hardware"
```

**Signs of a server crash:**
- `last reboot` shows an unexpected time
- `journalctl -b -1` has logs from before the gap
- `dmesg` shows `kernel panic` or OOM kill messages

---

### ❓ How do you identify high CPU and memory issues?

```bash
# Real-time CPU + memory overview
top
htop          # more visual (install if needed)

# CPU snapshot every 1 sec, 5 times
vmstat 1 5

# Top memory consumers
ps aux --sort=-%mem | head -10

# Top CPU consumers
ps aux --sort=-%cpu | head -10

# Check load averages (1, 5, 15 min)
uptime
# Output: load average: 3.50, 2.80, 1.90
# Rule: if load > number of CPUs, server is overloaded

# Check available memory
free -h
```

**Reading `vmstat` output:**
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 500000  10000 200000    0    0     0     0  500 1000 80  5 10  5  0
```
- `r` = processes waiting for CPU (high = CPU bottleneck)
- `si/so` = swap in/out (high = memory pressure)
- `us` = user CPU%, `id` = idle% (low idle = high load)

---

## 🚨 Production Scenario: "Website is Down"

### Step-by-Step Troubleshooting Flow

```
1. CONFIRM THE PROBLEM
   └── Can you reproduce it? Which URL? Which region?

2. CHECK SERVER HEALTH
   ├── uptime                        → Is server running? High load?
   ├── free -h                       → Is memory exhausted?
   └── df -h                         → Is disk full? (kills apps silently)

3. CHECK APPLICATION SERVICE
   ├── systemctl status nginx        → Is nginx running?
   ├── systemctl status <app>        → Is your app running?
   └── journalctl -u nginx -n 50    → Any errors?

4. CHECK NETWORK / PORT
   ├── ss -tulpn | grep 80           → Is port 80 open?
   ├── curl localhost                → Does it respond locally?
   └── curl http://<private-ip>      → Does it work internally?

5. CHECK LOGS
   ├── tail -f /var/log/nginx/error.log
   ├── tail -f /var/log/nginx/access.log
   └── journalctl -xe

6. CHECK AWS (if cloud)
   ├── EC2 instance state (running?)
   ├── Security Group: port 80/443 open?
   ├── Target Group health checks passing?
   └── ALB listener rules correct?

7. FIX & VERIFY
   └── After fix → curl + browser test + monitor logs
```

---

# 📌 Task 2 — Nginx + Networking Troubleshooting

## 🔍 Key Commands

```bash
systemctl status nginx    # Is nginx running?
nginx -t                  # Test nginx config for syntax errors
curl localhost            # Test from the same server
curl <private-ip>         # Test within the VPC/network
ss -tulpn                 # List all open ports and listening processes
```

---

## 📝 Notes

### ❓ What happens when Nginx service stops?

- Port 80 / 443 stops listening immediately
- All incoming requests get **Connection Refused**
- ALB health checks start failing → instances marked unhealthy
- Fix: `systemctl start nginx` or `systemctl restart nginx`
- Root cause: config error, OOM kill, manual stop, disk full

---

### ❓ Why does 502 Bad Gateway occur?

> **Nginx is up, but the backend app is down or not responding.**

```
User → Nginx → [❌ App is dead] → Nginx returns 502
```

**Causes:**
- App server crashed or stopped (Node, Gunicorn, etc.)
- App listening on wrong port
- Wrong `proxy_pass` URL in nginx config
- App started but not ready yet (startup time)

**Fix:**
```bash
systemctl status <your-app>          # Is the app running?
journalctl -u <your-app> -n 50      # Why did it crash?
ss -tulpn | grep <app-port>          # Is it listening?
nginx -t && systemctl reload nginx  # Check nginx config
```

---

### ❓ Why does 504 Gateway Timeout occur?

> **Nginx reached the app, but the app took too long to respond.**

```
User → Nginx → App (processing...) → [⏰ Timeout] → Nginx returns 504
```

**Causes:**
- App is overloaded / hanging
- Database query is slow
- External API call is timing out
- Nginx `proxy_read_timeout` too short

**Fix:**
```bash
# Increase timeout in nginx config
proxy_read_timeout 120s;
proxy_connect_timeout 120s;

# Check app performance / DB slow queries
# Check CPU/memory: top, vmstat
```

---

### ❓ Why does Connection Refused occur?

> **Nothing is listening on that port.**

**Causes:**
- Nginx is stopped
- App is stopped
- Wrong port in config
- Firewall / Security Group blocking

**Fix:**
```bash
ss -tulpn | grep 80        # Is anything on port 80?
systemctl status nginx     # Is nginx running?
# Check Security Group: port 80 inbound allowed?
```

---

### ❓ Difference between Port 80 and 443?

| | Port 80 (HTTP) | Port 443 (HTTPS) |
|---|---|---|
| **Protocol** | Plain HTTP | Encrypted HTTPS (TLS/SSL) |
| **Security** | Data visible in transit | Encrypted end-to-end |
| **Certificate** | Not required | SSL/TLS certificate required |
| **Use in prod** | Redirect to 443 only | All actual traffic |
| **Nginx config** | `listen 80;` | `listen 443 ssl;` |

> **Best practice:** Port 80 should only redirect to 443. Never serve real content on HTTP in production.

---

## 🏗️ Production Architecture — Request Flow

```
User (Browser)
      │
      ▼
  Route53 (DNS)
      │  Resolves domain → ALB IP
      ▼
  ALB (Application Load Balancer)
      │  Port 80/443, SSL termination, routes to target group
      ▼
  Nginx (on EC2)
      │  Reverse proxy, serves static files, proxies to app
      ▼
  Application (Node/Python/Java etc.)
      │  Business logic, connects to DB
      ▼
  Database (RDS/etc.)
```

### What can fail at each layer?

| Layer | What Can Fail | How to Troubleshoot |
|---|---|---|
| **Route53** | Wrong DNS record, TTL not expired, domain expired | `nslookup yourdomain.com` or `dig yourdomain.com` — check if it resolves to ALB |
| **ALB** | Listener not on port 80/443, target group unhealthy, security group blocking | AWS Console → ALB → Target Groups → Health Checks. Check SG allows 80/443 from internet |
| **Nginx** | Service stopped, config error, cert expired | `systemctl status nginx`, `nginx -t`, `curl localhost`, check error logs |
| **Application** | Crashed, wrong port, OOM killed, code bug | `systemctl status <app>`, `journalctl -u <app>`, `ss -tulpn \| grep <port>` |

---

# 📌 Task 3 — AWS Production Networking

## 📝 Core Concepts

### 🔒 Security Groups

- **Virtual firewall** at the **instance level**
- **Stateful** — if inbound is allowed, response automatically goes out
- Rules are **ALLOW only** (no explicit deny)

**Inbound rules:** What traffic can come IN to your instance
```
Example:
Type: HTTP    | Port: 80   | Source: 0.0.0.0/0      → Allow all internet
Type: SSH     | Port: 22   | Source: 10.0.0.0/8     → Allow only internal
Type: Custom  | Port: 5432 | Source: sg-0abc123      → Allow from app SG only
```

**Outbound rules:** What traffic can go OUT from your instance
```
Default: Allow ALL outbound (0.0.0.0/0)
Lock-down example: Only allow port 443 to 0.0.0.0/0 (internet HTTPS)
```

---

### 🗺️ Route Table

- Controls **where network traffic is directed**
- Each subnet is associated with a route table

```
Destination      Target
10.0.0.0/16  →  local            (traffic within VPC stays local)
0.0.0.0/0    →  igw-xxxxx        (everything else goes to Internet Gateway)
0.0.0.0/0    →  nat-xxxxx        (for private subnets → NAT Gateway)
```

---

### 🌐 Internet Gateway (IGW)

- Allows instances in **public subnets** to reach the internet
- Attached to the VPC
- Public subnet = subnet with route `0.0.0.0/0 → IGW`
- EC2 must have a **Public IP or Elastic IP** to use it

---

### 🔄 NAT Gateway

- Allows instances in **private subnets** to reach the internet (for updates, API calls)
- **Outbound only** — internet cannot initiate connections to private instances
- Lives in a **public subnet**, private subnet routes `0.0.0.0/0 → NAT GW`
- More secure than IGW for backend servers

```
Private EC2 → NAT Gateway (public subnet) → Internet Gateway → Internet
```

---

## 🚨 Production Scenario: ECS Cannot Connect to RDS

### Complete Troubleshooting Flow

```
ECS Task → [Can't connect] → RDS (PostgreSQL/MySQL on port 5432/3306)
```

**Check these things in order:**

#### Step 1 — Security Groups
```
ECS Task Security Group (Outbound):
  ✅ Allow port 5432 (or 3306) to RDS Security Group

RDS Security Group (Inbound):
  ✅ Allow port 5432 (or 3306) FROM ECS Security Group (not 0.0.0.0/0!)
```

#### Step 2 — Subnet & VPC
```
Are ECS and RDS in the same VPC?          → Must be same VPC or VPC peering
Are they in subnets that can talk?        → Check route tables
Is RDS in a private subnet?               → ECS must also be in same VPC
```

#### Step 3 — RDS Configuration
```
Is RDS instance running?                  → AWS Console → RDS → Status
Is "Publicly Accessible" needed?          → Usually NO for ECS-to-RDS
Correct endpoint used in app?             → Check env variable / secret
Correct port?                             → PostgreSQL=5432, MySQL=3306
```

#### Step 4 — Application Configuration
```
Is DB_HOST correct?                       → Should be RDS endpoint URL
Is DB_PORT correct?
Is DB_USER / DB_PASSWORD correct?         → Check Secrets Manager / env vars
Is the DB_NAME correct?
```

#### Step 5 — Network ACLs (if extra strict)
```
Does NACL on RDS subnet allow inbound from ECS subnet CIDR?
Does NACL on ECS subnet allow outbound to RDS subnet CIDR?
NACLs are STATELESS — must allow both directions explicitly
```

#### Step 6 — DNS Resolution
```bash
# Inside ECS task (exec into container):
nslookup <rds-endpoint>        # Does it resolve?
telnet <rds-endpoint> 5432     # Can it reach the port?
```

---

# 🎯 Interview Questions

---

### Q1. Difference between 502 and 504?

| | 502 Bad Gateway | 504 Gateway Timeout |
|---|---|---|
| **Meaning** | Backend is **down** or refusing connections | Backend is **too slow** or not responding in time |
| **Backend state** | Crashed / not running | Running but overloaded/hanging |
| **Root cause** | App stopped, wrong port, crash | Slow DB query, overloaded app, long computation |
| **Fix** | Restart the app | Optimize app/DB, increase timeout, scale up |

> **Memory trick:** 502 = "not answering the door." 504 = "answered but took forever."

---

### Q2. How to check which process is using a port?

```bash
# Method 1 — ss (modern, preferred)
ss -tulpn | grep :80

# Method 2 — netstat (older systems)
netstat -tulpn | grep :80

# Method 3 — lsof
lsof -i :80

# Method 4 — fuser
fuser 80/tcp

# Output shows: PID, process name, user
# Then: kill -9 <PID>  if you need to free the port
```

---

### Q3. Why does an app work on localhost but not externally?

**Common reasons:**

| Reason | Explanation | Fix |
|---|---|---|
| App bound to `127.0.0.1` | App only accepts local connections | Change to `0.0.0.0` in app config |
| Security Group blocking | Port not open in AWS SG | Add inbound rule for the port |
| Firewall (`iptables`/`ufw`) | OS-level firewall blocking | `ufw allow 80` or check iptables |
| Nginx not configured | nginx not proxying externally | Check `server_name` and `listen` in nginx.conf |
| No public IP | EC2 has no public IP | Assign Elastic IP or put behind ALB |

```bash
# Quick test — if this works but browser doesn't:
curl localhost:3000                 # Works ✅
curl http://<public-ip>:3000       # Fails ❌ → SG or firewall issue
```

---

### Q4. Security Group vs NACL?

| | Security Group | Network ACL (NACL) |
|---|---|---|
| **Level** | Instance level | Subnet level |
| **Stateful?** | ✅ Yes (return traffic automatic) | ❌ No (must allow both directions) |
| **Rules** | Allow only | Allow AND Deny |
| **Rule order** | All rules evaluated together | Rules evaluated in number order |
| **Default** | Deny all inbound, allow all outbound | Allow all in/out |
| **Use case** | Primary security layer | Extra defense, block specific IPs |

> **Simple rule:** Security Groups = your first line of defense (always use these). NACLs = extra layer for subnet-wide blocks.

---

### Q5. ECS to RDS Connection Troubleshooting Steps?

```
1. ✅ Check RDS is running (AWS Console)
2. ✅ Check ECS Task security group has OUTBOUND rule → port 5432/3306 to RDS SG
3. ✅ Check RDS security group has INBOUND rule ← port 5432/3306 from ECS SG
4. ✅ Both ECS and RDS in same VPC
5. ✅ Check app env vars: DB_HOST, DB_PORT, DB_USER, DB_PASS, DB_NAME
6. ✅ Check Secrets Manager / Parameter Store values are correct
7. ✅ Exec into ECS container and run: telnet <rds-endpoint> 5432
8. ✅ Check NACL on RDS subnet — does it allow ECS subnet CIDR?
9. ✅ Check RDS subnet route table — is routing correct within VPC?
10. ✅ Check RDS logs in CloudWatch for authentication errors
```

---

## 🧠 Quick Reference Cheat Sheet

```bash
# ===== LINUX HEALTH =====
uptime                          # Load average
free -h                         # Memory
df -h                           # Disk
top / htop                      # Live processes
vmstat 1 5                      # CPU + memory snapshots

# ===== LOGS =====
journalctl -u <service> -n 100  # Service logs
journalctl -b -1                 # Previous boot logs
dmesg | grep -i error           # Kernel errors
tail -f /var/log/nginx/error.log

# ===== NGINX =====
systemctl status nginx
nginx -t                        # Test config
systemctl reload nginx          # Reload without downtime

# ===== NETWORK =====
ss -tulpn                       # Open ports
curl -v http://localhost         # Test local
curl -I https://yourdomain.com  # Test external with headers
nslookup yourdomain.com         # DNS check
telnet <host> <port>            # Connectivity check
```

---

*Day 2 complete ✅ — Linux Logs · Nginx Troubleshooting · AWS Networking · Interview Prep*
