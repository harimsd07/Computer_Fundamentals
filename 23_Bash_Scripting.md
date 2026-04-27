# 23 — Bash Scripting

> **[← Index](00_INDEX.md)** | **Related: [Linux CLI](03_Linux_CLI.md) · [Services & Processes](15_Services_Processes.md) · [Troubleshooting](18_Troubleshooting.md)**

---

## Why Bash Scripting?

- **Automate repetitive tasks** — backups, log rotation, user creation
- **System administration** — server setup, deployment scripts
- **Scheduled jobs** — cron-driven maintenance
- **Glue code** — chain CLI tools together
- No compilation, runs everywhere Linux runs

---

## Script Structure

```bash
#!/usr/bin/env bash
# ─────────────────────────────────────────────────────
# Script:  backup.sh
# Purpose: Daily backup of /var/www to /backup/
# Author:  Alice <alice@example.com>
# Date:    2024-04-22
# Usage:   ./backup.sh [--dry-run]
# ─────────────────────────────────────────────────────

set -euo pipefail           # Safe mode (see below)
IFS=$'\n\t'                 # Safer word splitting

# Constants
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/backup.log"
readonly TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')

# ── Functions ─────────────────────────────────────────
log() { echo "[$(date '+%H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
die() { log "ERROR: $*" >&2; exit 1; }

# ── Main ──────────────────────────────────────────────
main() {
    log "Backup started"
    # script logic here
    log "Backup completed"
}

main "$@"
```

### Safe Mode Flags

```bash
set -e          # Exit immediately on error (errexit)
set -u          # Treat unset variables as errors (nounset)
set -o pipefail # Pipe fails if any command in pipe fails
set -x          # Print each command before executing (debug)
set -euo pipefail  # All three combined (recommended)
```

---

## Variables

```bash
# Assignment (no spaces around =)
NAME="Alice"
AGE=30
PI=3.14
EMPTY=""

# Using variables
echo "$NAME"          # Always quote variables
echo "${NAME}s"       # Curly braces for adjacent text → "Alices"
echo "$NAME $AGE"

# Command substitution
TODAY=$(date '+%Y-%m-%d')
HOSTNAME=$(hostname)
FILE_COUNT=$(ls /etc/ | wc -l)

# Arithmetic
RESULT=$((5 + 3))
COUNT=$((COUNT + 1))
let COUNT++
((COUNT++))           # C-style

# Read-only (constant)
readonly MAX_RETRIES=3

# Arrays
FRUITS=("apple" "banana" "cherry")
echo "${FRUITS[0]}"          # apple
echo "${FRUITS[@]}"          # All elements
echo "${#FRUITS[@]}"         # Array length
FRUITS+=("date")             # Append

# Associative arrays (bash 4+)
declare -A COLORS
COLORS["sky"]="blue"
COLORS["grass"]="green"
echo "${COLORS["sky"]}"
echo "${!COLORS[@]}"         # All keys
echo "${COLORS[@]}"          # All values
```

### Variable Default Values

```bash
# Use default if variable is unset or empty
echo "${NAME:-"World"}"          # Use "World" if NAME unset
echo "${PORT:-8080}"             # Default port

# Assign default if unset
: "${CONFIG_FILE:=/etc/app/config.yml}"
: "${LOG_LEVEL:=INFO}"

# Error if unset
echo "${REQUIRED_VAR:?Variable REQUIRED_VAR must be set}"

# Trim from end / beginning
FILE="backup_2024-04-22.tar.gz"
echo "${FILE%.tar.gz}"           # Remove suffix → backup_2024-04-22
echo "${FILE#backup_}"           # Remove prefix → 2024-04-22.tar.gz
echo "${FILE^^}"                 # Uppercase
echo "${FILE,,}"                 # Lowercase
echo "${FILE/2024/2025}"         # Replace first match
echo "${FILE//20/XX}"            # Replace all matches
```

---

## Conditionals

