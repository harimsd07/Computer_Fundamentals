# 38 — Linux System Hardening

> **[← Index](00_INDEX.md)** | **Related: [Security Concepts](14_Security_Concepts.md) · [User Permissions](05_Permissions.md) · [SSH](17_Cloud_Remote_Access.md) · [Monitoring & Logging](13_Monitoring_Logging.md) · [Firewall](14_Security_Concepts.md)**

---

## Hardening Philosophy

```
Principle of Least Privilege     → Give only what's needed
Defense in Depth                 → Multiple independent layers
Reduce Attack Surface            → Remove everything not required
Fail Securely                    → Default-deny, not default-allow
Keep It Simple                   → Complex configs introduce bugs
Audit Everything                 → You can't secure what you can't see
```

---

## Initial Server Setup Checklist

```bash
# ── 1. Update all packages immediately ───────────────
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y

# ── 2. Set hostname ───────────────────────────────────
sudo hostnamectl set-hostname prod-web-01
sudo nano /etc/hosts   # Add: 127.0.1.1  prod-web-01

# ── 3. Set timezone ───────────────────────────────────
sudo timedatectl set-timezone Asia/Kolkata
sudo timedatectl set-ntp true

# ── 4. Create non-root admin user ─────────────────────
sudo useradd -m -s /bin/bash -G sudo deploy
sudo passwd deploy
# Or with SSH key (preferred):
sudo mkdir -p /home/deploy/.ssh
sudo cp ~/.ssh/authorized_keys /home/deploy/.ssh/
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

---

## SSH Hardening

```bash
# /etc/ssh/sshd_config — hardened configuration

# Change default port (reduces automated scans)
Port 2222

# Disable root login completely
PermitRootLogin no

# Key auth only — DISABLE password auth after copying keys!
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Disable dangerous authentication methods
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no
UsePAM yes

# Restrict who can SSH
AllowUsers deploy alice
# or
AllowGroups sshusers

# Limit login attempts
MaxAuthTries 3
MaxSessions 5

# Disconnect idle sessions (10 min)
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable X11 and TCP forwarding (unless needed)
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no

# Use strong ciphers only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org

# Log level
LogLevel VERBOSE

# Banner (legal warning)
Banner /etc/ssh/banner.txt

# Apply
sudo sshd -t                  # Test config
sudo systemctl restart sshd
```

```bash
# SSH banner content
cat > /etc/ssh/banner.txt << 'EOF'
*************************************************************
WARNING: Authorized access only. All activity is monitored.
Unauthorized access is a criminal offence.
*************************************************************
EOF
```

### Fail2Ban — Block Brute Force

```bash
# Install
sudo apt install fail2ban

# /etc/fail2ban/jail.local
[DEFAULT]
bantime  = 3600       # Ban for 1 hour
findtime = 600        # Window to count failures (10 min)
maxretry = 3          # Ban after 3 failures
banaction = iptables-multiport
backend = systemd

[sshd]
enabled  = true
port     = 2222       # Match your SSH port
logpath  = %(sshd_log)s

[nginx-http-auth]
enabled = true
port    = http,https
logpath = %(nginx_error_log)s

[nginx-limit-req]
enabled  = true
port     = http,https
logpath  = %(nginx_error_log)s

# Commands
sudo systemctl enable --now fail2ban
sudo fail2ban-client status             # All jails
sudo fail2ban-client status sshd        # Specific jail
sudo fail2ban-client unban 1.2.3.4      # Unban IP
sudo fail2ban-client set sshd banip 1.2.3.4  # Manually ban
```

---

## User & Password Policy

```bash
# ── Password aging ────────────────────────────────────
# /etc/login.defs
PASS_MAX_DAYS   90         # Force change every 90 days
PASS_MIN_DAYS   1          # Can't change again within 1 day
PASS_WARN_AGE   14         # Warn 14 days before expiry
PASS_MIN_LEN    12         # Minimum length

# Apply to existing user
sudo chage -M 90 -m 1 -W 14 alice
sudo chage -l alice        # View current settings
sudo chage -E 2025-01-01 alice  # Account expiry date

# ── Password strength (PAM) ───────────────────────────
sudo apt install libpam-pwquality

# /etc/security/pwquality.conf
minlen = 14          # Minimum length
minclass = 3         # Require 3 of: upper, lower, digit, special
maxrepeat = 3        # No more than 3 same chars in a row
dcredit = -1         # At least 1 digit
ucredit = -1         # At least 1 uppercase
lcredit = -1         # At least 1 lowercase
ocredit = -1         # At least 1 special char
difok = 5            # 5 chars must differ from old password

