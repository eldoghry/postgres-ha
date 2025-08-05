## 💾 Restore data

pgBackRest is well-suited for both disaster recovery and accidental data manipulation scenarios.
As we integrate pgbackrest with patroni, Patroni service will be responsible for restoring data and rebuild cluster.

I can restore on of the following

1. specific pgBackRest backup (e.g., a full, differential, or incremental backup identified by its backup label, such as 20250803-140000F)
2. PITR restore based on specific time

Below is the steps to do PITR restore process

#### 1. Stop patroni in all db instance (important)

Stop Patroni services to prevent conflicts during the restore (Run this on all nodes leader and replicas):

```bash
sudo systemctl stop patroni
```

#### 2. Remove patroni Cluster Data from DCS

Run this on all nodes
Clear the cluster’s state in the DCS (etcd) to allow bootstrapping a new cluster:

```bash
sudo patronictl -c /etc/patroni/config.yml remove cluster_name
```

Confirm the cluster name and type Yes I am aware when prompted. This removes the cluster’s metadata from etcd.

#### 3. Wipe PG Data Directories

On all nodes, remove the PostgreSQL data directories to prepare for the restore:

```bash
# Note: some pg version have different data directory
#  you can find it vi running psql -h 192.168.137.101 -U postgres -c "show data_directory;"
sudo rm -rf /var/lib/postgresql/data
```

#### 4. Configure Patroni for PITR Bootstrap

Also on all db nodes

Edit the Patroni configuration file (/etc/patroni/config.yml) on all nodes to include a custom bootstrap method for pgBackRest. Add or modify the bootstrap section:

```bash
sudo nano /etc/patroni/config.yml
```

we should remove or comment the following

```yaml
bootstrap:
  #initdb:
  #   - encoding: UTF8
  #   - data-checksums
```

and add the following

```yaml
bootstrap:
  method: pgbackrest
  pgbackrest:
    command: /var/lib/postgresql/custom_bootstrap.sh # this script will be created later
    keep_existing_recovery_conf: False
    no_params: False
    recovery_conf:
      recovery_target: time
      recovery_target_time: "2025-08-03 14:24:00+03" # replace with you target timestamp you want to restore it
      recovery_target_action: promote
      restore_command: pgbackrest --stanza=cluster_1 archive-get %f "%p" # make sure stanza match one of pgbackrest configuration
```

#### 5. Create the Custom Bootstrap Script

create script file

```bash
sudo touch /var/lib/postgresql/custom_bootstrap.sh
sudo nano /var/lib/postgresql/custom_bootstrap.sh
```

add the following on the script, **_don't forget_** change time and stanza name

```bash
#!/bin/sh
mkdir -p /var/lib/pgsql/13/data
pgbackrest --stanza=cluster_1 --log-level-console=info --delta --type=time "--target=2025-08-03 14:24:00+03" --target-action=promote restore
```

change permissions and make it executable

```bash
sudo chown postgres:postgres /var/lib/postgresql/custom_bootstrap.sh
sudo chmod +x /var/lib/postgresql/custom_bootstrap.sh
```

#### 6. Start Patroni on All Nodes

Start Patroni on each node:

```bash
sudo systemctl start patroni
```

Patroni will execute the custom bootstrap script on the leader node, restoring the database to the specified timestamp using pgBackRest. The leader will then initialize, and replicas will sync via pgBackRest or basebackup (as configured in create_replica_methods).

---

#### 7. Verify the Restore

Check the Patroni cluster status:

```bash
sudo patronictl -c /etc/patroni/config.yml list
```

Ensure all nodes are running (one leader, others as replicas).

Verify the restored data:

```bash
sudo -u postgres psql -c "SELECT * FROM important_table;"
```

Confirm the table or data exists as expected.

Check PostgreSQL logs (e.g., /var/log/postgresql/postgresql-13-main.log) for recovery details, ensuring messages like “recovery stopping before…” and “last completed transaction…” appear.

#### 8. Reconfigure bootstrap on Patroni config

on all nodes

```bash
sudo systemctl stop patroni
sudo patronictl -c /etc/patroni/config.yml remove cluster_name ## change with your patroni cluster name
sudo nano /etc/patroni/config.yaml
```

add or uncomment the following

```yaml
bootstrap:
  initdb:
    - encoding: UTF8
    - data-checksums
```

and remove or comment the following

```yaml
bootstrap:
#   method: pgbackrest
#   pgbackrest:
#     command: /var/lib/postgresql/custom_bootstrap.sh # this script will be created later
#     keep_existing_recovery_conf: False
#     no_params: False
#     recovery_conf:
#       recovery_target: time
#       recovery_target_time: "2025-08-03 14:24:00+03" # replace with you target timestamp you want to restore it
#       recovery_target_action: promote
#       restore_command: pgbackrest --stanza=cluster_1 archive-get %f "%p" # make sure stanza match one of pgbackrest configuration
```

