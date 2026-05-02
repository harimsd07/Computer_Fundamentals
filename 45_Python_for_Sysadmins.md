# 45 — Python for Sysadmins & DevOps

> **[← Index](00_INDEX.md)** | **Related: [Bash Scripting](23_Bash_Scripting.md) · [Cloud AWS](44_Cloud_AWS_Deep_Dive.md) · [Docker & Containers](30_Docker_Containers.md) · [CI/CD](27_CICD_Fundamentals.md) · [Database Basics](24_Database_Basics.md)**

---

## Why Python for Sysadmins?

- **Richer than Bash** — proper data structures, OOP, exception handling
- **Cross-platform** — same script runs on Linux, Windows, macOS
- **Huge ecosystem** — boto3 (AWS), paramiko (SSH), fabric (deployment), ansible
- **Better for complex logic** — parsing, APIs, data processing
- **Readable** — easier to maintain than complex shell scripts

---

## Python Environment Setup

```bash
# Check Python version
python3 --version

# Virtual environment (always use for projects)
python3 -m venv venv
source venv/bin/activate          # Linux/macOS
venv\Scripts\activate             # Windows
deactivate                        # Exit venv

# Package management
pip install requests paramiko boto3 fabric psutil
pip install -r requirements.txt
pip freeze > requirements.txt

# Useful sysadmin packages
pip install \
    boto3 \           # AWS SDK
    paramiko \        # SSH client
    fabric \          # Remote execution
    psutil \          # System info (CPU, memory, disk)
    requests \        # HTTP client
    click \           # CLI framework
    rich \            # Beautiful terminal output
    pydantic \        # Data validation
    python-dotenv \   # Load .env files
    schedule \        # Job scheduling
    watchdog \        # File system events
    pyzmq \           # Messaging
    cryptography      # Encryption
```

---

## File System Operations

```python
#!/usr/bin/env python3
"""File system operations for sysadmins."""
import os
import shutil
import pathlib
import tempfile
import hashlib
import stat
from datetime import datetime, timedelta
from pathlib import Path

# ── Path operations ───────────────────────────────────
p = Path("/var/log/nginx")
p.exists()                         # True/False
p.is_file()
p.is_dir()
p.mkdir(parents=True, exist_ok=True)  # mkdir -p

# Read / write files
content = Path("/etc/hosts").read_text()
Path("/tmp/output.txt").write_text("Hello World\n")

# Iterate directory
for f in Path("/var/log").iterdir():
    print(f.name, f.stat().st_size)

# Recursive glob
for log in Path("/var/log").rglob("*.log"):
    print(log)

for conf in Path("/etc").glob("**/*.conf"):
    print(conf)

# ── File info ─────────────────────────────────────────
def file_info(path: str) -> dict:
    p = Path(path)
    s = p.stat()
    return {
        "name":     p.name,
        "size":     s.st_size,
        "size_hr":  f"{s.st_size / 1024 / 1024:.2f} MB",
        "modified": datetime.fromtimestamp(s.st_mtime).isoformat(),
        "created":  datetime.fromtimestamp(s.st_ctime).isoformat(),
        "perms":    oct(stat.S_IMODE(s.st_mode)),
        "owner_uid": s.st_uid,
    }

# ── Copy / move / delete ──────────────────────────────
shutil.copy2("/src/file.txt", "/dst/file.txt")      # Copy with metadata
shutil.copytree("/src/dir", "/dst/dir",              # Recursive copy
    ignore=shutil.ignore_patterns("*.tmp", ".git"))
shutil.move("/src/file.txt", "/dst/file.txt")        # Move
shutil.rmtree("/old/dir")                            # rm -rf
os.remove("/tmp/file.txt")                           # Delete file
os.makedirs("/new/nested/dir", exist_ok=True)        # mkdir -p

# ── Find large files ──────────────────────────────────
def find_large_files(root: str, min_size_mb: float = 100) -> list:
    min_bytes = min_size_mb * 1024 * 1024
    results = []
    for path in Path(root).rglob("*"):
        try:
            if path.is_file() and path.stat().st_size >= min_bytes:
                results.append({
                    "path": str(path),
                    "size_mb": round(path.stat().st_size / 1024 / 1024, 2)
                })
        except PermissionError:
            continue
    return sorted(results, key=lambda x: x["size_mb"], reverse=True)

# ── Find old files ────────────────────────────────────
def find_old_files(directory: str, days: int = 30) -> list:
    cutoff = datetime.now() - timedelta(days=days)
    old_files = []
    for p in Path(directory).rglob("*"):
        if p.is_file():
            mtime = datetime.fromtimestamp(p.stat().st_mtime)
            if mtime < cutoff:
                old_files.append(str(p))
    return old_files

# ── Checksum ──────────────────────────────────────────
def sha256_file(filepath: str) -> str:
    sha256 = hashlib.sha256()
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            sha256.update(chunk)
    return sha256.hexdigest()

# ── Temporary files ───────────────────────────────────
with tempfile.NamedTemporaryFile(mode='w', suffix='.tmp', delete=True) as tmp:
    tmp.write("temporary content")
    print(f"Temp file: {tmp.name}")
# File auto-deleted when context manager exits

with tempfile.TemporaryDirectory() as tmpdir:
    work_path = Path(tmpdir) / "work.txt"
    work_path.write_text("working...")
# Directory auto-deleted
```

