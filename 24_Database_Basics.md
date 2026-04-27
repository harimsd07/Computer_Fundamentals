# 24 — Database Basics (MySQL / PostgreSQL / SQLite)

> **[← Index](00_INDEX.md)** | **Related: [Services & Processes](15_Services_Processes.md) · [Security Concepts](14_Security_Concepts.md) · [Backup & DR](29_Backup_Disaster_Recovery.md)**

---

## Database Types Overview

```mermaid
graph TD
    DB[Databases] --> REL[Relational / SQL]
    DB --> NOSQL[Non-Relational / NoSQL]

    REL --> MYSQL["MySQL / MariaDB\nWeb apps, LAMP stack"]
    REL --> PSQL["PostgreSQL\nComplex queries, JSON, PostGIS"]
    REL --> SQLITE["SQLite\nEmbedded, file-based"]
    REL --> MSSQL["MS SQL Server\nWindows, .NET apps"]

    NOSQL --> MONGO["MongoDB\nDocument store, JSON"]
    NOSQL --> REDIS["Redis\nKey-value, cache, pub/sub"]
    NOSQL --> ELASTIC["Elasticsearch\nFull-text search"]
```

---

## MySQL / MariaDB

### Installation

```bash
# Ubuntu/Debian
sudo apt install mysql-server
sudo mysql_secure_installation       # Initial hardening wizard

# Arch Linux
sudo pacman -S mysql
sudo mysqld --initialize             # Initialize data directory
sudo systemctl start mysqld

# MariaDB (MySQL drop-in replacement)
sudo apt install mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

### Connecting

```bash
mysql -u root -p                     # Connect as root
mysql -u alice -p mydb               # Connect as alice to mydb
mysql -h 192.168.1.50 -u alice -p    # Remote connection
mysql -u root -p -e "SHOW DATABASES;" # Run query from CLI
```

### Database Management

```sql
-- Show databases
SHOW DATABASES;

-- Create database
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Use database
USE myapp;

-- Drop database (⚠️ irreversible)
DROP DATABASE myapp;

-- Show tables
SHOW TABLES;

-- Show table structure
DESCRIBE users;
SHOW CREATE TABLE users;
```

### User Management

```sql
-- Create user
CREATE USER 'alice'@'localhost' IDENTIFIED BY 'StrongPassword123!';
CREATE USER 'appuser'@'%' IDENTIFIED BY 'AppPass!';    -- From any host

-- Grant privileges
GRANT ALL PRIVILEGES ON myapp.* TO 'alice'@'localhost';
GRANT SELECT, INSERT, UPDATE ON myapp.* TO 'appuser'@'%';
GRANT SELECT ON myapp.users TO 'readonly'@'localhost';
FLUSH PRIVILEGES;

-- Show grants
SHOW GRANTS FOR 'alice'@'localhost';

-- Revoke privileges
REVOKE INSERT ON myapp.* FROM 'appuser'@'%';

-- Change password
ALTER USER 'alice'@'localhost' IDENTIFIED BY 'NewPassword!';

