# 06 — File Management Concepts

> **[← Permissions](05_Permissions.md)** | **[Index](00_INDEX.md)** | **[Networking →](07_Networking_Fundamentals.md)**

---

## File Types and Extensions

### Linux File Types

> See also: [File System Structure](02_File_System.md#file-types-in-linux)

Linux identifies file type by content (magic bytes), not extension. The `file` command reveals actual type:

```bash
file document.pdf       # PDF document, version 1.6
file image.jpg          # JPEG image data
file script.sh          # Bourne-Again shell script
file binary             # ELF 64-bit LSB executable
file archive.tar.gz     # gzip compressed data
```

### Common File Categories

| Category | Linux Extensions | Windows Extensions |
|----------|-----------------|-------------------|
| Text | `.txt`, `.md`, `.csv`, `.log` | `.txt`, `.log`, `.csv` |
| Script | `.sh`, `.bash`, `.py`, `.pl` | `.bat`, `.cmd`, `.ps1` |
| Config | `.conf`, `.yaml`, `.toml`, `.ini` | `.ini`, `.cfg`, `.xml`, `.reg` |
| Executable | (no ext required) | `.exe`, `.msi`, `.dll` |
| Archive | `.tar`, `.gz`, `.zip`, `.bz2`, `.xz` | `.zip`, `.7z`, `.rar` |
| Image | `.jpg`, `.png`, `.gif`, `.svg` | same |
| Document | `.pdf`, `.odt`, `.docx` | `.docx`, `.xlsx`, `.pdf` |

---

## Compression and Archiving

### Why Compress?
- Reduce disk space usage
- Faster file transfers
- Bundle many files into one (archiving)

### `tar` — Tape Archive (Linux)

`tar` is primarily for **archiving** (bundling); compression is added via flags.

```bash
# Create archive
tar -cvf archive.tar files/         # Create, verbose, file
tar -czvf archive.tar.gz files/     # + gzip compression
tar -cjvf archive.tar.bz2 files/    # + bzip2 compression
tar -cJvf archive.tar.xz files/     # + xz compression

# Extract archive
tar -xvf archive.tar                # Extract, verbose
tar -xzvf archive.tar.gz            # Extract gzip
tar -xjvf archive.tar.bz2           # Extract bzip2
tar -xJvf archive.tar.xz            # Extract xz
tar -xvf archive.tar -C /dest/      # Extract to specific directory

# List contents (don't extract)
tar -tvf archive.tar
tar -tzvf archive.tar.gz

# Extract specific file
tar -xvf archive.tar file.txt

# Flags explained:
# c = create  x = extract  t = list
# v = verbose  f = file (must specify filename)
# z = gzip  j = bzip2  J = xz
```

### `gzip` / `gunzip` — Compress Single Files

```bash
gzip file.txt               # Compress → file.txt.gz (original deleted)
gzip -k file.txt            # Keep original
gzip -d file.txt.gz         # Decompress (same as gunzip)
gunzip file.txt.gz          # Decompress
gzip -l file.txt.gz         # List compression ratio
gzip -9 file.txt            # Maximum compression
gzip -1 file.txt            # Fastest (least compression)
```

### `zip` / `unzip` — Cross-Platform Archives

```bash
zip archive.zip file1 file2     # Create zip
zip -r archive.zip folder/      # Recursive
zip -e archive.zip file.txt     # Encrypt with password
unzip archive.zip               # Extract
unzip archive.zip -d /dest/     # Extract to directory
unzip -l archive.zip            # List contents
unzip -p archive.zip file.txt   # Print file to stdout
```

### Compression Algorithm Comparison

| Algorithm | Extension | Speed | Ratio | Use Case |
|-----------|-----------|-------|-------|---------|
| **gzip** | `.gz` | Fast | Medium | General purpose |
| **bzip2** | `.bz2` | Medium | Better | Source tarballs |
| **xz** | `.xz` | Slow | Best | Package distributions |
| **lz4** | `.lz4` | Very fast | Low | Real-time compression |
| **zstd** | `.zst` | Fast | Good | Modern alternative |
| **zip** | `.zip` | Fast | Medium | Windows compat |

---

## File Search

### `find` — Search by Attributes

```bash
# By name
find /home -name "*.txt"
find / -iname "readme*"          # Case-insensitive

# By type
find . -type f                   # Files only
find . -type d                   # Directories only
find . -type l                   # Symbolic links

# By size
find / -size +500M               # Larger than 500MB
find / -size -1k                 # Smaller than 1KB
find / -size 1024c               # Exactly 1024 bytes

# By time (days)
find . -mtime -7                 # Modified within 7 days
find . -mtime +30                # Modified more than 30 days ago
find . -atime -1                 # Accessed today
find . -newer /etc/hosts         # Newer than /etc/hosts

# By permission
find . -perm 644
find . -perm /111                # Any execute bit set

# By owner
find . -user alice
find . -group devs

# Combining conditions
find /var -name "*.log" -mtime +7 -size +100M

# Execute on results
find . -name "*.tmp" -delete                    # Delete matches
find . -name "*.txt" -exec ls -l {} \;          # ls each file
find . -name "*.log" -exec gzip {} \;           # Compress each
```

### `grep` — Search by Content

> See [Linux CLI](03_Linux_CLI.md#grep--search-text) for full reference

```bash
grep "error" /var/log/syslog
grep -r "TODO" /home/alice/projects/
grep -rl "password" /etc/          # Filenames only
grep -n "function" *.py            # Show line numbers
```

### `locate` — Fast Filename Search

```bash
locate filename                # Uses pre-built database
locate -i filename             # Case-insensitive
sudo updatedb                  # Update database
locate "*.conf" | grep nginx   # Combine with grep
```

---

## Directory Structure Management

### Organizing Projects

```
project/
├── README.md
├── .gitignore
├── src/                    # Source code
│   ├── main.py
│   └── utils/
├── tests/                  # Test files
├── docs/                   # Documentation
├── scripts/                # Helper scripts
├── config/                 # Configuration files
├── logs/                   # Log output (gitignored)
└── data/                   # Data files (often gitignored)
```

### Useful Directory Commands

```bash
# Show directory tree
tree /path                   # If tree is installed
find . -type d               # All directories
ls -R | grep :/ | sed 's/:/\n/;s/[^/]*/  /g'  # DIY tree

# Disk usage
du -sh directory/            # Size of directory
du -sh *                     # Size of each item in current dir
du -sh * | sort -h           # Sort by size
df -h                        # Disk usage per filesystem

# Count files
ls | wc -l                   # Count files in current dir
find . -type f | wc -l       # Count all files recursively
```

---

## File Metadata

```bash
stat file.txt               # Full metadata
ls -l file.txt              # Basic metadata
file file.txt               # File type detection

# Timestamps
touch file.txt               # Update all timestamps to now
touch -a file.txt            # Update only atime
touch -m file.txt            # Update only mtime
touch -t 202401011200 f      # Set specific timestamp

# Attributes (Linux extended attributes)
lsattr file.txt              # List attributes
chattr +i file.txt           # Make immutable (cannot be deleted/modified, even by root)
chattr -i file.txt           # Remove immutable
chattr +a file.txt           # Append-only
```

---

## Temporary Files and Cleanup

```bash
# Linux temp directories
/tmp                    # Cleared on reboot
/var/tmp                # Persists across reboots

# Find and clean old temp files
find /tmp -mtime +7 -delete             # Delete files older than 7 days
find /var/log -name "*.gz" -mtime +30 -delete  # Old compressed logs

# Clean package cache
sudo apt clean                          # Debian/Ubuntu
sudo pacman -Sc                         # Arch Linux

# Clear thumbnail cache
rm -rf ~/.cache/thumbnails/

# Windows temp cleanup
del /Q %TEMP%\*                         # Clear user temp
cleanmgr                                # Disk cleanup GUI
```

---

## File Integrity and Checksums

```bash
# Generate checksums
md5sum file.txt                 # MD5 (fast, not secure)
sha1sum file.txt                # SHA-1
sha256sum file.txt              # SHA-256 (recommended)
sha512sum file.txt              # SHA-512 (most secure)

# Verify checksum
sha256sum -c checksums.txt      # Check against file
echo "HASH  file.txt" | sha256sum -c -

# Generate checksum file for multiple files
sha256sum *.iso > SHA256SUMS
sha256sum -c SHA256SUMS         # Verify all
```

**Windows (PowerShell):**
```powershell
Get-FileHash file.txt -Algorithm SHA256
Get-FileHash file.txt -Algorithm MD5
```

---

## File Transfer

### `cp` and `mv` (local)
> See [Linux CLI →](03_Linux_CLI.md)

### `rsync` — Efficient Remote/Local Sync

```bash
# Local sync
rsync -av source/ dest/             # Archive + verbose
rsync -av --delete source/ dest/    # Delete extra files in dest

# Remote sync over SSH
rsync -avz source/ user@host:/dest/      # Compress during transfer
rsync -avz user@host:/remote/ local/     # Pull from remote

# Common flags
# -a = archive (recursive, preserve permissions/timestamps)
# -v = verbose
# -z = compress during transfer
# -n = dry run (test without changes)
# -P = show progress + resume partial
# --exclude='*.log'
# --delete = remove files in dest not in source
```

### `scp` — Secure Copy

```bash
scp file.txt user@host:/path/       # Copy to remote
scp user@host:/path/file.txt ./     # Copy from remote
scp -r dir/ user@host:/path/        # Recursive
scp -P 2222 file user@host:/path/   # Custom port
```

> See [Cloud & Remote Access →](17_Cloud_Remote_Access.md) for SSH details

---

## Windows File Management

```powershell
# Search
Get-ChildItem -Recurse -Filter "*.log"
Get-ChildItem -Path C:\ -Include "*.txt" -Recurse

# Compress (built-in zip)
Compress-Archive -Path folder\ -DestinationPath archive.zip
Expand-Archive -Path archive.zip -DestinationPath dest\

# File info
Get-Item file.txt | Select-Object *
(Get-Item file.txt).LastWriteTime
(Get-Item file.txt).Length

# Checksums
Get-FileHash file.txt -Algorithm SHA256
```

---

## Related Topics

- [File System Structure ←](02_File_System.md)
- [Linux CLI ←](03_Linux_CLI.md)
- [Windows CLI ←](04_Windows_CLI.md)
- [User Permissions ←](05_Permissions.md)
- [Cloud & Remote Access →](17_Cloud_Remote_Access.md)
- [Git Fundamentals →](19_Git_Fundamentals.md) — version-controlled file management

---

> [← Permissions](05_Permissions.md) | [Index](00_INDEX.md) | [Networking Fundamentals →](07_Networking_Fundamentals.md)
