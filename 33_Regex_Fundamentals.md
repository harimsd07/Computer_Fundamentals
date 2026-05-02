# 33 — Regular Expressions (Regex)

> **[← Index](00_INDEX.md)** | **Related: [Linux CLI](03_Linux_CLI.md) · [Bash Scripting](23_Bash_Scripting.md) · [Nginx & Apache](25_Nginx_Apache.md) · [Networking Tools](08_Networking_Tools.md)**

---

## What is Regex?

A **Regular Expression** is a pattern used to match, search, and manipulate text. Regex is used everywhere:

- `grep`, `sed`, `awk` — Linux text processing
- `nginx` config — URL routing rules
- Programming languages — Python, JavaScript, PHP, Go
- Log analysis — extracting fields from logs
- Validation — email, IP, phone number patterns
- CI/CD — matching branch names, file paths

---

## Regex Syntax — Core Building Blocks

### Literal Characters

```
cat     → matches exactly "cat"
Cat     → matches "Cat" (case-sensitive by default)
123     → matches "123"
```

### Metacharacters (Special Characters)

```
.       → Any single character (except newline)
^       → Start of line/string
$       → End of line/string
*       → Zero or more of previous
+       → One or more of previous
?       → Zero or one of previous (optional)
{n}     → Exactly n times
{n,}    → At least n times
{n,m}   → Between n and m times
[]      → Character class (any one inside)
[^]     → Negated class (any one NOT inside)
|       → Alternation (OR)
()      → Grouping / capturing
\       → Escape metacharacter
```

---

## Character Classes

```
[abc]       → a, b, or c
[^abc]      → anything except a, b, c
[a-z]       → any lowercase letter
[A-Z]       → any uppercase letter
[0-9]       → any digit
[a-zA-Z]    → any letter
[a-zA-Z0-9] → alphanumeric
[a-z0-9_-]  → slug characters

# Shorthand classes (POSIX / Perl)
\d          → digit [0-9]
\D          → non-digit [^0-9]
\w          → word char [a-zA-Z0-9_]
\W          → non-word char
\s          → whitespace [ \t\r\n\f]
\S          → non-whitespace
\b          → word boundary
\B          → non-word boundary
```

---

## Quantifiers

```
Pattern     Matches
────────────────────────────────────────────
a*          → "", "a", "aa", "aaa", ...
a+          → "a", "aa", "aaa", ... (at least 1)
a?          → "" or "a"
a{3}        → "aaa" exactly
a{2,4}      → "aa", "aaa", or "aaaa"
a{2,}       → "aa" or more

# Greedy vs Lazy
.*          → Greedy: matches as MUCH as possible
.*?         → Lazy:   matches as LITTLE as possible

Example: Input = "<b>bold</b>"
<.*>        → matches "<b>bold</b>"  (greedy — whole thing)
<.*?>       → matches "<b>"          (lazy — shortest match)
```

---

## Anchors and Boundaries

```
^pattern        → Must match at START of line
pattern$        → Must match at END of line
^pattern$       → Must match ENTIRE line
\bword\b        → Whole word "word" (word boundary)
\Bword\B        → "word" NOT at word boundary

Examples:
^Error          → Lines starting with "Error"
\.log$          → Lines ending with ".log"
^\d{4}-\d{2}-\d{2}$   → Entire line must be a date like 2024-04-22
\bcat\b         → "cat" but NOT "concatenate" or "scat"
```

---

## Groups and Capturing

```
(pattern)       → Capturing group (store match)
(?:pattern)     → Non-capturing group (group without storing)
(?P<name>pat)   → Named capturing group (Python/PCRE)
\1, \2          → Backreference to group 1, 2

Examples:
(foo)(bar)      → Captures "foo" in \1, "bar" in \2
(\w+)@(\w+)     → email: \1=user, \2=domain
(?:https?)://   → Match http:// or https://, don't capture

# Alternation with groups
(cat|dog)       → "cat" or "dog"
^(ERROR|WARN)   → Line starts with ERROR or WARN
```

---

## Lookahead and Lookbehind (PCRE)

```
(?=pattern)     → Positive lookahead  (followed by)
(?!pattern)     → Negative lookahead  (NOT followed by)
(?<=pattern)    → Positive lookbehind (preceded by)
(?<!pattern)    → Negative lookbehind (NOT preceded by)

Examples:
\d+(?= dollars)     → digits followed by " dollars"
\d+(?! dollars)     → digits NOT followed by " dollars"
(?<=\$)\d+          → digits preceded by "$"
(?<!\$)\d+          → digits NOT preceded by "$"

# Practical: match password only (not username)
(?<=Password: )\S+  → value after "Password: "
```