-- Drop user
DROP USER 'alice'@'localhost';
```

### Table Operations

```sql
-- Create table
CREATE TABLE users (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT,
    username    VARCHAR(50)  NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    role        ENUM('admin','user','guest') DEFAULT 'user',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active   BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (id),
    INDEX idx_email (email),
    INDEX idx_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Add column
ALTER TABLE users ADD COLUMN last_login TIMESTAMP NULL;

-- Modify column
ALTER TABLE users MODIFY COLUMN role VARCHAR(20) DEFAULT 'user';

-- Add index
ALTER TABLE users ADD INDEX idx_role (role);

-- Drop table
DROP TABLE IF EXISTS users;
```

### CRUD Operations

```sql
-- INSERT
INSERT INTO users (username, email, password, role)
VALUES ('alice', 'alice@example.com', '$2b$10$hash...', 'admin');

-- INSERT multiple rows
INSERT INTO users (username, email, password) VALUES
    ('bob',     'bob@example.com',     '$2b$10$hash1'),
    ('charlie', 'charlie@example.com', '$2b$10$hash2');

-- SELECT
SELECT * FROM users;
SELECT id, username, email FROM users WHERE is_active = 1;
SELECT * FROM users WHERE role = 'admin' ORDER BY created_at DESC LIMIT 10;
SELECT * FROM users WHERE email LIKE '%@example.com';
SELECT COUNT(*) FROM users;
SELECT role, COUNT(*) AS count FROM users GROUP BY role;

-- JOIN
SELECT u.username, p.bio, p.avatar
FROM users u
INNER JOIN profiles p ON u.id = p.user_id
WHERE u.is_active = 1;

-- LEFT JOIN (include users with no profile)
SELECT u.username, p.bio
FROM users u
LEFT JOIN profiles p ON u.id = p.user_id;

-- UPDATE
UPDATE users SET last_login = NOW() WHERE id = 42;
UPDATE users SET is_active = 0 WHERE last_login < DATE_SUB(NOW(), INTERVAL 1 YEAR);

-- DELETE
DELETE FROM users WHERE id = 42;
DELETE FROM users WHERE is_active = 0 AND created_at < '2020-01-01';

-- TRUNCATE (delete all rows, reset AUTO_INCREMENT)
TRUNCATE TABLE sessions;
```

### Transactions

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

-- If both succeed:
COMMIT;

-- If something went wrong:
ROLLBACK;

-- With error handling (stored procedure)
DELIMITER //
CREATE PROCEDURE transfer_funds(IN from_id INT, IN to_id INT, IN amount DECIMAL(10,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    COMMIT;
END //
DELIMITER ;
```

### Backup and Restore

```bash
# Backup single database
mysqldump -u root -p myapp > myapp_backup_$(date +%Y%m%d).sql

# Backup with compression
mysqldump -u root -p myapp | gzip > myapp_$(date +%Y%m%d).sql.gz

# Backup all databases
mysqldump -u root -p --all-databases > all_databases.sql

# Backup specific tables
mysqldump -u root -p myapp users sessions > tables_backup.sql

# Restore
mysql -u root -p myapp < myapp_backup.sql
gunzip < myapp_backup.sql.gz | mysql -u root -p myapp

# Hot backup (without locking)
mysqldump -u root -p --single-transaction myapp > backup.sql
```

### MySQL Performance

```sql
-- Show running queries
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- Kill a query
KILL 42;

-- Explain query plan
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';

-- Show slow queries (if slow_query_log enabled)
-- /etc/mysql/mysql.conf.d/mysqld.cnf:
-- slow_query_log = 1
-- slow_query_log_file = /var/log/mysql/slow.log
-- long_query_time = 2

-- Table statistics
SHOW TABLE STATUS;
SELECT * FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'myapp';

-- Index usage
SHOW INDEX FROM users;
```

---

## PostgreSQL

### Installation

```bash
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Connect as postgres superuser
sudo -u postgres psql

# Create user and database
sudo -u postgres createuser --interactive
sudo -u postgres createdb myapp
```

### Connection

```bash
psql -U alice -d myapp                   # Local connection
psql -h localhost -U alice -d myapp      # TCP connection
psql "postgresql://alice:pass@host:5432/myapp"  # URI format
psql -U alice -d myapp -c "SELECT version();"   # Run command
```

### PostgreSQL-Specific Features

```sql
-- PSQL meta-commands (start with \)
\l          -- List databases
\c myapp    -- Connect to database
\dt         -- List tables
\d users    -- Describe table
\du         -- List users/roles
\timing     -- Toggle query timing
\q          -- Quit

-- Create role
CREATE ROLE alice WITH LOGIN PASSWORD 'secure_password';
CREATE ROLE readonly_role;
GRANT CONNECT ON DATABASE myapp TO alice;
GRANT USAGE ON SCHEMA public TO alice;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
GRANT readonly_role TO alice;

-- PostgreSQL data types (richer than MySQL)
SERIAL / BIGSERIAL      -- Auto-increment
TEXT                    -- Unlimited string
JSONB                   -- Binary JSON (indexable)
UUID                    -- UUID type
ARRAY                   -- Array columns
TIMESTAMP WITH TIME ZONE -- Timezone-aware
INET / CIDR             -- IP address types

-- JSON operations (PostgreSQL strength)
CREATE TABLE events (
    id      SERIAL PRIMARY KEY,
    data    JSONB NOT NULL
);
INSERT INTO events (data) VALUES ('{"user":"alice","action":"login","ip":"1.2.3.4"}');
SELECT data->>'user' FROM events WHERE data->>'action' = 'login';
SELECT * FROM events WHERE data @> '{"user":"alice"}';   -- Contains
CREATE INDEX idx_events_data ON events USING GIN(data);  -- JSON index

-- Common Table Expressions (CTE)
WITH active_users AS (
    SELECT id, username FROM users WHERE is_active = TRUE
),
recent_logins AS (
    SELECT user_id, MAX(created_at) AS last_login
    FROM sessions GROUP BY user_id
)
SELECT u.username, r.last_login
FROM active_users u
LEFT JOIN recent_logins r ON u.id = r.user_id;
```

### PostgreSQL Backup

```bash
# Backup
pg_dump -U alice myapp > myapp_backup.sql
pg_dump -U alice -Fc myapp > myapp_backup.dump   # Custom format (faster restore)
pg_dumpall -U postgres > all_databases.sql        # All databases

# Restore
psql -U alice myapp < myapp_backup.sql
pg_restore -U alice -d myapp myapp_backup.dump
pg_restore -U alice -d myapp -j 4 myapp_backup.dump  # Parallel restore

# pg_basebackup (physical backup)
pg_basebackup -U postgres -D /backup/pg_base -Ft -z -P
```

### PostgreSQL Configuration

```bash
# Key config files
/etc/postgresql/15/main/postgresql.conf    # Main config
/etc/postgresql/15/main/pg_hba.conf        # Client authentication

# pg_hba.conf — authentication rules
# TYPE  DATABASE  USER      ADDRESS       METHOD
local   all       postgres                peer        # Unix socket, OS auth
local   all       all                     peer
host    all       all       127.0.0.1/32  scram-sha-256  # TCP, password
host    myapp     alice     10.0.0.0/8    scram-sha-256

# postgresql.conf — key settings
max_connections = 100
shared_buffers = 256MB            # 25% of RAM recommended
effective_cache_size = 1GB        # 75% of RAM
work_mem = 4MB                    # Per-sort operation
maintenance_work_mem = 64MB
log_slow_queries = 2000           # ms
```

---

## SQLite

Lightweight, serverless, file-based database. Perfect for development, embedded apps, and local tools.

```bash
# Install
sudo apt install sqlite3

# Create/open database
sqlite3 myapp.db

# Run commands
sqlite3 myapp.db "SELECT * FROM users;"
sqlite3 myapp.db < schema.sql
```

```sql
-- SQLite meta-commands
.tables             -- List tables
.schema users       -- Show create statement
.headers on         -- Show column headers
.mode column        -- Column-aligned output
.mode csv           -- CSV output
.output file.csv    -- Output to file
.quit               -- Exit

-- SQLite specifics
-- No strict types (type affinity)
-- No ALTER COLUMN (workaround: recreate table)
-- No DROP COLUMN (pre-3.35)
CREATE TABLE users (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL,
    email   TEXT UNIQUE,
    created DATETIME DEFAULT (datetime('now'))
);

-- SQLite backup
.backup main /path/to/backup.db
-- Or via CLI:
sqlite3 myapp.db ".backup backup.db"
cp myapp.db myapp.db.backup    -- Simple file copy (when no writes)
```

---

## Database Security Best Practices

```sql
-- 1. Never use root for applications
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'StrongPass!';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'appuser'@'localhost';

-- 2. Principle of least privilege
-- Read-only user for reporting
CREATE USER 'reporter'@'localhost' IDENTIFIED BY 'ReportPass!';
GRANT SELECT ON myapp.* TO 'reporter'@'localhost';

-- 3. Never store plaintext passwords
-- Use bcrypt/argon2 hashed passwords in application layer

-- 4. Encrypt sensitive columns
-- Application-level encryption or:
-- MySQL: AES_ENCRYPT() / AES_DECRYPT()
INSERT INTO users (ssn) VALUES (AES_ENCRYPT('123-45-6789', @key));
SELECT AES_DECRYPT(ssn, @key) FROM users;

-- 5. Audit logging
-- MySQL: enable general_log and binary log
-- PostgreSQL: enable pgaudit extension
```

```bash
# MySQL security checklist
mysql_secure_installation      # Run this first
# - Remove anonymous users
# - Disable remote root login
# - Remove test databases
# - Reload privileges

# Bind to localhost only (if no remote connections needed)
# /etc/mysql/mysql.conf.d/mysqld.cnf:
# bind-address = 127.0.0.1
```

---

## Common Database Admin Tasks

```bash
# MySQL: Check database sizes
mysql -u root -p -e "
SELECT table_schema AS 'Database',
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.TABLES
GROUP BY table_schema
ORDER BY SUM(data_length + index_length) DESC;"

# MySQL: Find long-running queries
mysql -u root -p -e "SHOW PROCESSLIST;" | awk '$6 > 30'

# PostgreSQL: Active connections
psql -U postgres -c "SELECT pid, usename, application_name, state, query FROM pg_stat_activity;"

# PostgreSQL: Table sizes
psql -U postgres -d myapp -c "
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;"
```

---

## Related Topics

- [Services & Processes ←](15_Services_Processes.md) — DB as systemd service
- [Security Concepts ←](14_Security_Concepts.md) — DB security
- [Backup & Disaster Recovery →](29_Backup_Disaster_Recovery.md)
- [Nginx & Apache →](25_Nginx_Apache.md) — web + DB stack
- [Docker & Containers →](30_Docker_Containers.md) — DB in containers
- [Monitoring & Logging ←](13_Monitoring_Logging.md) — slow query logs

---

> [Index](00_INDEX.md)
