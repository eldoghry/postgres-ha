## PgBackRest Setup

### 🛠️ Step 1: Install pgBackRest on All Nodes

On all three servers (Primary, Replica, and Backup Server):

```bash
sudo apt update
sudo apt install pgbackrest -y
```

### 🧑‍🔧 Step 2: Create backup OS User on All Nodes

```bash
sudo useradd --system --home /var/lib/pgbackrest --shell /bin/bash backup
sudo mkdir -p /var/lib/pgbackrest/.ssh
sudo chown backup:backup /var/lib/pgbackrest
```

### 🔑 Step 3: SSH Key Setup (Backup Server → DBs)

On backup server (192.168.137.103):

```bash
sudo -u backup ssh-keygen -t rsa -b 4096 -f /var/lib/pgbackrest/.ssh/id_rsa -N ""
```

Copy the key to Primary and Replica:

```bash
ssh-copy-id -i /var/lib/pgbackrest/.ssh/id_rsa.pub backup@192.168.137.101
ssh-copy-id -i /var/lib/pgbackrest/.ssh/id_rsa.pub backup@192.168.137.102
```

If ssh-copy-id command not work, transfer keys manually,
from backup server copy content of public ssh key

```bash
sudo -u backup cat /var/lib/pgbackrest/.ssh/id_rsa.pub
```

then on database servers

```bash
# create new file and paste public key content here
sudo -u backup nano /var/lib/pgbackrest/.ssh/authorized_keys
```

Ensure SSH works:

```bash
sudo -u backup ssh backup@192.168.137.101 hostname
sudo -u backup ssh backup@192.168.137.102 hostname
```

### 📁 Step 4: Create Backup Storage Directory

On backup server:

```bash
sudo mkdir -p /pgbackrest_repo
sudo chown backup:backup /pgbackrest_repo
```

### 📝 Step 5: Configure pgbackrest.conf on All Nodes

#### 🧩 On Backup Server (192.168.137.103):

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
repo1-path=/pgbackrest_repo
repo1-retention-full=7
repo1-retention-diff=14
start-fast=y
log-level-console=info
log-level-file=debug
backup-standby=y         # <-- enforce replica-only backups

[global:archive-push]
compress-level=3

[cn-db]
pg1-host=192.168.137.102 # <--- Replica comes first
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
pg1-user=postgres
pg1-host-user=backup

pg2-host=192.168.137.101 # Primary
pg2-path=/var/lib/postgresql/16/main
pg2-port=5432
pg2-user=postgres
pg2-host-user=backup
```

🧠 What happens with backup-standby=y
pgBackRest will connect to the first available host (replica)

It will check that it is in recovery mode (pg_is_in_recovery() = true)

If it's not in recovery (i.e., a primary), the backup will fail

This guarantees no backups run from a promoted or primary node

Internally, pgBackRest does:

```sql
SELECT pg_is_in_recovery();
```

If it gets true, it's a replica

If it gets false, it's a primary

#### 🧩 On Primary and Replica DB nodes:

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
repo1-path=/var/lib/pgbackrest
start-fast=y
log-level-console=info
log-level-file=debug

[cn-db]
pg1-path=/var/lib/postgresql/16/main

```

### 🪪 Step 6: Add pgbackrest Permissions to postgres

On Primary and Replica:

```bash
sudo usermod -aG backup postgres
```

Give proper permissions to pgbackrest directory:

```bash
sudo mkdir -p /var/lib/pgbackrest
sudo chown postgres:backup /var/lib/pgbackrest
chmod 750 /var/lib/pgbackrest

```

### 🔄 Step 7: Enable Archiving in Patroni

Update patroni.yml on Primary and Replica:

```yaml
postgresql:
  parameters:
    wal_level: replica
    archive_mode: "on"
    archive_command: "pgbackrest --stanza=cn-db archive-push %p"
    max_wal_senders: 10
    wal_keep_size: 256MB
    archive_timeout: 60
```

Then reload Patroni:

```bash
sudo patronictl reload <cluster-name>
```

### 📦 Step 8: Create the Stanza

From Backup Server:

```bash
sudo -u backup pgbackrest --stanza=cn-db stanza-create
```

Check stanza info:

```bash
sudo -u backup pgbackrest --stanza=cn-db info

```

### 🧪 Step 9: Run a Test Backup (from Replica)

```bash
sudo -u backup pgbackrest --stanza=cn-db backup
```

Make sure the backup is stored under /pgbackrest_repo.

### 📅 Step 10: Schedule Cron Job on Backup Server

Edit the crontab:

```bash
sudo crontab -u backup -e
```

add

```bash
# Full backup on Sunday at 1 AM
0 1 * * 0 pgbackrest --stanza=cn-db --type=full backup

# Differential backup daily at 2 AM
0 2 * * 1-6 pgbackrest --stanza=cn-db --type=diff backup

```