---

## Common Regex Patterns

### Email Address
```regex
^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$
```

### IPv4 Address
```regex
^((25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(25[0-5]|2[0-4]\d|[01]?\d\d?)$
```

### IPv6 Address (simplified)
```regex
^([0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$
```

### URL
```regex
^https?://[^\s/$.?#].[^\s]*$
```

### Date (YYYY-MM-DD)
```regex
^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$
```

### Time (HH:MM:SS)
```regex
^([01]\d|2[0-3]):[0-5]\d:[0-5]\d$
```

### Phone Number (Indian)
```regex
^(\+91|0)?[6-9]\d{9}$
```

### MAC Address
```regex
^([0-9A-Fa-f]{2}[:\-]){5}[0-9A-Fa-f]{2}$
```

### Username (alphanumeric + underscore, 3-20 chars)
```regex
^[a-zA-Z0-9_]{3,20}$
```

### Strong Password (min 8, upper+lower+digit+special)
```regex
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$
```

### Linux File Path
```regex
^(/[^/ ]*)+/?$
```

### HTTP Status Code in Log
```regex
" (2\d{2}|3\d{2}|4\d{2}|5\d{2}) \d+
```

### Semantic Version
```regex
^v?(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(-[a-zA-Z0-9.]+)?$
```

---

## Regex in Linux Tools

### `grep`

```bash
# Basic regex (BRE — Basic Regular Expression)
grep "^ERROR" /var/log/app.log          # Lines starting with ERROR
grep "\.log$" filelist.txt              # Lines ending with .log
grep "[0-9]\{4\}" file.txt              # BRE: 4 digits (escape {})

# Extended regex (ERE) — use -E or egrep
grep -E "^(ERROR|WARN)" /var/log/app.log
grep -E "\d{4}-\d{2}-\d{2}" log.txt    # Dates
grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" # IP addresses
grep -E "https?://" urls.txt            # HTTP/HTTPS URLs
grep -E "^\s*$" file.txt               # Empty/blank lines
grep -Ev "^\s*#|^\s*$" config.txt      # Non-comment, non-empty lines

# Perl-compatible regex (PCRE) — use -P
grep -P "(?<=Password: )\S+" config.txt  # Lookahead/behind
grep -P "\bERROR\b" log.txt             # Word boundary

# Useful flags
grep -c "pattern" file         # Count matching lines
grep -n "pattern" file         # Show line numbers
grep -i "pattern" file         # Case-insensitive
grep -v "pattern" file         # Invert — lines NOT matching
grep -o "pattern" file         # Print only matched part
grep -r "pattern" /dir/        # Recursive
grep -l "pattern" *.txt        # Print only filenames
```

### `sed` — Stream Editor

```bash
# Substitute (s/pattern/replacement/flags)
sed 's/foo/bar/'          file    # Replace first match per line
sed 's/foo/bar/g'         file    # Replace ALL matches
sed 's/foo/bar/gi'        file    # Case-insensitive, all
sed 's/foo/bar/2'         file    # Replace 2nd occurrence only
sed -i 's/foo/bar/g'      file    # Edit file in-place
sed -i.bak 's/foo/bar/g'  file    # In-place with backup

# Using groups/backreferences
sed 's/\(first\) \(second\)/\2 \1/'  file  # Swap two words (BRE)
sed -E 's/(first) (second)/\2 \1/'   file  # ERE version

# Delete lines
sed '/pattern/d'          file    # Delete matching lines
sed '/^$/d'               file    # Delete empty lines
sed '/^#/d'               file    # Delete comment lines
sed '5d'                  file    # Delete line 5
sed '5,10d'               file    # Delete lines 5-10

# Print specific lines
sed -n '5p'               file    # Print only line 5
sed -n '5,10p'            file    # Print lines 5-10
sed -n '/pattern/p'       file    # Print matching lines (like grep)

# Insert/append
sed '5a\New line after 5'  file   # Append after line 5
sed '5i\New line before 5' file   # Insert before line 5

# Multiple commands
sed -e 's/foo/bar/g' -e '/empty/d' file

# Real-world examples
# Remove trailing whitespace
sed -i 's/[[:space:]]*$//' file.txt

# Extract value from config
sed -n 's/^PORT=//p' .env        # Extract port number

# Comment out a line containing pattern
sed -i '/^HOSTNAME=/s/^/#/' /etc/hosts

# Uncomment lines
sed -i 's/^#\(HOSTNAME=\)/\1/' /etc/hosts

# Replace entire line matching pattern
sed -i 's/^PORT=.*/PORT=8080/' .env
```