---

## Running System Commands

```python
import subprocess
import shlex

# ── subprocess.run (recommended) ──────────────────────
def run_command(cmd: str | list, check: bool = True, timeout: int = 30) -> subprocess.CompletedProcess:
    """Run a shell command safely."""
    if isinstance(cmd, str):
        cmd = shlex.split(cmd)   # Safely tokenize command string

    result = subprocess.run(
        cmd,
        capture_output=True,     # Capture stdout + stderr
        text=True,               # Return str not bytes
        timeout=timeout,
        check=check,             # Raise CalledProcessError on non-zero exit
    )
    return result

# Usage
result = run_command("df -h")
print(result.stdout)
print(result.returncode)

# Handle errors
try:
    result = run_command(["systemctl", "restart", "nginx"])
    print("nginx restarted successfully")
except subprocess.CalledProcessError as e:
    print(f"Failed: {e.stderr}")

# ── Capture output ────────────────────────────────────
def get_ip_addresses() -> list[str]:
    result = subprocess.run(
        ["ip", "-4", "addr", "show"],
        capture_output=True, text=True
    )
    import re
    return re.findall(r'inet\s+(\d+\.\d+\.\d+\.\d+)', result.stdout)

# ── Real-time output streaming ────────────────────────
def run_streaming(cmd: list):
    """Run command and stream output line by line."""
    with subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True,
        bufsize=1
    ) as proc:
        for line in proc.stdout:
            print(line, end='', flush=True)
        proc.wait()
        if proc.returncode != 0:
            raise subprocess.CalledProcessError(proc.returncode, cmd)

run_streaming(["apt", "upgrade", "-y"])
```

---

## System Information with psutil

```python
import psutil
from datetime import datetime

# ── CPU ───────────────────────────────────────────────
psutil.cpu_percent(interval=1)           # Overall %
psutil.cpu_percent(interval=1, percpu=True)  # Per-core
psutil.cpu_count()                       # Logical CPUs
psutil.cpu_count(logical=False)         # Physical cores
psutil.cpu_freq()                        # Frequency

# ── Memory ────────────────────────────────────────────
mem = psutil.virtual_memory()
print(f"Total:     {mem.total / 1024**3:.2f} GB")
print(f"Available: {mem.available / 1024**3:.2f} GB")
print(f"Used:      {mem.percent:.1f}%")

swap = psutil.swap_memory()
print(f"Swap used: {swap.percent:.1f}%")

# ── Disk ──────────────────────────────────────────────
for partition in psutil.disk_partitions():
    try:
        usage = psutil.disk_usage(partition.mountpoint)
        print(f"{partition.mountpoint}: {usage.percent:.1f}% used "
              f"({usage.free / 1024**3:.1f} GB free)")
    except PermissionError:
        pass

io = psutil.disk_io_counters()
print(f"Read:  {io.read_bytes / 1024**2:.1f} MB")
print(f"Write: {io.write_bytes / 1024**2:.1f} MB")

# ── Network ───────────────────────────────────────────
for nic, addrs in psutil.net_if_addrs().items():
    for addr in addrs:
        if addr.family == 2:   # AF_INET = IPv4
            print(f"{nic}: {addr.address}")

net_io = psutil.net_io_counters()
print(f"Sent:     {net_io.bytes_sent / 1024**2:.1f} MB")
print(f"Received: {net_io.bytes_recv / 1024**2:.1f} MB")

# ── Processes ─────────────────────────────────────────
# Top 10 by CPU
processes = []
for proc in psutil.process_iter(['pid', 'name', 'cpu_percent', 'memory_percent']):
    try:
        processes.append(proc.info)
    except (psutil.NoSuchProcess, psutil.AccessDenied):
        pass

top_cpu = sorted(processes, key=lambda p: p['cpu_percent'], reverse=True)[:10]
for p in top_cpu:
    print(f"PID {p['pid']:6} {p['name']:<20} CPU: {p['cpu_percent']:5.1f}% MEM: {p['memory_percent']:5.1f}%")

# Find process by name
def find_process(name: str) -> list:
    return [p for p in psutil.process_iter(['pid', 'name', 'status'])
            if name.lower() in p.info['name'].lower()]

# Kill process
def kill_process(pid: int, force: bool = False):
    try:
        proc = psutil.Process(pid)
        if force:
            proc.kill()    # SIGKILL
        else:
            proc.terminate()  # SIGTERM
            proc.wait(timeout=10)
    except psutil.NoSuchProcess:
        pass

# ── Uptime ────────────────────────────────────────────
boot_time = datetime.fromtimestamp(psutil.boot_time())
uptime = datetime.now() - boot_time
print(f"Uptime: {uptime.days}d {uptime.seconds//3600}h {(uptime.seconds%3600)//60}m")
```

