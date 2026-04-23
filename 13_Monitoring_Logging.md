# 13 — System Monitoring and Logging

> **[← VPN](12_VPN.md)** | **[Index](00_INDEX.md)** | **[Security Concepts →](14_Security_Concepts.md)**

---

## Why Monitor Systems?

- **Availability** — detect outages before users do
- **Performance** — find bottlenecks (CPU, RAM, disk, network)
- **Security** — detect unauthorized access, anomalies
- **Capacity planning** — predict when you'll run out of resources
- **Troubleshooting** — diagnose problems after they occur

---

## Linux System Monitoring

### Resource Monitoring Commands

#### CPU and Memory

```bash
# CPU and process monitor
top                     # Interactive process list (press q to quit)
htop                    # Enhanced top (needs install)
atop                    # Advanced resource monitor

# Inside top:
# P = sort by CPU
# M = sort by memory
# k = kill process (enter PID)
# 1 = show per-CPU stats
# q = quit

# CPU info
lscpu                   # CPU architecture details
cat /proc/cpuinfo       # Raw CPU info
nproc                   # Number of CPU cores

# Memory
free -h                 # Memory usage (human readable)
free -m                 # In megabytes
vmstat 1 5              # Virtual memory stats (1s interval, 5 times)
cat /proc/meminfo       # Detailed memory info

# Real-time resource stats
sar -u 1 5              # CPU utilization (1s, 5 samples)
sar -r 1 5              # Memory utilization
sar -n DEV 1 5          # Network interface stats
```

#### Disk Monitoring

```bash
# Disk space
df -h                   # Filesystem usage (human readable)
df -i                   # Inode usage
du -sh /var/log/        # Size of directory
du -sh * | sort -h      # Sort by size
ncdu /var/              # Interactive disk usage

# Disk I/O
iostat -x 1 5           # I/O statistics (extended, 1s, 5 samples)
iotop                   # Interactive I/O per process

# Disk health
smartctl -a /dev/sda    # SMART data (needs smartmontools)
```

#### Network Monitoring

```bash
iftop                   # Interactive bandwidth per connection
nethogs                 # Bandwidth per process
nload eth0              # Network interface load
vnstat                  # Historical bandwidth usage
vnstat -l               # Live monitoring
watch -n 1 'cat /proc/net/dev'   # Raw network counters
```

#### System Load

```bash
uptime                  # Load averages (1, 5, 15 min)
# Example: load average: 0.5, 0.3, 0.2
# Numbers = avg processes in run/wait queue
# Rule: load > CPU count = overloaded

w                       # Uptime + who's logged in
cat /proc/loadavg       # Raw load average
```

---

## Linux Logging — The Log Files

### Log Directory: `/var/log/`

```
/var/log/
├── syslog              # General system messages (Debian/Ubuntu)
├── messages            # General system messages (RHEL/CentOS)
├── auth.log            # Authentication events (Debian/Ubuntu)
├── secure              # Auth events (RHEL/CentOS)
├── kern.log            # Kernel messages
├── dmesg               # Boot-time kernel messages
├── boot.log            # System boot log
├── cron                # Cron job execution
├── mail.log            # Mail server log
├── nginx/
│   ├── access.log      # HTTP access log
│   └── error.log       # HTTP errors
├── mysql/
│   └── error.log       # MySQL errors
├── apache2/
│   ├── access.log
│   └── error.log
└── journal/            # systemd journal (binary format)
```

### Essential Log Viewing Commands

```bash
# View logs
cat /var/log/syslog             # Print entire log
less /var/log/syslog            # Paginated view
tail /var/log/syslog            # Last 10 lines
tail -n 100 /var/log/syslog     # Last 100 lines
tail -f /var/log/syslog         # Follow in real time ⭐
tail -f /var/log/nginx/access.log

# Search in logs
grep "error" /var/log/syslog
grep -i "fail" /var/log/auth.log
grep "Apr 22" /var/log/syslog   # Filter by date

# Show logs from today
grep "$(date '+%b %e')" /var/log/syslog

# View kernel messages
dmesg                           # All kernel messages
dmesg | tail -50                # Last 50
dmesg -T                        # With human-readable timestamps
dmesg --level=err,warn          # Only errors and warnings
dmesg -w                        # Follow in real time
```

---

## systemd Journal (`journalctl`)

Modern Linux systems use `systemd-journald` — a structured, binary log system.

```bash
# Basic usage
journalctl                      # All logs (oldest first)
journalctl -r                   # Reverse (newest first)
journalctl -n 50                # Last 50 entries
journalctl -f                   # Follow (like tail -f)
journalctl --since today
journalctl --since "2024-04-22 10:00:00"
journalctl --since "1 hour ago"
journalctl --until "2024-04-22 12:00:00"

# Filter by unit/service
journalctl -u nginx             # Nginx logs only
journalctl -u nginx -u mysql    # Multiple services
journalctl -u ssh -f            # Follow SSH logs

# Filter by priority
journalctl -p err               # Errors only
journalctl -p warning           # Warnings and above
# Levels: emerg, alert, crit, err, warning, notice, info, debug

# Filter by PID/UID
journalctl _PID=1234
journalctl _UID=1000

# Kernel messages
journalctl -k                   # Kernel messages only
journalctl -k --since "1 hour ago"

# Show boot logs
journalctl -b                   # Current boot
journalctl -b -1                # Previous boot
journalctl --list-boots         # List all boots

# Disk usage
journalctl --disk-usage
journalctl --vacuum-size=500M   # Reduce to 500MB
journalctl --vacuum-time=30d    # Remove entries older than 30 days
```

