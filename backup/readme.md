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

and do the vers vice, you should generate ssh keys on databases servers and copy public keys on backup server keys under /home/pgbackrest/.ssh/authorized_keys

so backup server can reach dbs servers, and dbs servers can reach backup servers

Ensure SSH works:

```bash
sudo -u backup ssh backup@192.168.137.101 hostname
sudo -u backup ssh backup@192.168.137.102 hostname
```

**Important Notes:**
Make sure the remote ~/.ssh directory and authorized_keys file have proper permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Don't share your private key.

### 📁 Step 4: Create Backup Storage Directory

On backup server:

```bash
sudo mkdir -p /backup
sudo chown pgbackrest:pgbackrest /backup
sudo chmod 750 /backup
```

### 📝 Step 5: Configure pgbackrest.conf on All Nodes

#### 🧩 On Backup Server (192.168.137.103):

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

Copy the following, please make sure to remove comments first to avoid errors later

```ini
[global]
repo1-path=/backup #backup location on backup server
repo1-type=posix
repo1-retention-full=7
repo1-retention-diff=14
start-fast=y
log-level-console=info
log-level-file=debug

[global:archive-push]
compress-type=zst
compress-level=3
delta=y  # Optional: for smaller incremental backups

[cn-db] # stanza name you can change it, should be same on all servers
pg1-host=192.168.137.102 # <--- Replica comes first
pg1-path=/var/lib/postgresql/data # database data location
pg1-port=5432
pg1-user=postgres
pg1-host-user=postgres

pg2-host=192.168.137.101 # Primary
pg2-path=/var/lib/postgresql/data
pg2-port=5432
pg2-user=postgres
pg2-host-user=postgres
backup-standby=y         # <-- enforce replica-only backups
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
repo1-host=192.168.137.103
repo1-host-user=pgbackrest
start-fast=y
log-level-console=info
log-level-file=debug

[cn-db]
pg1-path=/var/lib/postgresql/data

```

### 🔄 Step 6: Enable Archiving in Patroni

Update patroni.yml on Primary and Replica:

```bash
sudo nano /etc/patroni/config.yml
```

and under postgresql section add the following

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

### 📦 Step 7: Create the Stanza

From Backup Server:

```bash
sudo -u pgbackrest pgbackrest --stanza=cn-db stanza-create
```

you should see response like the following

```bash
2025-07-30 08:59:46.489 P00   INFO: stanza-create command begin 2.56.0: --exec-id=4413-91bd72c4 --log-level-console=info --log-level-file=debug --pg1-host=192.168.137.102 --pg2-host=192.168.137.101 --pg1-host-user=postgres --pg2-host-user=postgres --pg1-path=/var/lib/postgresql/data --pg2-path=/var/lib/postgresql/data --pg1-port=5432 --pg2-port=5432 --pg1-user=postgres --pg2-user=postgres --repo1-path=/backup --stanza=cn-db
2025-07-30 08:59:47.481 P00   INFO: stanza-create for stanza 'cn-db' on repo1
2025-07-30 08:59:47.481 P00   INFO: stanza 'cn-db' already exists on repo1 and is valid
2025-07-30 08:59:47.682 P00   INFO: stanza-create command end: completed successfully (1195ms)
```

Check stanza info:

```bash
sudo -u pgbackrest pgbackrest --stanza=cn-db info

```

run a Test Backup (pull data from replica based on configuration pg1-host)

```bash
sudo -u pgbackrest pgbackrest --stanza=cn-db --type=full backup
```

show see output like that

```bash
sudo -u pgbackrest pgbackrest --stanza=cn-db --type=full backup
2025-07-30 12:44:35.199 P00   INFO: backup command begin 2.56.0: --backup-standby=y --exec-id=8071-93300c4c --log-level-console=info --log-level-file=debug --pg1-host=192.168.137.102 --pg2-host=192.168.137.101 --pg1-host-user=postgres --pg2-host-user=postgres --pg1-path=/var/lib/postgresql/data --pg2-path=/var/lib/postgresql/data --pg1-port=5432 --pg2-port=5432 --pg1-user=postgres --pg2-user=postgres --repo1-path=/backup --repo1-retention-diff=14 --repo1-retention-full=7 --repo1-type=posix --stanza=cn-db --start-fast --type=full
2025-07-30 12:44:36.460 P00   INFO: execute non-exclusive backup start: backup begins after the requested immediate checkpoint completes
2025-07-30 12:44:36.606 P00   INFO: backup start archive = 000000230000000100000019, lsn = 1/19000028
2025-07-30 12:44:36.606 P00   INFO: wait for replay on the standby to reach 1/19000028
2025-07-30 12:44:36.696 P00   INFO: replay on the standby reached 1/19000028
2025-07-30 12:44:36.696 P00   INFO: check archive for segment 000000230000000100000019
2025-07-30 12:45:17.839 P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive2025-07-30 12:45:17.859 P00   INFO: backup stop archive = 00000023000000010000001A, lsn = 1/1A000088
2025-07-30 12:45:17.870 P00   INFO: check archive for segment(s) 000000230000000100000019:00000023000000010000001A
2025-07-30 12:45:18.594 P00   INFO: new backup label = 20250730-124436F
2025-07-30 12:45:18.709 P00   INFO: full backup size = 1.6GB, file total = 2297
2025-07-30 12:45:18.709 P00   INFO: backup command end: completed successfully (43511ms)
2025-07-30 12:45:18.709 P00   INFO: expire command begin 2.56.0: --exec-id=8071-93300c4c --log-level-console=info --log-level-file=debug --repo1-path=/backup --repo1-retention-diff=14 --repo1-retention-full=7 --repo1-type=posix --stanza=cn-db
2025-07-30 12:45:18.710 P00   INFO: expire command end: completed successfully (1ms)
```

```bash
sudo -u pgbackrest pgbackrest --stanza=cn-db info
################################################
stanza: cn-db
    status: ok
    cipher: none

    db (current)
        wal archive min/max (16): 0000000100000000000000FB/00000023000000010000001A

        full backup: 20250730-124436F
            timestamp start/stop: 2025-07-30 12:44:36+00 / 2025-07-30 12:45:17+00
            wal start/stop: 000000230000000100000019 / 00000023000000010000001A
            database size: 1.6GB, database backup size: 1.6GB
            repo1: backup set size: 228MB, backup size: 228MB
```

Make sure the backup is stored under /backup.

### 📅 Step 8: Schedule Cron Job on Backup Server

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