### `awk` — Field Processing

```bash
# Print specific field
awk '{print $1}'           file    # First field
awk '{print $NF}'          file    # Last field
awk '{print $1, $3}'       file    # Fields 1 and 3
awk -F: '{print $1}'       /etc/passwd   # Custom delimiter ":"
awk -F, '{print $2}'       data.csv

# Conditionals
awk '$3 > 100 {print $0}'  file    # Lines where field 3 > 100
awk '/ERROR/ {print $0}'   file    # Lines matching pattern
awk '!/^#/ {print}'        file    # Skip comment lines

# Built-in variables
# NR = current line number
# NF = number of fields in current line
# FS = field separator (default: whitespace)
# OFS = output field separator
# RS = record separator (default: newline)

# Print line numbers
awk '{print NR": "$0}' file

# Filter by line number
awk 'NR==5'               file     # Print line 5
awk 'NR>=5 && NR<=10'     file     # Lines 5-10

# Arithmetic
awk '{sum += $1} END {print "Total:", sum}' numbers.txt
awk 'END {print NR, "lines"}' file  # Count lines

# Format output
awk -F: '{printf "%-20s %s\n", $1, $7}' /etc/passwd

# Process nginx access log
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
# Prints: count of each HTTP status code

# Sum bytes transferred
awk '{sum += $10} END {printf "Total: %.2f MB\n", sum/1024/1024}' /var/log/nginx/access.log

# Extract IPs with 5xx errors
awk '$9 ~ /^5/ {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

---

## Regex in Python

```python
import re

# Basic matching
pattern = r'\d{4}-\d{2}-\d{2}'   # Raw string (r"") avoids double-escaping
text = "Date: 2024-04-22, Next: 2024-05-01"

# Search (find first match anywhere)
match = re.search(pattern, text)
if match:
    print(match.group())    # "2024-04-22"
    print(match.start())    # Start index
    print(match.end())      # End index

# Match (only at beginning of string)
re.match(r'^\d+', '123abc')  # Matches
re.match(r'^\d+', 'abc123')  # None

# Find all matches
dates = re.findall(pattern, text)   # ["2024-04-22", "2024-05-01"]

# Find all with groups
emails = re.findall(r'(\w+)@(\w+\.\w+)', "alice@ex.com, bob@test.org")
# [('alice', 'ex.com'), ('bob', 'test.org')]

# Substitute
result = re.sub(r'\bfoo\b', 'bar', 'foo foobar foo')  # "bar foobar bar"
result = re.sub(r'(\d{4})-(\d{2})-(\d{2})', r'\3/\2/\1', '2024-04-22')  # "22/04/2024"

# Split
parts = re.split(r'[,\s]+', 'a, b,c  d')  # ['a', 'b', 'c', 'd']

# Compile for reuse (performance)
ip_pattern = re.compile(r'^(\d{1,3}\.){3}\d{1,3}$')
if ip_pattern.match(user_input):
    print("Valid IP")

# Flags
re.IGNORECASE  # or re.I  — case-insensitive
re.MULTILINE   # or re.M  — ^ and $ match line boundaries
re.DOTALL      # or re.S  — . matches newline too
re.VERBOSE     # or re.X  — allow whitespace/comments in pattern

pattern = re.compile(r'''
    ^               # Start of string
    (\d{1,3}\.){3}  # Three groups of 1-3 digits followed by dot
    \d{1,3}         # Final octet
    $               # End of string
''', re.VERBOSE)
```

---

## Regex in JavaScript

```javascript
// Literal syntax
const pattern = /\d{4}-\d{2}-\d{2}/;
const patternG = /\d{4}-\d{2}-\d{2}/g;    // Global flag
const patternI = /hello/i;                   // Case-insensitive
const patternGI = /hello/gi;                 // Both

// Test (returns boolean)
/^\d+$/.test('12345');          // true
/^\d+$/.test('123a5');          // false

// Match
'2024-04-22'.match(/(\d{4})-(\d{2})-(\d{2})/);
// ["2024-04-22", "2024", "04", "22", ...]
// result[0] = full match, result[1],[2],[3] = groups

'find all dates: 2024-04-22 and 2024-05-01'.match(/\d{4}-\d{2}-\d{2}/g);
// ["2024-04-22", "2024-05-01"]  (g flag = array of all)

// Replace
'hello world'.replace(/world/, 'there');          // "hello there"
'aabbcc'.replace(/(.)\1/g, '$1');                  // "abc" (remove doubles)
'2024-04-22'.replace(/(\d{4})-(\d{2})-(\d{2})/, '$3/$2/$1');  // "22/04/2024"