---

## SSH Automation with Paramiko

```python
import paramiko
import socket
from pathlib import Path

class SSHClient:
    def __init__(self, hostname: str, username: str,
                 key_path: str = None, password: str = None, port: int = 22):
        self.hostname = hostname
        self.username = username
        self.port = port
        self.client = paramiko.SSHClient()
        self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

        connect_kwargs = {"hostname": hostname, "username": username, "port": port}
        if key_path:
            connect_kwargs["key_filename"] = str(Path(key_path).expanduser())
        elif password:
            connect_kwargs["password"] = password
        else:
            connect_kwargs["key_filename"] = str(Path("~/.ssh/id_ed25519").expanduser())

        self.client.connect(**connect_kwargs, timeout=10)

    def run(self, command: str, timeout: int = 60) -> tuple[str, str, int]:
        """Execute command, return (stdout, stderr, exit_code)."""
        stdin, stdout, stderr = self.client.exec_command(command, timeout=timeout)
        exit_code = stdout.channel.recv_exit_status()
        return stdout.read().decode(), stderr.read().decode(), exit_code

    def run_sudo(self, command: str, password: str) -> tuple[str, str, int]:
        """Run command with sudo."""
        return self.run(f"echo '{password}' | sudo -S {command}")

    def upload(self, local_path: str, remote_path: str):
        """Upload file via SFTP."""
        sftp = self.client.open_sftp()
        try:
            sftp.put(local_path, remote_path)
        finally:
            sftp.close()

    def download(self, remote_path: str, local_path: str):
        """Download file via SFTP."""
        sftp = self.client.open_sftp()
        try:
            sftp.get(remote_path, local_path)
        finally:
            sftp.close()

    def close(self):
        self.client.close()

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.close()

# Usage
with SSHClient("10.0.0.10", "deploy", key_path="~/.ssh/id_ed25519") as ssh:
    stdout, stderr, code = ssh.run("df -h")
    if code == 0:
        print(stdout)
    else:
        print(f"Error: {stderr}")

    # Upload config
    ssh.upload("./nginx.conf", "/etc/nginx/nginx.conf")

    # Restart service
    stdout, stderr, code = ssh.run("sudo systemctl restart nginx")


# Multi-host execution
def check_disk_usage(hosts: list[str], username: str, threshold: float = 85.0):
    """Check disk usage across multiple servers."""
    alerts = []
    for host in hosts:
        try:
            with SSHClient(host, username) as ssh:
                stdout, _, _ = ssh.run("df -h --output=target,pcent | tail -n +2")
                for line in stdout.strip().splitlines():
                    mount, pct = line.split()
                    usage = float(pct.strip('%'))
                    if usage >= threshold:
                        alerts.append({"host": host, "mount": mount, "usage": usage})
        except Exception as e:
            alerts.append({"host": host, "error": str(e)})
    return alerts
```

---

## HTTP APIs with requests

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import json

