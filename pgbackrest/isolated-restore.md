# Restore On Isolated machine

Below I’ll give you a practical, repeatable restore-and-verify operation plan (what to run, where, when, and what to check) that:

- uses an isolated restore host so your Patroni cluster is never affected,
- exercises both full backup + WAL replay (pgBackRest PITR) and basic consistency checks, and

## Architecture

- **backup server**: hold current backups.
- **restore-node**: isolated machine running same OS & PG versions, outside patroni cluster to restore data on it

### Step 1: Setup postgres

on _restore-node_ make sure that you use the same cluster db version

```bash
# Step 1: Import PostgreSQL Official GPG Key
sudo apt update
sudo apt install -y curl gnupg lsb-release

curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

# Step 2: Add PostgreSQL 16 APT Repository
 echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
sudo tee /etc/apt/sources.list.d/pgdg.list

# Step 3: Install PostgreSQL 16
sudo apt update
sudo apt install -y postgresql-16

# Step 4: Enable & Start PostgreSQL
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Step 5: Verify Installation
psql --version
sudo -i -u postgres psql

## inside shell run the following
SELECT version();
```

### Step 2: Setup pgbackrest

on _restore-node_ make sure you setup that same version of pgbackrest that been used by _backup-node_

```bash
sudo apt update
sudo apt install -y pgbackrest

## create directories and update permissions
sudo mkdir -p -m 770 /var/log/pgbackrest
sudo chown postgres:postgres /var/log/pgbackrest
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /etc/pgbackrest/conf.d
sudo touch /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
sudo rm -rf /etc/pgbackrest.conf
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest
```

### Step 3: Configure SSH Access

make sure both backup and restore node can connect each other through ssh without password

On the Primary Server (as postgres user):

```bash
sudo -u postgres mkdir -m 750 -p /var/lib/postgresql/.ssh
sudo -u postgres ssh-keygen -f /var/lib/postgresql/.ssh/id_rsa -t rsa -b 4096 -N ""

# if ssh-keygen not found install the following
sudo apt install -y openssh-server
sudo service ssh start


chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

copy **id_rsa.pub** of _restore_node_ into **authorized_keys** file of _backup_node_ and verse vice.

Ensure SSH works:

```bash
sudo -u pgbackrest ssh postgres@restore_node_ip hostname # run from backup node
sudo -u postgres ssh pgbackrest@backup_node_ip hostname # run from restore node
```

### Step 4: Config pgbackrest

on _restore-node_ create pgbackrest config file and add the following

```bash

sudo nano /etc/pgbackrest/pgbackrest.conf
```

add the following

```conf
[global]
repo1-host=192.168.137.103  # backup server ip
repo1-host-user=pgbackrest
start-fast=y
log-level-console=info
log-level-file=debug

[cn-db] # make sure to use the same stanza that exist backup server
pg1-path=/var/lib/postgresql/data # make sure to use your current data directory
```

check pgbackrest configured correctly

run the following to see current available backups on backup servers.

```bash
sudo -u postgres pgbackrest --stanza=cn-db info
```

### Step 5: Restore data

On **restore_node**

#### stop postgres and allow wal log archiving

first I need to stop postgres

```bash
sudo systemctl stop postgres
```

update postgres configeration to allow wal log archiving

```bash
 sudo nano  /etc/postgresql/16/main/postgresql.conf
```

make sure that you set the following

```conf
wal_level = replica
archive_mode = on
archive_command = 'pgbackrest --stanza=cn-db archive-push %p'
max_wal_senders = 3
wal_keep_size = 256MB
```

delete or mv current data directory

```bash
sudo rm -rf /var/lib/postgres/$current_data_dir
```

#### start pgbackrest restore

get target backup lable from

```bash
sudo -u postgres pgbackrest --stanza=sarwa-db info
```

restore data

```bash
sudo -u postgres pgbackrest --stanza=sarwa-db --delta --set=20250810-103436F --log-level-console=detail restore


sudo -u postgres pgbackrest --stanza=sarwa-db --delta --log-level-console=detail restore ## restore latest
```

start postgres

```bash
sudo systemctl start postgresql
```

you can check logs for any

```bash
sudo cat /var/log/postgresql/postgresql-<version>-main.log
```

---

## Create Restore script

create restore log files location and update permissions.

```bash
sudo mkdir -p /var/log/pgbackrest-tests
sudo chown someuser:root /var/log/pgbackrest-tests
sudo chmod 770 /var/log/pgbackrest-tests
```

create script

```bash
sudo nano ~/.verify_restore.sh
```

and add the following

```bash
#!/bin/bash
set -euo pipefail

STANZA="sarwa-db"
DATA_PATH="/var/lib/postgresql/16/main"
LOG_DIR="/var/log/pgbackrest-tests"
LOG_FILE="$LOG_DIR/pgbackrest_restore_$(date +%F_%H-%M-%S).log"
DB_NAME="sarwa"

exec > >(tee -a "$LOG_FILE") 2>&1

echo "=== Starting pgBackRest restore test: $(date) ==="

# Stop PostgreSQL
echo "Stopping PostgreSQL..."
sudo systemctl stop postgresql
sleep 5

# Safety checks
if systemctl is-active --quiet postgresql; then
    echo "ERROR: PostgreSQL still running. Aborting."
    exit 1
fi

if [ ! -d "$DATA_PATH" ]; then
    echo "ERROR: Data directory $DATA_PATH does not exist."
    exit 1
fi

echo "Deleting old data directory..."
sudo -u postgres rm -rf "$DATA_PATH"/*
echo "Data directory cleared."

# Ensure backups exist
echo "Checking backups..."
sudo -u postgres pgbackrest --stanza=$STANZA info

# Restore
echo "Restoring latest backup..."
sudo -u postgres pgbackrest --stanza=$STANZA --delta restore

# Start PostgreSQL
echo "Starting PostgreSQL..."
sudo systemctl start postgresql
sleep 10

if ! pg_isready -q; then
    echo "ERROR: PostgreSQL failed to start."
    sudo -u postgres cat /var/log/postgresql/postgresql-16-main.log
    exit 1
fi

echo "PostgreSQL started successfully."

# Verify database exists
echo "Verifying sarwa database..."
if sudo -u postgres psql -lqt | cut -d \| -f 1 | grep -qw sarwa; then
    echo "✅ Success: $DB_NAME database found."
    sudo -u postgres psql -d "$DB_NAME" -c "\dt"
else
    echo "❌ ERROR: $DB_NAME database missing!"
    exit 1
fi

echo "=== Restore test completed successfully: $(date) ==="
```

Schedule via cron: **_0 2 _ _ 0 .verify_restore.sh_**

```bash
sudo crontab -u someuser -e

### add the following
0 2 * * 0 /path/to/restore_test.sh
```

to manually start postgres

```bash
sudo -u postgres /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/main
```

make sure restore server not running old instance of postgres

```bash
ps -ef | grep postgres
```