then start patroni one by one

```bash
sudo systemctl start patroni
```

patroni config example at the end.

```yaml
scope: cn-postgresql-cluster
namespace: /service/
name: primary-db # nod

etcd3:
  hosts: 192.168.137.100:2379,192.168.137.101:2379,192.168.137.102:2379 # etcd cluster nodes
  protocol: https
  cacert: /etc/etcd/ssl/ca.crt
  cert: /etc/etcd/ssl/etcd-node2.crt # node1's etcd certificate
  key: /etc/etcd/ssl/etcd-node2.key # node1's etcd key

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.137.101:8008 # IP for node1's REST API
  certfile: /var/lib/postgresql/ssl/primary-db-rest.pem
  cafile: /etc/etcd/ssl/ca.crt

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576 # Failover parameters
    postgresql:
      parameters:
        ssl: "on" # Enable SSL
        ssl_cert_file: /var/lib/postgresql/ssl/db-server.crt # PostgreSQL server certificate
        ssl_key_file: /var/lib/postgresql/ssl/db-server.key # PostgreSQL server key
      pg_hba: # Access rules
        - hostssl replication replicator 127.0.0.1/32 md5
        - hostssl replication replicator 192.168.137.101/32 md5
        - hostssl replication replicator 192.168.137.102/32 md5
        - hostssl all all 127.0.0.1/32 md5
        - hostssl all all 0.0.0.0/0 md5
  #  method: pgbackrest
  #  pgbackrest:
  #    command: /var/lib/postgresql/custom_bootstrap.sh
  #    keep_existing_recovery_conf: False
  #    no_params: False
  #    recovery_conf:
  #      recovery_target: time
  #      recovery_target_time: '2025-08-03 07:15:00+03'
  #      recovery_target_action: promote
  #      restore_command: pgbackrest --stanza=cn-db archive-get %f "%p"
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.137.101:5432 # IP for node1's PostgreSQL
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/16/bin # Binary directory for PostgreSQL 16
  authentication:
    superuser:
      username: postgres
      password: postgres # Superuser password - be sure to change
    replication:
      username: replicator
      password: replicatorpassword # Replication password - be sure to change
  parameters:
    max_connections: 100
    shared_buffers: 256MB
    wal_level: replica
    archive_mode: "on"
    archive_command: "pgbackrest --stanza=cn-db archive-push %p"
    restore_command: "pgbackrest --stanza=cn-db archive-get %f %p"
    max_wal_senders: 10
    wal_keep_size: 256MB
    archive_timeout: 60

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

### Restore PGbackrest (full, differential, or incremental backup)

1. Identify the Backup

Run the following to list available backups and find the desired backup label:

```bash
sudo -u postgres pgbackrest info --stanza=cn-db
```

Note the backup label (e.g., 20250803-140000F for a full backup taken on August 3, 2025, at 14:00).

then repeat the same steps of what we done on PITR

only difference on patroni config and custome restore script

Replace the bootstrap.initdb Section with a Custom pgbackrest Method

- Locate the bootstrap section in /etc/patroni/config.yml

```yaml
bootstrap:
  method: pgbackrest
  pgbackrest:
    command: /var/lib/postgresql/custom_bootstrap.sh
    keep_existing_recovery_conf: False
    no_params: False
    recovery_conf:
      recovery_target_action: promote
      restore_command: pgbackrest --stanza=cn-db archive-get %f "%p"
```

notice No recovery_target or recovery_target_time is needed, as you’re restoring the entire backup, not a specific point in time.we

- Custom Bootstrap Script

make sure to write correct stanza and set backup label from pgbackrest info --stanza=cn-db.

```bash
#!/bin/sh
mkdir -p /var/lib/postgresql/data
pgbackrest --stanza=cn-db --log-level-console=info --delta --set=20250803-140000F --target-action=promote restore
```

---

### ✅ 1. Disaster Recovery (DB instance down / corrupted):

#### Scenario:

The entire PostgreSQL instance or server is down or lost due to disk failure, crash, etc.

#### Solution

### ✅ 2. Accidental Data Manipulation (e.g., table deleted):

sudo -u postgres pgbackrest --stanza=cn-db --type=time --target="2025-08-03 07:15:00" restore

sudo -u postgres pgbackrest --stanza=cn-db --type=time --target="2025-08-03 07:15:00" restore

---

helper commands

```bash
ps aux | grep postgres
sudo -u postgres psql -c "SELECT pg_is_in_recovery();" # should be false
sudo etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ssl/ca.crt --cert=/etc/etcd/ssl/etcd-node3.crt --key=/etc/etcd/ssl/etcd-node3.key del /service/cn-postgresql-cluster --prefix
sudo etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ssl/ca.crt --cert=/etc/etcd/ssl/etcd-node3.crt --key=/etc/etcd/ssl/etcd-node3.key get /service/--prefix

# start postgres manual
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/postgresql/data start

```