// Replace with function
'hello world'.replace(/\b\w/g, c => c.toUpperCase()); // "Hello World"

// Split
'a, b , c,d'.split(/\s*,\s*/);    // ['a', 'b', 'c', 'd']

// Named groups (ES2018+)
const m = '2024-04-22'.match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/);
m.groups.year;   // "2024"
m.groups.month;  // "04"
```

---

## Regex in Nginx Config

```nginx
# Location matching — regex with ~  (case-sensitive) or ~* (case-insensitive)
location ~ \.php$ {
    fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
}

location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
    expires 30d;
    add_header Cache-Control "public";
}

# Capture groups for rewrites
location ~ ^/api/v(\d+)/(.*)$ {
    proxy_pass http://backend-v$1/$2;
}

# Block sensitive files
location ~ /\.(env|git|svn|htaccess) {
    deny all;
    return 404;
}

# Rewrite rules
rewrite ^/old/(.*)$ /new/$1 permanent;           # Redirect
rewrite ^/product/(\d+)$ /item?id=$1 last;       # Internal rewrite
```

---

## Regex Flags Reference

| Flag | Character | Meaning |
|------|-----------|---------|
| Global | `g` | Find all matches (not just first) |
| Case-insensitive | `i` | Ignore case |
| Multiline | `m` | `^`/`$` match line boundaries |
| Dotall | `s` | `.` matches newline too |
| Unicode | `u` | Unicode support |
| Verbose | `x` | Allow whitespace + comments in pattern |

---

## Debugging Regex

```bash
# Test regex interactively
echo "test string" | grep -P "your_pattern"
echo "192.168.1.100" | grep -P "^(\d{1,3}\.){3}\d{1,3}$"

# Online tools:
# regex101.com   — explain + debug + test with PCRE/Python/JS
# regexr.com     — visual regex builder
# debuggex.com   — visual railroad diagram

# In Python — see what matched
import re
m = re.search(r'(\d+)\.(\d+)', 'version 3.14')
print(m.group(0))   # "3.14" (full match)
print(m.group(1))   # "3"    (group 1)
print(m.group(2))   # "14"   (group 2)
print(m.span())     # (8, 12) start/end indices
```

---

## Practical Examples — Log Analysis

```bash
# Extract all unique IP addresses from nginx access log
grep -oP '^\d+\.\d+\.\d+\.\d+' /var/log/nginx/access.log | sort -u

# Count 404 errors per IP
awk '$9 == 404 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Extract slow requests (>1000ms) from nginx log
awk '{if ($NF > 1) print $0}' /var/log/nginx/access.log

# Find failed SSH logins
grep -oP '(?<=from )\d+\.\d+\.\d+\.\d+' /var/log/auth.log | sort | uniq -c | sort -rn

# Extract email addresses from a file
grep -oP '[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}' contacts.txt

# Extract all URLs from HTML file
grep -oP 'https?://[^\s"<>]+' page.html

# Find lines with timestamps older than a specific time
grep -P '^\[2024-04-22 0[0-9]:' app.log    # Morning hours only

# Parse Apache combined log format
awk '{
    ip=$1; time=$4; method=$6; url=$7; status=$9; size=$10;
    gsub(/[\[\]"]/, "", time);
    printf "IP: %-15s Status: %s URL: %s\n", ip, status, url
}' /var/log/apache2/access.log
```

---

## Quick Reference Card

```
ANCHORS          QUANTIFIERS       CLASSES
^  start         *  0 or more      .   any char
$  end           +  1 or more      \d  digit
\b word boundary ?  0 or 1         \w  word char
\B non-boundary  {n} exactly n     \s  whitespace
                 {n,} at least n   \D  non-digit
GROUPS           {n,m} n to m      \W  non-word
() capture       *? lazy           \S  non-space
(?:) non-capture +? lazy
\1 backreference SPECIAL
                 [abc] class       LOOKAROUND
FLAGS            [^abc] negated    (?=) ahead
g  global        [a-z] range       (?!) neg ahead
i  case-insens   | alternation     (?<=) behind
m  multiline     \ escape          (?<!) neg behind
s  dot-all
```

---

## Related Topics

- [Linux CLI ←](03_Linux_CLI.md) — grep, sed, awk usage
- [Bash Scripting ←](23_Bash_Scripting.md) — regex in scripts
- [Nginx & Apache ←](25_Nginx_Apache.md) — location matching
- [Monitoring & Logging ←](13_Monitoring_Logging.md) — log parsing
- [Troubleshooting ←](18_Troubleshooting.md) — log analysis

---

> [Index](00_INDEX.md)