```bash
# if / elif / else
if [[ "$NAME" == "Alice" ]]; then
    echo "Hello Alice"
elif [[ "$NAME" == "Bob" ]]; then
    echo "Hello Bob"
else
    echo "Hello stranger"
fi

# Test operators — [[ ]] (preferred over [ ])
# String comparisons
[[ "$a" == "$b" ]]      # equal
[[ "$a" != "$b" ]]      # not equal
[[ "$a" < "$b" ]]       # less than (lexicographic)
[[ -z "$a" ]]           # empty string
[[ -n "$a" ]]           # non-empty string
[[ "$a" =~ ^[0-9]+$ ]]  # regex match

# Numeric comparisons (use (( )) or -eq etc.)
[[ "$a" -eq "$b" ]]     # equal
[[ "$a" -ne "$b" ]]     # not equal
[[ "$a" -gt "$b" ]]     # greater than
[[ "$a" -lt "$b" ]]     # less than
[[ "$a" -ge "$b" ]]     # greater or equal
(( a > b ))             # arithmetic test

# File tests
[[ -f "$file" ]]        # is a regular file
[[ -d "$dir" ]]         # is a directory
[[ -e "$path" ]]        # exists (any type)
[[ -r "$file" ]]        # readable
[[ -w "$file" ]]        # writable
[[ -x "$file" ]]        # executable
[[ -s "$file" ]]        # non-empty file
[[ -L "$path" ]]        # is symbolic link
[[ "$f1" -nt "$f2" ]]   # f1 newer than f2

# Logical operators
[[ "$a" == "x" && "$b" == "y" ]]   # AND
[[ "$a" == "x" || "$b" == "y" ]]   # OR
[[ ! -f "$file" ]]                  # NOT

# Short-circuit operators
command && echo "success"       # Run if command succeeded
command || echo "failed"        # Run if command failed
command || { log "fail"; exit 1; }   # Multiple commands on failure
```

### `case` Statement

```bash
case "$ENVIRONMENT" in
    production|prod)
        LOG_LEVEL="ERROR"
        DEBUG=false
        ;;
    staging|stage)
        LOG_LEVEL="WARN"
        DEBUG=false
        ;;
    development|dev)
        LOG_LEVEL="DEBUG"
        DEBUG=true
        ;;
    *)
        echo "Unknown environment: $ENVIRONMENT"
        exit 1
        ;;
esac
```

---

## Loops

```bash
# for loop — list
for name in Alice Bob Charlie; do
    echo "Hello, $name"
done

# for loop — files
for file in /var/log/*.log; do
    echo "Processing: $file"
    gzip "$file"
done

# for loop — C style
for ((i = 0; i < 10; i++)); do
    echo "Item $i"
done

# for loop — range
for i in {1..10}; do echo "$i"; done
for i in {0..100..5}; do echo "$i"; done   # 0 5 10 ... 100

# while loop
COUNT=0
while [[ $COUNT -lt 5 ]]; do
    echo "Count: $COUNT"
    ((COUNT++))
done

# while — read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# while — read command output
while IFS= read -r user; do
    echo "User: $user"
done < <(getent passwd | cut -d: -f1)

# until loop (opposite of while)
until [[ -f /tmp/ready ]]; do
    echo "Waiting..."
    sleep 5
done

# Loop control
for i in {1..10}; do
    [[ $i -eq 5 ]] && continue    # Skip 5
    [[ $i -eq 8 ]] && break       # Stop at 8
    echo "$i"
done
```

---

## Functions

```bash
# Define function
greet() {
    local name="$1"          # local scope (best practice)
    local greeting="${2:-Hello}"
    echo "$greeting, $name!"
}

# Call
greet "Alice"
greet "Bob" "Hi"

# Return values
# Exit code (0=success, 1-255=error)
is_root() {
    [[ "$EUID" -eq 0 ]]      # Returns 0 if root, 1 otherwise
}

if is_root; then
    echo "Running as root"
fi

# Return data via echo (capture with $())
get_timestamp() {
    echo "$(date '+%Y-%m-%d_%H-%M-%S')"
}
TS=$(get_timestamp)

# Return data via global variable (avoid when possible)
parse_config() {
    CONFIG_HOST="localhost"
    CONFIG_PORT=5432
}
parse_config
echo "$CONFIG_HOST:$CONFIG_PORT"

# Practical function pattern
check_command() {
    local cmd="$1"
    if ! command -v "$cmd" &>/dev/null; then
        die "Required command not found: $cmd"
    fi
}

require_root() {
    if [[ "$EUID" -ne 0 ]]; then
        die "This script must be run as root"
    fi
}

cleanup() {
    # Always runs on exit (trap)
    rm -f /tmp/lockfile.$$
    log "Script finished"
}
trap cleanup EXIT
trap 'die "Script interrupted"' INT TERM
```

