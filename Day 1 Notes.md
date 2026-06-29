# 📘 DevOps Study Notes DAY 1

> **Topics Covered:** Linux Troubleshooting · AWS Networking · Interview Prep  


---

## 📑 Table of Contents

- [🐧 Linux Troubleshooting Commands](#-linux-troubleshooting-commands)
- [☁️ AWS Networking](#️-aws-networking)
  - [VPC](#vpc-virtual-private-cloud)
  - [Public Subnet](#public-subnet)
  - [Private Subnet](#private-subnet)
  - [Internet Gateway](#internet-gateway-igw)
  - [NAT Gateway](#nat-gateway)
  - [Security Group](#security-group)
  - [NACL](#nacl-network-acl)
- [🎯 Interview Questions](#-interview-questions)

---

## 🐧 Linux Troubleshooting Commands

| Command | Description |
|---|---|
| `free -h` | Shows RAM and swap memory usage in human-readable format |
| `top` | Displays real-time CPU, memory, and running process information |
| `df -h` | Shows disk space usage of all mounted filesystems |
| `ps -ef` | Lists all running processes with detailed information |
| `systemctl status nginx` | Checks whether the Nginx service is running and its current status |
| `journalctl -xe` | Displays recent system logs and detailed error messages for troubleshooting |
| `ss -tulpn` | Shows listening TCP/UDP ports along with the associated process IDs and names |

### 🔍 Quick Examples

```bash
# Check memory usage
free -h

# Monitor system in real-time
top

# Check disk space
df -h

# List all running processes
ps -ef

# Check nginx status
systemctl status nginx

# View system logs with errors
journalctl -xe

# View open ports and listening services
ss -tulpn
```

---

## ☁️ AWS Networking

### VPC (Virtual Private Cloud)

A VPC is your **private network inside AWS** where you launch resources such as EC2, ECS, RDS, and Load Balancers.

```
VPC CIDR:       10.0.0.0/16
Public Subnet:  10.0.1.0/24
Private Subnet: 10.0.2.0/24
```

---

### Public Subnet

A subnet that can access the internet through an **Internet Gateway**.

**Common Resources:**
- ALB (Application Load Balancer)
- Bastion Host
- NAT Gateway

**Route Table:**
```
0.0.0.0/0 → Internet Gateway
```

---

### Private Subnet

A subnet that does **not** have direct internet access.

**Common Resources:**
- ECS Tasks
- EC2 Application Servers
- RDS Databases

**Benefits:**
- ✅ More secure
- ✅ Hidden from the internet

---

### Internet Gateway (IGW)

Allows resources in a **public subnet** to communicate with the internet.

```
User
 ↓
Internet
 ↓
Internet Gateway
 ↓
Public Subnet
```

---

### NAT Gateway

Allows resources in a **private subnet** to access the internet **without** exposing them to incoming internet traffic.

**Use Cases:**
- ECS pulling Docker images
- EC2 installing packages
- Downloading updates

```
Private EC2/ECS
      ↓
 NAT Gateway
      ↓
Internet Gateway
      ↓
  Internet
```

---

### Security Group

Acts as a **firewall at the resource/instance level**.

| Feature | Detail |
|---|---|
| Attached to | EC2, ECS, ALB, RDS |
| Type | Stateful |
| Rules | Allow only |

```
Allow TCP 80  from 0.0.0.0/0
Allow TCP 443 from 0.0.0.0/0
```

---

### NACL (Network ACL)

Acts as a **firewall at the subnet level**.

| Feature | Detail |
|---|---|
| Attached to | Subnet |
| Type | Stateless |
| Rules | Allow & Deny |

```
Allow TCP 80
Allow TCP 443
Deny specific IP range
```

---

## 🎯 Interview Questions

### 1️⃣ Security Group vs NACL

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance Level | Subnet Level |
| State | Stateful | Stateless |
| Rules | Allow only | Allow & Deny |
| Management | Easier | More granular |
| Applied to | EC2 / ECS / RDS | Entire Subnet |

**Real-world Example:**
- **Security Group** → Allow MySQL port `3306` from ECS only
- **NACL** → Block a malicious IP range from the entire subnet

---

### 2️⃣ Why NAT Gateway?

Private resources often need **outbound internet access**.

**Examples:**
- ECS pulling images from Docker Hub
- EC2 running `yum update`
- Downloading software packages

```
Without NAT:  Private EC2/ECS → ❌ No Internet

With NAT:     Private EC2/ECS → NAT Gateway → Internet ✅
```

---

### 3️⃣ What happens when you open google.com?

```
Browser
   ↓
DNS Lookup (checks local cache → resolver → Google IP)
   ↓
TCP Connection established
   ↓
TLS Handshake (HTTPS)
   ↓
HTTP Request sent
   ↓
Google returns HTML/CSS/JS
   ↓
Browser renders the page 🎉
```

**Step-by-step:**
1. Browser checks local DNS cache
2. DNS query sent to resolver
3. Resolver finds Google's IP address
4. Browser establishes TCP connection
5. TLS handshake happens (HTTPS)
6. Browser sends HTTP request
7. Google returns HTML/CSS/JS
8. Browser renders the webpage

---

### 4️⃣ How does ECS connect to RDS?

```
Internet
   ↓
Route53
   ↓
ALB (Public Subnet)
   ↓
ECS Service (Private Subnet)
   ↓
RDS (Private Subnet)
```

**Security Group Rules:**

```
# ECS Security Group
Allow port 8080 from ALB SG

# RDS Security Group
Allow port 3306 from ECS SG
```

> ⚠️ **Key Points:**
> - RDS should **never** be publicly accessible
> - ECS talks to RDS using the **private endpoint** inside the VPC
> - All communication happens over **private IPs**, not the internet

---