# ── Session with retries ──────────────────────────────
def create_session(retries: int = 3, backoff: float = 0.3) -> requests.Session:
    session = requests.Session()
    retry = Retry(
        total=retries,
        backoff_factor=backoff,
        status_forcelist=[429, 500, 502, 503, 504],
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    return session

# ── API client example ────────────────────────────────
class GrafanaClient:
    def __init__(self, base_url: str, api_token: str):
        self.base_url = base_url.rstrip("/")
        self.session = create_session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_token}",
            "Content-Type": "application/json",
        })

    def _request(self, method: str, path: str, **kwargs) -> dict:
        url = f"{self.base_url}/api/{path}"
        response = self.session.request(method, url, timeout=30, **kwargs)
        response.raise_for_status()
        return response.json()

    def get_dashboards(self) -> list:
        return self._request("GET", "search?type=dash-db")

    def get_dashboard(self, uid: str) -> dict:
        return self._request("GET", f"dashboards/uid/{uid}")

    def create_annotation(self, panel_id: int, text: str, tags: list = None):
        return self._request("POST", "annotations", json={
            "panelId": panel_id,
            "text": text,
            "tags": tags or [],
        })

# ── Webhook notifications ─────────────────────────────
def send_slack_alert(webhook_url: str, title: str, message: str,
                     color: str = "danger") -> bool:
    payload = {
        "attachments": [{
            "color": color,
            "title": title,
            "text": message,
            "footer": "SysAdmin Bot",
            "ts": int(__import__('time').time()),
        }]
    }
    try:
        response = requests.post(webhook_url, json=payload, timeout=10)
        return response.status_code == 200
    except requests.RequestException as e:
        print(f"Slack notification failed: {e}")
        return False

def send_teams_alert(webhook_url: str, title: str, message: str) -> bool:
    payload = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": "FF0000",
        "summary": title,
        "sections": [{"activityTitle": title, "activityText": message}]
    }
    response = requests.post(webhook_url, json=payload, timeout=10)
    return response.ok
```

---

## Log Parsing & Analysis

```python
import re
import gzip
import collections
from pathlib import Path
from datetime import datetime
from dataclasses import dataclass, field
from typing import Iterator

@dataclass
class NginxLogEntry:
    ip: str
    timestamp: datetime
    method: str
    path: str
    status: int
    size: int
    referer: str
    user_agent: str

NGINX_PATTERN = re.compile(
    r'(?P<ip>\S+) - \S+ \[(?P<ts>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+) HTTP/[\d.]+" '
    r'(?P<status>\d+) (?P<size>\d+) '
    r'"(?P<referer>[^"]*)" "(?P<ua>[^"]*)"'
)

def parse_nginx_log(filepath: str) -> Iterator[NginxLogEntry]:
    open_fn = gzip.open if filepath.endswith('.gz') else open
    with open_fn(filepath, 'rt', encoding='utf-8', errors='ignore') as f:
        for line in f:
            m = NGINX_PATTERN.match(line)
            if m:
                try:
                    yield NginxLogEntry(
                        ip        = m.group('ip'),
                        timestamp = datetime.strptime(m.group('ts'), '%d/%b/%Y:%H:%M:%S %z'),
                        method    = m.group('method'),
                        path      = m.group('path').split('?')[0],  # strip query string
                        status    = int(m.group('status')),
                        size      = int(m.group('size')),
                        referer   = m.group('referer'),
                        user_agent= m.group('ua'),
                    )
                except (ValueError, KeyError):
                    continue

def analyze_access_log(filepath: str) -> dict:
    entries = list(parse_nginx_log(filepath))
    total = len(entries)
    errors = [e for e in entries if e.status >= 400]
    top_ips = collections.Counter(e.ip for e in entries).most_common(10)
    top_paths = collections.Counter(e.path for e in entries).most_common(10)
    status_dist = collections.Counter(e.status for e in entries)

    return {
        "total_requests":  total,
        "error_count":     len(errors),
        "error_rate_pct":  round(len(errors) / total * 100, 2) if total else 0,
        "top_ips":         top_ips,
        "top_paths":       top_paths,
        "status_dist":     dict(sorted(status_dist.items())),
        "total_bytes":     sum(e.size for e in entries),
    }

if __name__ == "__main__":
    import json
    report = analyze_access_log("/var/log/nginx/access.log")
    print(json.dumps(report, indent=2, default=str))
```

---

## AWS Automation with Boto3

```python
import boto3
import json
from botocore.exceptions import ClientError

# ── Session & clients ─────────────────────────────────
session = boto3.Session(profile_name='production', region_name='ap-south-1')
ec2 = session.client('ec2')
s3  = session.client('s3')
rds = session.client('rds')
ssm = session.client('ssm')