# /etc/pam.d/common-password
password requisite pam_pwquality.so retry=3

# ── Lock account after failed logins (PAM tally) ──────
# /etc/pam.d/common-auth  (add at top)
auth required pam_tally2.so deny=5 unlock_time=900 onerr=fail audit

# Check tally
sudo pam_tally2 --user alice
sudo pam_tally2 --reset --user alice   # Manually unlock

# ── Sudo hardening ────────────────────────────────────
sudo visudo    # Always use visudo, never edit sudoers directly

# Good sudoers entries:
# Require password for every sudo (not cached)
Defaults timestamp_timeout=0

# Log all sudo commands
Defaults logfile=/var/log/sudo.log
Defaults log_input, log_output

# Restrict to specific commands only
deploy ALL=(root) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl restart php8.2-fpm
alice  ALL=(root) /usr/bin/apt update, /usr/bin/apt upgrade

# No sudo from SSH without tty
Defaults requiretty
```

---

## Filesystem Hardening

```bash
# ── /tmp mount options ────────────────────────────────
# /etc/fstab — add noexec,nosuid,nodev to /tmp
tmpfs   /tmp   tmpfs   defaults,noexec,nosuid,nodev,size=1G   0 0
tmpfs   /var/tmp tmpfs defaults,noexec,nosuid,nodev,size=500M 0 0

# Apply without reboot
sudo mount -o remount /tmp

# ── Set immutable flag on critical files ──────────────
sudo chattr +i /etc/passwd
sudo chattr +i /etc/shadow
sudo chattr +i /etc/sudoers
sudo chattr +i /etc/hosts

# View attributes
lsattr /etc/passwd
# Remove immutable (must do to make changes)
sudo chattr -i /etc/passwd

# ── File permission audit ─────────────────────────────
# Find world-writable files (security risk)
find / -xdev -type f -perm -0002 -ls 2>/dev/null

# Find SUID/SGID files (review these carefully)
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -ls 2>/dev/null

# Find files with no owner
find / -xdev \( -nouser -o -nogroup \) -ls 2>/dev/null

# Secure /etc permissions
chmod 644 /etc/passwd
chmod 000 /etc/shadow     # Only root can read
chmod 644 /etc/group
chmod 600 /etc/ssh/sshd_config
```

---

## Kernel Hardening (sysctl)

```bash
# /etc/sysctl.d/99-hardening.conf

# ── Network security ──────────────────────────────────
# Ignore ICMP redirects (prevent routing attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Don't send ICMP redirects
net.ipv4.conf.all.send_redirects = 0

# Ignore broadcast pings (smurf attack prevention)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Enable SYN cookies (SYN flood protection)
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048

# Disable IP source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Enable reverse path filtering (anti-spoofing)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Log martian packets (spoofed/impossible addresses)
net.ipv4.conf.all.log_martians = 1

# ── Memory security ───────────────────────────────────
# Randomize memory layout (ASLR)
kernel.randomize_va_space = 2

# Restrict core dumps
fs.suid_dumpable = 0

# Restrict dmesg access to root
kernel.dmesg_restrict = 1

# Restrict kernel pointers in /proc (prevent info leaks)
kernel.kptr_restrict = 2

# Restrict ptrace (prevent process inspection by non-root)
kernel.yama.ptrace_scope = 1

# ── Apply ─────────────────────────────────────────────
sudo sysctl -p /etc/sysctl.d/99-hardening.conf
sudo sysctl --system    # Apply all sysctl configs
```

---

## AppArmor & SELinux — Mandatory Access Control

### AppArmor (Ubuntu/Debian default)

```bash
# Check status
sudo aa-status
sudo apparmor_status

# Modes
# enforce = actively blocks violations
# complain = logs violations but allows them (learning mode)

# Switch modes
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx

# Generate profile for an application (learning mode)
sudo aa-genprof /usr/local/bin/myapp
# Run app, do all operations, then:
# Press S (scan) → A (allow) or D (deny) for each event
# Press F to finish

# Reload profiles
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
sudo systemctl reload apparmor

# Check denials in log
sudo journalctl -f | grep apparmor
sudo grep "DENIED" /var/log/syslog
```

### SELinux (RHEL/CentOS/Fedora)

```bash
# Check status
sestatus
getenforce                    # Enforcing / Permissive / Disabled

# Set mode (temporary)
setenforce 1                  # Enforcing
setenforce 0                  # Permissive (for debugging)

# Permanent: /etc/selinux/config
SELINUX=enforcing             # enforcing / permissive / disabled

# Check SELinux denials
ausearch -m avc -ts recent
journalctl -t setroubleshoot

