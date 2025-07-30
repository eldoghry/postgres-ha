## PgBackRest Setup

### 🛠️ Step 1: Install pgBackRest on All Nodes

On all three servers (Primary, Replica, and Backup Server):

```bash
sudo apt update
sudo apt install pgbackrest -y
```

make sure all nodes share the same pgbackrest versio

```bash
pgbackrest version # pgBackRest 2.56.0

```

optional if any node can't get latest stable version add the following
Add the PostgreSQL APT Repository which provide latest stable version on pgbackrest

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo apt update

# remove old version
sudo apt remove pgbackrest

# reinstall it again
sudo apt install pgbackrest -y
```

or you can build it from the source please check pgbackrest documentation [pgbackrest build section](https://pgbackrest.org/user-guide.html#build)

### 🧑‍🔧 Step 2: Create backup user on Backup server

Create Custom user for pgbackrest operation

```bash
sudo adduser --disabled-password --gecos "" pgbackrest
```

create logs and configuration files and directory

```bash
sudo mkdir -p -m 770 /var/log/pgbackrest
sudo chown pgbackrest:pgbackrest /var/log/pgbackrest
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /etc/pgbackrest/conf.d
sudo touch /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
sudo chown pgbackrest:pgbackrest /etc/pgbackrest/pgbackrest.conf
sudo rm -rf /etc/pgbackrest.conf
```

Create backup location on Backup server

```bash
sudo mkdir -p /backup
sudo chown pgbackrest:pgbackrest /backup
sudo chmod 750 /backup
```

**On Database Servers**

create logs and configuration files and directory and give permission for **_postgres_** user.

```bash
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

### 🔑 Step 3: SSH Key Setup (Backup Server → DBs)

On backup server (192.168.137.103):

```bash
sudo -u pgbackrest ssh-keygen -t rsa -b 4096 -f /home/pgbackrest/.ssh/id_rsa -N ""
```

Copy the key to Primary and Replica:

```bash
ssh-copy-id -i /home/pgbackrest/.ssh/id_rsa.pub user@192.168.137.101
ssh-copy-id -i /home/pgbackrest/.ssh/id_rsa.pub user@192.168.137.102
```

If ssh-copy-id command not work, transfer keys manually,
from backup server copy content of public ssh key

```bash
sudo -u pgbackrest cat /home/pgbackrest/.ssh/id_rsa.pub
```

then on database servers

```bash
# create new file and paste public key content here
sudo -u postgres nano /var/lib/postgresql/.ssh/authorized_keys
```

and do the veris vice, you should generate ssh keys on databases servers and copy public keys on backup server keys under /home/pgbackrest/.ssh/authorized_keys

so backup server can reach dbs servers, and dbs servers can reach backup servers

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
# sudo patronictl reload <cluster-name>
sudo patronictl -c /etc/patroni/config.yml reload cn-postgresql-cluster

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