# ── EC2 operations ────────────────────────────────────
def get_instances(filters: list = None) -> list:
    """Get EC2 instances with optional filters."""
    kwargs = {}
    if filters:
        kwargs['Filters'] = filters

    instances = []
    paginator = ec2.get_paginator('describe_instances')
    for page in paginator.paginate(**kwargs):
        for reservation in page['Reservations']:
            for inst in reservation['Instances']:
                name = next((t['Value'] for t in inst.get('Tags', [])
                             if t['Key'] == 'Name'), 'unnamed')
                instances.append({
                    'id':         inst['InstanceId'],
                    'name':       name,
                    'type':       inst['InstanceType'],
                    'state':      inst['State']['Name'],
                    'private_ip': inst.get('PrivateIpAddress', ''),
                    'public_ip':  inst.get('PublicIpAddress', ''),
                    'az':         inst['Placement']['AvailabilityZone'],
                })
    return instances

# ── S3 operations ─────────────────────────────────────
def s3_list_objects(bucket: str, prefix: str = '') -> list:
    """List all objects in S3 bucket (handles pagination)."""
    objects = []
    paginator = s3.get_paginator('list_objects_v2')
    for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
        objects.extend(page.get('Contents', []))
    return objects

def s3_upload_file(local_path: str, bucket: str, key: str,
                   metadata: dict = None) -> bool:
    """Upload file to S3 with optional metadata."""
    extra_args = {}
    if metadata:
        extra_args['Metadata'] = metadata
    try:
        s3.upload_file(local_path, bucket, key, ExtraArgs=extra_args)
        return True
    except ClientError as e:
        print(f"Upload failed: {e}")
        return False

# ── SSM Parameter Store (secret management) ──────────
def get_parameter(name: str, decrypt: bool = True) -> str:
    """Get SSM parameter."""
    response = ssm.get_parameter(Name=name, WithDecryption=decrypt)
    return response['Parameter']['Value']

def put_parameter(name: str, value: str, secure: bool = True):
    """Put SSM parameter."""
    ssm.put_parameter(
        Name=name,
        Value=value,
        Type='SecureString' if secure else 'String',
        Overwrite=True,
    )

# Usage: store/retrieve secrets
put_parameter('/prod/db/password', 'MySecretPass123!')
db_password = get_parameter('/prod/db/password')

# ── RDS snapshots ─────────────────────────────────────
def create_rds_snapshot(instance_id: str) -> str:
    """Create RDS snapshot and return snapshot ID."""
    snapshot_id = f"{instance_id}-{datetime.now().strftime('%Y%m%d%H%M%S')}"
    response = rds.create_db_snapshot(
        DBInstanceIdentifier=instance_id,
        DBSnapshotIdentifier=snapshot_id,
        Tags=[{'Key': 'AutoCreated', 'Value': 'true'}]
    )
    return response['DBSnapshot']['DBSnapshotIdentifier']

def cleanup_old_snapshots(instance_id: str, keep: int = 7):
    """Delete RDS snapshots older than keep count."""
    response = rds.describe_db_snapshots(
        DBInstanceIdentifier=instance_id,
        SnapshotType='manual',
    )
    snapshots = sorted(
        response['DBSnapshots'],
        key=lambda x: x['SnapshotCreateTime'],
        reverse=True
    )
    for snapshot in snapshots[keep:]:
        print(f"Deleting: {snapshot['DBSnapshotIdentifier']}")
        rds.delete_db_snapshot(
            DBSnapshotIdentifier=snapshot['DBSnapshotIdentifier']
        )
```

---

## Practical Sysadmin Scripts

### Health Check Dashboard (Terminal)

```python
#!/usr/bin/env python3
"""System health dashboard — run with: watch -n 5 python3 health.py"""
import psutil, subprocess, datetime
from pathlib import Path

def banner(text: str):
    print(f"\n{'='*50}")
    print(f"  {text}")
    print('='*50)

def check_services(services: list[str]) -> list[dict]:
    results = []
    for svc in services:
        r = subprocess.run(["systemctl", "is-active", svc],
                          capture_output=True, text=True)
        results.append({
            "name": svc,
            "status": r.stdout.strip(),
            "ok": r.returncode == 0
        })
    return results