# Allow a denied operation
audit2why < /var/log/audit/audit.log      # Explain why denied
audit2allow -a -M mypolicy               # Generate allow module
semodule -i mypolicy.pp                  # Install module

# File contexts
ls -Z /var/www/html/          # Show SELinux labels
restorecon -Rv /var/www/html/ # Restore default labels
chcon -t httpd_sys_content_t /var/www/html/myfile.html  # Set label

# Port contexts
semanage port -l | grep http  # List HTTP-allowed ports
semanage port -a -t http_port_t -p tcp 8080  # Allow port 8080 for httpd
```

---

## Audit Framework (auditd)

```bash
# Install
sudo apt install auditd audispd-plugins

# Start
sudo systemctl enable --now auditd

# ── Audit rules (/etc/audit/rules.d/hardening.rules) ──
# Watch sensitive files
-w /etc/passwd -p wa -k passwd-changes
-w /etc/shadow -p wa -k shadow-changes
-w /etc/sudoers -p wa -k sudoers-changes
-w /etc/ssh/sshd_config -p wa -k sshd-config

# Watch for privilege escalation
-a always,exit -F arch=b64 -S setuid -S setgid -k privilege-escalation

# Log all sudo usage
-w /usr/bin/sudo -p x -k sudo-exec

# Monitor /tmp execution (malware)
-a always,exit -F dir=/tmp -F perm=x -k tmp-exec

# Failed logins
-a always,exit -F arch=b64 -S open -F exit=-EACCES -k access-denied

# Apply rules
sudo augenrules --load
sudo systemctl restart auditd

# ── Query audit logs ──────────────────────────────────
sudo ausearch -k passwd-changes          # Search by key
sudo ausearch -k passwd-changes --start today
sudo ausearch -ua alice                  # By user
sudo ausearch -m LOGIN --start today     # Login events
sudo aureport -au                        # Authentication report
sudo aureport -l                         # Login report
sudo aureport --failed                   # Failed events
```

---

## Automatic Security Updates

```bash
# Ubuntu — unattended-upgrades
sudo apt install unattended-upgrades

# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";   # Don't auto-reboot
Unattended-Upgrade::Mail "admin@example.com";

# /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";

sudo systemctl enable unattended-upgrades
sudo unattended-upgrades --dry-run  # Test
```

---

## Security Scanning Tools

```bash
# ── Lynis — System audit ──────────────────────────────
sudo apt install lynis
sudo lynis audit system              # Full audit
sudo lynis audit system --quick      # Quick scan
# Generates hardening index + recommendations

# ── ClamAV — Antivirus ────────────────────────────────
sudo apt install clamav clamav-daemon
sudo freshclam                       # Update virus definitions
sudo clamscan -r /home               # Scan home dirs
sudo clamscan -r / --exclude-dir=/proc --exclude-dir=/sys  # Full scan
sudo systemctl enable --now clamav-daemon

# ── Rootkit hunters ───────────────────────────────────
sudo apt install rkhunter chkrootkit

sudo rkhunter --update               # Update database
sudo rkhunter --check                # Scan for rootkits
sudo rkhunter --check --report-warnings-only

sudo chkrootkit                      # Quick rootkit scan

# ── OpenSCAP — Compliance scanning ───────────────────
sudo apt install openscap-scanner scap-security-guide
sudo oscap xccdf eval \
    --profile xccdf_org.ssgproject.content_profile_cis \
    --results scan-results.xml \
    --report scan-report.html \
    /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

---

## Quick Hardening Checklist

```
✓ Update all packages immediately after provisioning
✓ Create non-root sudo user; disable root SSH login
✓ SSH: key auth only; disable password auth; change port
✓ Install and configure Fail2Ban
✓ Configure UFW/iptables: default deny inbound
✓ Enable automatic security updates
✓ Set strong password policy (PAM)
✓ Restrict /tmp with noexec,nosuid,nodev
✓ Apply kernel hardening via sysctl
✓ Enable auditd with rules for sensitive files
✓ Enable AppArmor or SELinux
✓ Remove unused packages and services
✓ Disable unused kernel modules
✓ Run Lynis audit and fix findings
✓ Set up centralized log shipping
✓ Regular rootkit scans (rkhunter)
```

---

## Related Topics

- [Security Concepts ←](14_Security_Concepts.md) — auth, encryption, firewalls
- [User Permissions ←](05_Permissions.md) — Linux permissions model
- [Cloud & Remote Access ←](17_Cloud_Remote_Access.md) — SSH keys
- [Monitoring & Logging ←](13_Monitoring_Logging.md) — audit logs
- [Bash Scripting ←](23_Bash_Scripting.md) — automation scripts

---

> [Index](00_INDEX.md)