---

## Input / Arguments

```bash
# Positional parameters
$0   # Script name
$1   # First argument
$2   # Second argument
$@   # All arguments (as separate words)
$*   # All arguments (as single string)
$#   # Number of arguments

# Example usage
if [[ $# -lt 2 ]]; then
    echo "Usage: $0 <source> <destination>"
    exit 1
fi

SOURCE="$1"
DEST="$2"

# Shift (consume arguments one by one)
while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose)   VERBOSE=true; shift ;;
        -n|--dry-run)   DRY_RUN=true; shift ;;
        -f|--file)      FILE="$2"; shift 2 ;;
        -h|--help)      usage; exit 0 ;;
        --)             shift; break ;;
        -*)             die "Unknown option: $1" ;;
        *)              ARGS+=("$1"); shift ;;
    esac
done

# Interactive input
read -p "Enter your name: " USER_NAME
read -s -p "Enter password: " PASSWORD   # -s = silent
echo ""

# Select menu
echo "Choose environment:"
select ENV in development staging production; do
    echo "Selected: $ENV"
    break
done
```

---

## Error Handling

```bash
# Check exit codes
if ! cp "$SRC" "$DEST"; then
    die "Failed to copy $SRC to $DEST"
fi

# Capture exit code
cp "$SRC" "$DEST"
EXIT_CODE=$?
if [[ $EXIT_CODE -ne 0 ]]; then
    log "Copy failed with exit code: $EXIT_CODE"
fi

# Try/catch pattern
run_with_retry() {
    local max_attempts=3
    local attempt=1
    local delay=5

    while [[ $attempt -le $max_attempts ]]; do
        if "$@"; then
            return 0
        fi
        log "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
        ((delay *= 2))   # Exponential backoff
    done

    die "All $max_attempts attempts failed for: $*"
}

run_with_retry curl -f https://api.example.com/health

# Trap errors
trap 'echo "Error on line $LINENO"' ERR

# ERR trap with set -e
set -e
trap 'echo "Script failed at line $LINENO with exit code $?"' ERR
```

---

## String Operations

```bash
STR="Hello, World!"

# Length
echo "${#STR}"                  # 13

# Substring
echo "${STR:7}"                 # World!
echo "${STR:7:5}"               # World

# Replace
echo "${STR/World/Bash}"        # Hello, Bash!    (first)
echo "${STR//l/L}"              # HeLLo, WorLd!   (all)

# Upper/Lower case
echo "${STR^^}"                 # HELLO, WORLD!
echo "${STR,,}"                 # hello, world!

# Trim pattern from start/end
FILE="backup_2024.tar.gz"
echo "${FILE#*_}"               # Remove shortest match from start
echo "${FILE##*_}"              # Remove longest match from start
echo "${FILE%.tar.gz}"          # Remove from end
echo "${FILE%.*}"               # Remove extension

# Split string into array
IFS=',' read -ra PARTS <<< "a,b,c,d"
echo "${PARTS[0]}"              # a
echo "${PARTS[@]}"              # a b c d

# Join array
printf '%s,' "${PARTS[@]}"     # a,b,c,d,
```

---

## File Operations in Scripts

```bash
# Read entire file into variable
CONTENT=$(cat /etc/hostname)

# Read file line by line (safe for spaces)
while IFS= read -r line; do
    echo ">> $line"
done < /etc/hosts

# Write to file
echo "content" > file.txt            # Overwrite
echo "content" >> file.txt           # Append
cat > file.txt << 'EOF'             # Heredoc
line 1
line 2
EOF

# Temp files (secure)
TMPFILE=$(mktemp /tmp/script.XXXXXX)
TMPDIR=$(mktemp -d /tmp/work.XXXXXX)
trap 'rm -rf "$TMPFILE" "$TMPDIR"' EXIT

# Check and create directories
[[ -d "$BACKUP_DIR" ]] || mkdir -p "$BACKUP_DIR"

# Find and process files
find /var/log -name "*.log" -mtime +30 | while IFS= read -r logfile; do
    log "Archiving: $logfile"
    gzip "$logfile"
done
```