print(f"\n{'='*50}")
print(f"  System Health — {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
print('='*50)

# CPU
cpu = psutil.cpu_percent(interval=1)
cpu_icon = "🔴" if cpu > 85 else "🟡" if cpu > 70 else "🟢"
print(f"{cpu_icon} CPU:    {cpu:5.1f}%  ({'!' if cpu > 85 else 'OK'})")

# Memory
mem = psutil.virtual_memory()
mem_icon = "🔴" if mem.percent > 90 else "🟡" if mem.percent > 80 else "🟢"
print(f"{mem_icon} Memory: {mem.percent:5.1f}%  ({mem.available/1024**3:.1f} GB free)")

# Disk
banner("Disk Usage")
for part in psutil.disk_partitions():
    try:
        u = psutil.disk_usage(part.mountpoint)
        icon = "🔴" if u.percent > 90 else "🟡" if u.percent > 80 else "🟢"
        bar_len = int(u.percent / 5)
        bar = "█" * bar_len + "░" * (20 - bar_len)
        print(f"{icon} {part.mountpoint:<15} [{bar}] {u.percent:5.1f}%  {u.free/1024**3:.1f}GB free")
    except PermissionError:
        pass

# Services
banner("Services")
for svc in check_services(["nginx", "mysql", "ssh", "docker"]):
    icon = "✅" if svc["ok"] else "❌"
    print(f"{icon} {svc['name']:<15} {svc['status']}")

# Top processes
banner("Top Processes (CPU)")
procs = sorted(
    [p.info for p in psutil.process_iter(['pid','name','cpu_percent','memory_percent'])
     if p.info['cpu_percent'] is not None],
    key=lambda x: x['cpu_percent'], reverse=True
)[:5]
for p in procs:
    print(f"  PID {p['pid']:6}  {p['name']:<20}  CPU: {p['cpu_percent']:5.1f}%  MEM: {p['memory_percent']:4.1f}%")
```

### Automated Deployment Script

```python
#!/usr/bin/env python3
"""Deploy application to multiple servers via SSH."""
import argparse
import logging
from pathlib import Path
import paramiko

logging.basicConfig(level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s")
log = logging.getLogger(__name__)

SERVERS = {
    "staging":    ["10.0.1.10"],
    "production": ["10.0.1.20", "10.0.1.21", "10.0.1.22"],
}

DEPLOY_COMMANDS = [
    "cd /var/www/myapp",
    "git fetch origin",
    "git checkout {version}",
    "composer install --no-dev --optimize-autoloader",
    "php artisan migrate --force",
    "php artisan config:cache",
    "php artisan route:cache",
    "sudo systemctl reload php8.2-fpm",
    "sudo systemctl reload nginx",
]

def deploy_to_server(host: str, version: str, ssh_key: str) -> bool:
    log.info(f"Deploying {version} to {host}")
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        client.connect(host, username="deploy", key_filename=ssh_key, timeout=30)
        for cmd_template in DEPLOY_COMMANDS:
            cmd = cmd_template.format(version=version)
            log.info(f"  [{host}] {cmd}")
            _, stdout, stderr = client.exec_command(cmd, timeout=120)
            exit_code = stdout.channel.recv_exit_status()
            if exit_code != 0:
                log.error(f"  [{host}] FAILED: {stderr.read().decode()}")
                return False
        log.info(f"✅ {host} deployed successfully")
        return True
    except Exception as e:
        log.error(f"❌ {host} failed: {e}")
        return False
    finally:
        client.close()

def main():
    parser = argparse.ArgumentParser(description="Deploy application")
    parser.add_argument("--env", choices=SERVERS.keys(), required=True)
    parser.add_argument("--version", required=True, help="Git tag or branch")
    parser.add_argument("--key", default="~/.ssh/id_ed25519")
    args = parser.parse_args()

    hosts = SERVERS[args.env]
    key = str(Path(args.key).expanduser())
    results = {}

    for host in hosts:
        results[host] = deploy_to_server(host, args.version, key)

    success = all(results.values())
    log.info(f"\nDeployment {'SUCCEEDED' if success else 'FAILED'}")
    for host, ok in results.items():
        log.info(f"  {'✅' if ok else '❌'} {host}")

    return 0 if success else 1

if __name__ == "__main__":
    import sys
    sys.exit(main())
```

---

## Related Topics

- [Bash Scripting ←](23_Bash_Scripting.md) — shell automation
- [Cloud AWS ←](44_Cloud_AWS_Deep_Dive.md) — boto3 use cases
- [Database Basics ←](24_Database_Basics.md) — Python DB drivers
- [Docker & Containers ←](30_Docker_Containers.md) — Docker SDK for Python
- [CI/CD ←](27_CICD_Fundamentals.md) — Python in pipelines
- [Monitoring Prometheus ←](42_Monitoring_Prometheus_Grafana.md) — prom-client

---

> [Index](00_INDEX.md)