---

## Log Rotation

**Log rotation** prevents log files from consuming all disk space.

```bash
# logrotate configuration
# /etc/logrotate.conf (global)
# /etc/logrotate.d/ (per-application)

# Example: /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily               # Rotate daily
    missingok           # Don't error if missing
    rotate 14           # Keep 14 old log files
    compress            # Compress rotated logs
    delaycompress       # Don't compress most recent
    notifempty          # Don't rotate empty logs
    create 0640 www-data adm
    sharedscripts
    postrotate
        nginx -s reopen  # Tell nginx to reopen log files
    endscript
}

# Manual rotation
sudo logrotate -f /etc/logrotate.conf      # Force rotation
sudo logrotate -d /etc/logrotate.d/nginx   # Dry run (debug)
```

---

## Windows Event Viewer

### Event Log Categories

| Log | Content |
|-----|---------|
| **Application** | Application-specific events (errors, warnings) |
| **System** | Windows components: drivers, services |
| **Security** | Login/logoff, audit events, privilege use |
| **Setup** | OS/software installation events |
| **Forwarded Events** | Events forwarded from other computers |

### Event Levels

| Level | Icon | Meaning |
|-------|------|---------|
| Information | ℹ️ | Normal operation |
| Warning | ⚠️ | Potential issue |
| Error | ❌ | Failure (non-critical) |
| Critical | 💀 | System-level failure |
| Audit Success | ✅ | Successful security action |
| Audit Failure | 🔒 | Failed security action (e.g., bad password) |

### Important Event IDs

| Event ID | Log | Meaning |
|---------|-----|---------|
| **4624** | Security | Successful logon |
| **4625** | Security | Failed logon (wrong password) |
| **4634** | Security | Logoff |
| **4648** | Security | Logon with explicit credentials |
| **4720** | Security | User account created |
| **4722** | Security | User account enabled |
| **4725** | Security | User account disabled |
| **4740** | Security | Account locked out |
| **4776** | Security | DC validated credentials |
| **7036** | System | Service started/stopped |
| **1102** | Security | Audit log cleared ⚠️ |
| **41** | System | Unexpected shutdown (power loss) |
| **6008** | System | Unexpected shutdown recorded |

```powershell
# PowerShell event log queries
Get-EventLog -LogName Security -InstanceId 4625 -Newest 20    # Failed logins
Get-EventLog -LogName System -EntryType Error -Newest 50       # System errors
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624; StartTime=(Get-Date).AddDays(-1)}

# Filter by time
Get-EventLog -LogName Application -After "2024-04-22" -EntryType Error

# Export to CSV
Get-EventLog -LogName System -Newest 100 | Export-Csv events.csv
```

---

## Performance Monitoring

### Windows Performance Monitor

```powershell
# Get performance counters
Get-Counter "\Processor(_Total)\% Processor Time"
Get-Counter "\Memory\Available MBytes"
Get-Counter "\PhysicalDisk(_Total)\Disk Reads/sec"
Get-Counter "\Network Interface(*)\Bytes Total/sec"

# Continuous monitoring
Get-Counter "\Processor(_Total)\% Processor Time" -Continuous -SampleInterval 2

# Multiple counters
Get-Counter @(
    "\Processor(_Total)\% Processor Time",
    "\Memory\Available MBytes",
    "\LogicalDisk(C:)\% Free Space"
)
```

### Linux: `vmstat`, `sar`, `iostat`

```bash
# vmstat: Virtual Memory Statistics
vmstat 1 10          # 1 second interval, 10 samples
# r = run queue, b = blocked, swpd = swap used
# us = user CPU%, sy = system CPU%, id = idle CPU%
# si/so = swap in/out per second

# sar: System Activity Reporter
sar -u 1 10          # CPU (1s interval, 10 samples)
sar -r 1 10          # Memory
sar -b 1 10          # Disk I/O
sar -n DEV 1 10      # Network per interface
sar -q 1 10          # Load average and run queue

# iostat: I/O Statistics
iostat 1 5           # CPU + disk I/O
iostat -x 1 5        # Extended disk stats
# %util = percentage of time device was busy (>80% = saturated)
# await = average I/O wait time (ms)
```

---

## Centralized Logging

### syslog / rsyslog

```bash
# /etc/rsyslog.conf
# Format: facility.severity  destination

auth,authpriv.*          /var/log/auth.log
*.*;auth,authpriv.none   /var/log/syslog
kern.*                   /var/log/kern.log

# Forward to remote syslog server
*.* @@192.168.1.10:514   # TCP (@@)
*.* @192.168.1.10:514    # UDP (@)
```

---

## Related Topics

- [Linux CLI ←](03_Linux_CLI.md) — `tail`, `grep`, `less`
- [Windows CLI ←](04_Windows_CLI.md) — PowerShell event logs
- [Security Concepts →](14_Security_Concepts.md) — security monitoring
- [Services & Processes →](15_Services_Processes.md) — service logs
- [Troubleshooting →](18_Troubleshooting.md) — using logs to diagnose

---

> [← VPN](12_VPN.md) | [Index](00_INDEX.md) | [Security Concepts →](14_Security_Concepts.md)
