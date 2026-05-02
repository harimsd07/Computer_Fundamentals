# 🖥️ System Administration & DevOps — Complete Study Notes

<div align="center">

![Markdown](https://img.shields.io/badge/Format-Markdown-blue?style=flat-square&logo=markdown) ![Files](https://img.shields.io/badge/Files-46-green?style=flat-square) ![Lines](https://img.shields.io/badge/Lines-22%2C825%2B-orange?style=flat-square) ![Topics](https://img.shields.io/badge/Topics-45-purple?style=flat-square) ![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

**A comprehensive, production-grade reference for System Administrators, DevOps Engineers, and Cloud Practitioners.**

_46 markdown files · 22,825+ lines · Fully cross-linked · Mermaid diagrams throughout_

</div>

---

## 📖 What Is This?

This repository is a **complete, structured knowledge base** covering every major topic a modern sysadmin or DevOps engineer needs to know — from Linux fundamentals to Kubernetes, from bash scripting to AWS, from Git basics to ELK Stack.

Every file is:

- **Self-contained** — can be read independently
- **Cross-linked** — every file links to related topics
- **Practical** — real commands, real configs, real examples
- **Visual** — Mermaid architecture diagrams, flowcharts, sequence diagrams
- **Up to date** — covers modern tooling (Terraform, K8s, GitHub Actions, Prometheus, etc.)

---

## 🗂️ Topics at a Glance

### 🖥️ Operating System Fundamentals

|#|Topic|
|---|---|
|01|OS Fundamentals — Linux vs Windows, Kernel, User Space, GUI vs CLI|
|02|File System Structure and Navigation — FHS, inodes, mounts|
|03|Linux Command Line Basics — ls, cd, grep, pipes, redirects|
|04|Windows CMD and PowerShell Basics — dir, tasklist, Get-Process|
|05|User Permissions and Privilege Management — chmod, sudo, UAC, ACL|
|06|File Management Concepts — compression, search, checksums, rsync|

### 🌐 Networking

|#|Topic|
|---|---|
|07|Networking Fundamentals — OSI, IP, Subnet, Gateway, DNS, Ports, TCP/UDP|
|08|Networking Tools — ping, traceroute, netstat, nslookup, curl, nmap|
|09|Active Directory — Domain, DC, GPO, Kerberos, LDAP, FSMO Roles|
|10|IIS Web Server Basics — app pools, bindings, web.config|
|11|Time Synchronization — NTP, chrony, w32tm, stratum hierarchy|
|12|VPN Basics — IPSec, SSL/TLS, WireGuard, OpenVPN, split tunnel|
|22|DNS Deep Dive — BIND9, zone files, reverse DNS, SPF/DKIM/DMARC|
|32|Network Protocols Deep Dive — HTTP/2/3, DHCP, ARP, BGP, OSPF|

### 🔐 Security

|#|Topic|
|---|---|
|13|System Monitoring and Logging — journalctl, Event Viewer, syslog|
|14|Basic Security Concepts — auth, encryption, firewall, PKI, EDR|
|26|SSL/TLS & Certificates — Let's Encrypt, certbot, openssl, HTTPS|
|38|Linux System Hardening — SSH, Fail2Ban, PAM, auditd, sysctl, AppArmor|

### ⚙️ System Management

|#|Topic|
|---|---|
|15|Services and Process Management — systemd, cron, Windows SCM|
|16|Software Installation Methods — apt, pacman, winget, pip, npm|
|17|Cloud and Remote Access — SSH, RDP, Docker basics, AWS CLI|
|18|Troubleshooting Methodology — identify, isolate, fix, verify, document|
|35|Storage & RAID — LVM, RAID levels, ZFS, SAN/NAS, disk tools|
|37|Virtualization & Hypervisors — KVM, VMware ESXi, Hyper-V, VirtualBox|
|39|Windows Server Administration — DHCP, DNS, File Server, Registry, WEF|

### 🔧 Git & Version Control

|#|Topic|
|---|---|
|19|Git Fundamentals — objects, commits, staging, .gitignore|
|20|Git Commands and Workflow — init, clone, add, commit, push, pull|
|21|Branching, Merging, Conflicts & Best Practices — rebase, cherry-pick, Helm|

### 🌍 Web Servers & Email

|#|Topic|
|---|---|
|25|Web Servers — Nginx & Apache — vhosts, reverse proxy, PHP-FPM, SSL|
|34|Load Balancing & Reverse Proxy — HAProxy, Nginx LB, health checks|
|40|Email & Mail Servers — Postfix, Dovecot, SPF/DKIM/DMARC, SpamAssassin|

### 💻 Scripting & Automation

|#|Topic|
|---|---|
|23|Bash Scripting — variables, loops, functions, error handling, real scripts|
|31|PowerShell Scripting — objects, remoting, AD automation, modules|
|33|Regex Fundamentals — syntax, groups, lookaheads, grep/sed/awk/Python|
|45|Python for Sysadmins — psutil, paramiko, boto3, log parsing, SSH|

### 🗄️ Data & Storage

|#|Topic|
|---|---|
|24|Database Basics — MySQL, PostgreSQL, SQLite — CRUD, users, backup|
|29|Backup and Disaster Recovery — rsync, Restic, 3-2-1 rule, RPO/RTO|

### 🚀 DevOps & Cloud Native

|#|Topic|
|---|---|
|27|CI/CD Fundamentals — GitHub Actions, GitLab CI, Docker pipelines|
|28|Infrastructure as Code — Terraform & Ansible — HCL, playbooks, roles|
|30|Docker & Containers — Dockerfile, Compose, networking, volumes|
|41|Kubernetes Deep Dive — kubectl, Deployments, RBAC, Helm, HPA, Jobs|
|44|AWS Cloud Deep Dive — VPC, EC2, S3, IAM, RDS, Lambda, CloudWatch|

### 📊 Observability & Monitoring

|#|Topic|
|---|---|
|36|SNMP & Network Monitoring — Nagios, Zabbix, SNMP OIDs, traps|
|42|Monitoring with Prometheus & Grafana — PromQL, alerting, exporters|
|43|ELK Stack — Elasticsearch, Logstash, Kibana, Filebeat, ILM|

---

## 🚀 Quick Start

### Clone and Open

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/sysadmin-notes.git
cd sysadmin-notes

# Start at the master index
# Open 00_INDEX.md in your preferred viewer
```

### Recommended Viewers

|Viewer|Platform|Notes|
|---|---|---|
|**[Obsidian](https://obsidian.md)**|Windows / macOS / Linux|Best experience — graph view, backlinks|
|**VS Code** + [Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)|All|Renders Mermaid diagrams|
|**GitHub**|Browser|Renders tables + code blocks|
|**[Typora](https://typora.io)**|Windows / macOS / Linux|Clean WYSIWYG editor|
|**[Zettlr](https://www.zettlr.com)**|All|Academic-style, great for links|

> ⚠️ **Keep all `.md` files in the same folder** — cross-links use relative paths.

---

## ✨ Features

|Feature|Details|
|---|---|
|📐 **Architecture Diagrams**|Mermaid flowcharts, sequence diagrams, state diagrams, gitGraphs|
|🔗 **Cross-linked**|Every file links forward and backward to related topics|
|💻 **Dual coverage**|Linux AND Windows side-by-side for most topics|
|📋 **Comparison tables**|Tools, commands, config options — all tabulated|
|🛠️ **Real commands**|Copy-paste ready — no placeholder examples|
|🗺️ **Learning paths**|4 structured paths: Linux Sysadmin, Windows Sysadmin, DevOps, Security|
|📚 **4-week schedule**|Day-by-day study plan included in index|
|⚡ **Cheat sheets**|Quick reference for Linux, Git, Docker, K8s, AWS, Networking|

---

## 📊 Repository Stats

```
Total Files    : 46 (1 index + 45 topic files)
Total Lines    : 22,825+
Largest File   : 43_ELK_Stack.md (870 lines)
Sections       : 10
Learning Paths : 4
```

### Lines per Section

|Section|Files|Total Lines|
|---|---|---|
|OS Fundamentals|6|~2,313|
|Networking|8|~3,175|
|Security|4|~1,557|
|System Management|7|~2,937|
|Git & Version Control|3|~1,364|
|Web Servers & Email|3|~1,718|
|Scripting & Automation|4|~2,811|
|Data & Storage|2|~1,067|
|DevOps & Cloud Native|5|~3,475|
|Observability & Monitoring|3|~2,163|

---

## 📖 Learning Paths

### 🟢 Linux Sysadmin (Beginner → Intermediate)

```
01 OS → 02 FileSystem → 03 LinuxCLI → 05 Permissions → 07 Networking →
13 Monitoring → 15 Services → 23 Bash → 25 Nginx → 26 SSL → 22 DNS → 29 Backup
```

### 🔵 Windows Sysadmin

```
01 OS → 04 WinCLI → 05 Permissions → 09 AD → 10 IIS →
11 NTP → 31 PowerShell → 39 WinServer
```

### 🟡 DevOps Engineer

```
19 Git → 20 GitCmds → 21 Branching → 23 Bash → 27 CICD →
30 Docker → 28 IaC → 41 Kubernetes → 42 Prometheus → 44 AWS
```

### 🔴 Security Engineer

```
05 Permissions → 14 Security → 26 SSL → 38 Hardening →
12 VPN → 22 DNS → 09 AD → 13 Monitoring → 43 ELK
```

---

## 🗓️ 4-Week Study Schedule

|Week|Days|Topics|
|---|---|---|
|**Week 1** — Foundation|Day 1–2|01, 02, 03 — OS + Linux CLI|
||Day 3–4|05, 06, 07 — Permissions + Networking|
||Day 5–7|19, 20, 21 — Git complete|
|**Week 2** — Sysadmin|Day 1–2|15, 16, 18 — Services + Troubleshooting|
||Day 3–4|13, 14 — Monitoring + Security|
||Day 5–7|23, 33 — Bash + Regex|
|**Week 3** — Web & Network|Day 1–2|22, 08 — DNS + Networking Tools|
||Day 3–4|25, 26 — Nginx + SSL/TLS|
||Day 5–7|09, 40, 38 — AD + Email + Hardening|
|**Week 4** — DevOps|Day 1–2|30, 27 — Docker + CI/CD|
||Day 3–4|28, 41 — IaC + Kubernetes|
||Day 5–7|42, 43, 44 — Prometheus + ELK + AWS|

---

## ⚡ Quick Reference

### Linux One-Liners

```bash
# Find large files
find / -xdev -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -20

# Top 10 CPU processes
ps aux --sort=-%cpu | head -11

# Check all service statuses
systemctl list-units --type=service --state=failed

# Disk usage sorted
du -sh /* 2>/dev/null | sort -rh | head -20

# Watch logs in real time
journalctl -f --since today

# Active network connections
ss -tulpn | grep LISTEN
```

### Git One-Liners

```bash
# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Delete all local branches that are merged
git branch --merged | grep -v '\*\|main\|master' | xargs git branch -d

# Search all commits for a string
git log -S "function_name" --oneline

# Show what changed in last commit
git show --stat HEAD

# Clean untracked files (dry run first!)
git clean -nd && git clean -fd
```

### Docker One-Liners

```bash
# Remove all stopped containers + unused images
docker system prune -af

# Shell into running container
docker exec -it $(docker ps -qf "name=myapp") bash

# Show logs since 1 hour ago
docker logs --since=1h myapp

# Resource usage live
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### Kubernetes One-Liners

```bash
# Get all resources in namespace
kubectl get all -n production

# Watch pod restarts
kubectl get pods -w --all-namespaces | grep -v Running

# Debug with temporary pod
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# Get events sorted by time
kubectl get events --sort-by='.lastTimestamp' -A
```

---

## 🤝 Contributing

Contributions are welcome! Please:

1. **Fork** the repository
2. **Create** a feature branch: `git switch -c add/topic-name`
3. **Follow** the existing file format and cross-linking style
4. **Include** at least one Mermaid diagram for architecture topics
5. **Add** the new file to `00_INDEX.md`
6. **Submit** a Pull Request with a clear description

### File Naming Convention

```
NN_Topic_Name.md    e.g. 46_Git_Advanced.md
```

### File Template

```markdown
# NN — Topic Name

> **[← Index](00_INDEX.md)** | **Related: [File A](XX_File_A.md) · [File B](XX_File_B.md)**

---

## Section 1

Content...

---

## Related Topics
- [Related Topic ←](XX_Related.md)

---

> [Index](00_INDEX.md)
```

---

## 📄 License

This project is licensed under the **MIT License** — see the LICENSE file for details.

You are free to:

- ✅ Use for personal study and reference
- ✅ Share with colleagues and students
- ✅ Fork and extend for your own needs
- ✅ Use in commercial training material (with attribution)

---

## 🙏 Acknowledgements

These notes draw on knowledge from:

- Official documentation: Linux man pages, Microsoft Docs, AWS Docs, Kubernetes Docs
- Tools: Nginx, Postfix, Elasticsearch, Prometheus, Terraform, Ansible
- Standards: RFC documents, CIS Benchmarks, OWASP, FHS

---

<div align="center">

**⭐ If this helped you, please star the repository!**

_Built with ❤️ for the sysadmin and DevOps community_

</div>