---

## Practical Script Examples

### System Health Check

```bash
#!/usr/bin/env bash
set -euo pipefail

WARN_CPU=80
WARN_MEM=85
WARN_DISK=90

check_cpu() {
    local usage
    usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
    echo "CPU: ${usage}%"
    (( $(echo "$usage > $WARN_CPU" | bc -l) )) && echo "⚠️  CPU HIGH"
}

check_memory() {
    local usage
    usage=$(free | awk '/Mem/{printf("%.0f", $3/$2*100)}')
    echo "RAM: ${usage}%"
    (( usage > WARN_MEM )) && echo "⚠️  MEMORY HIGH"
}

check_disk() {
    df -h | awk 'NR>1 {
        gsub(/%/,"",$5)
        if ($5+0 > '"$WARN_DISK"')
            print "⚠️  DISK HIGH: "$6" at "$5"%"
        else
            print "OK: "$6" at "$5"%"
    }'
}

check_services() {
    local services=("nginx" "mysql" "ssh")
    for svc in "${services[@]}"; do
        if systemctl is-active --quiet "$svc"; then
            echo "✓ $svc running"
        else
            echo "✗ $svc NOT running"
        fi
    done
}

echo "=== System Health Check: $(date) ==="
check_cpu
check_memory
check_disk
check_services
```

### Backup Script

```bash
#!/usr/bin/env bash
set -euo pipefail

SOURCE="/var/www"
DEST="/backup/www"
RETENTION_DAYS=30
TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')
BACKUP_NAME="www_${TIMESTAMP}.tar.gz"

log() { echo "[$(date '+%H:%M:%S')] $*"; }
die() { log "ERROR: $*" >&2; exit 1; }

[[ -d "$SOURCE" ]] || die "Source not found: $SOURCE"
mkdir -p "$DEST"

log "Starting backup of $SOURCE"
tar -czf "${DEST}/${BACKUP_NAME}" -C "$(dirname "$SOURCE")" "$(basename "$SOURCE")"
log "Backup created: ${DEST}/${BACKUP_NAME}"

# Remove old backups
log "Removing backups older than $RETENTION_DAYS days"
find "$DEST" -name "www_*.tar.gz" -mtime +"$RETENTION_DAYS" -delete
log "Old backups removed"

# List current backups
log "Current backups:"
ls -lh "$DEST"/*.tar.gz 2>/dev/null || log "No backups found"
```

### User Creation Script

```bash
#!/usr/bin/env bash
set -euo pipefail

[[ $EUID -eq 0 ]] || { echo "Must run as root"; exit 1; }
[[ $# -eq 2 ]] || { echo "Usage: $0 <username> <group>"; exit 1; }

USERNAME="$1"
GROUP="$2"

# Create group if not exists
getent group "$GROUP" &>/dev/null || groupadd "$GROUP"

# Create user
useradd -m -s /bin/bash -G "$GROUP" "$USERNAME"

# Set random password
PASS=$(openssl rand -base64 12)
echo "${USERNAME}:${PASS}" | chpasswd
chage -d 0 "$USERNAME"    # Force password change on first login

echo "User '$USERNAME' created"
echo "Temp password: $PASS"
echo "User must change password on first login"
```

---

## Debugging Techniques

```bash
# Trace execution
bash -x script.sh           # Print each command
set -x                      # Enable inside script
set +x                      # Disable trace

# Dry run pattern
DRY_RUN="${DRY_RUN:-false}"
run() {
    if [[ "$DRY_RUN" == "true" ]]; then
        echo "[DRY RUN] $*"
    else
        "$@"
    fi
}
run rm -rf /tmp/old_files/

# Print variable values
declare -p VARNAME          # Show type + value
echo "DEBUG: value=$VAR"

# Check script with shellcheck
shellcheck script.sh        # Install: apt install shellcheck
```

---

## Related Topics

- [Linux CLI ←](03_Linux_CLI.md) — commands used in scripts
- [Services & Processes ←](15_Services_Processes.md) — cron, systemd timers
- [File Management ←](06_File_Management.md) — find, tar, rsync
- [Troubleshooting ←](18_Troubleshooting.md)
- [Backup & Disaster Recovery →](29_Backup_Disaster_Recovery.md)

---

> [Index](00_INDEX.md)
