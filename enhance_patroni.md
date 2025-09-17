# Enhance Patroni config

## Passwords

create strong password

```bash
openssl rand -base64 32
```

change users password on database

```sql
ALTER USER postgres WITH PASSWORD 'NewSuperSecretPassword123!';
ALTER USER replicator WITH PASSWORD 'AnotherSecretPass456!';
```

## config file

Update patroni config based on server resource
I assume server is (16G ram, 8vpc cpu)

```yaml
scope: cn-postgresql-cluster
namespace: /service/
name: cn-db02 # node2

etcd3:
  hosts: 192.168.137.100:2379,192.168.137.101:2379,192.168.137.102:2379 # etcd cluster nodes
  protocol: https
  cacert: /etc/etcd/ssl/ca.crt
  cert: /etc/etcd/ssl/etcd-db02.crt # node2's etcd certificate
  key: /etc/etcd/ssl/etcd-db02.key # node2's etcd key

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.137.101:8008 # IP for node1's REST API
  certfile: /var/lib/postgresql/ssl/patroni-db02-rest.pem
  cafile: /etc/etcd/ssl/ca.crt

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576 # Failover parameters
    postgresql:
      parameters:
        ssl: "on" # Enable SSL
        ssl_cert_file: /var/lib/postgresql/ssl/server.crt # PostgreSQL server certificate
        ssl_key_file: /var/lib/postgresql/ssl/server.key # PostgreSQL server key
      pg_hba: # Access rules
        - hostssl replication replicator 127.0.0.1/32 md5
        - host replication replicator 10.172.50.20/32 md5
        - hostssl all all 127.0.0.1/32 md5
        - hostssl all all 0.0.0.0/0 md5
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.172.50.19:5432 # IP for node2's PostgreSQL
  data_dir: /var/lib/postgresql/16/main # Data directory for PostgreSQL 16
  bin_dir: /usr/lib/postgresql/16/bin # Binary directory for PostgreSQL 16
  authentication:
    superuser:
      username: postgres
      password: someStrongPassword # Superuser password - be sure to change
    replication:
      username: replicator
      password: someStrongPassword # Replication password - be sure to change
  parameters:
    max_connections: 200 #
    shared_buffers: 4GB # 25% - 40% from ram size
    work_mem: 16MB # controls per-query memory.
    maintenance_work_mem: 512MB # Important for VACUUM/CREATE INDEX
    effective_cache_size: 12GB # tells optimizer how much OS cache is available.
    wal_level: replica
    max_wal_senders: 10
    wal_keep_size: 2GB
    max_replication_slots: 10
    archive_mode: "on"
    archive_command: "pgbackrest --stanza=cn-backup archive-push %p"
    restore_command: "pgbackrest --stanza=cn-backup archive-get %f %p"
    archive_timeout: 300
    logging_collector: on
    log_directory: "pg_log"
    log_filename: "postgresql-%a.log"
    log_rotation_age: 1d
    log_rotation_size: 100MB
    log_statement: "ddl"
    log_min_duration_statement: 5000

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

## maximum_lag_on_failover

_maximum_lag_on_failover_ is one of the most important Patroni safety knobs because it decides when failover is allowed if the leader dies.

```yaml
maximum_lag_on_failover: 1048576 # in bytes (1 MB)
```

- Patroni measures replication lag between leader and replicas.

- If the replica is more than this value behind, Patroni will not promote it automatically on failover.

- This protects you from data loss (promoting a replica that doesn’t have the latest committed transactions).

### ⚖️ Choosing the Right Value

Right now you set:

```yaml
maximum_lag_on_failover: 1048576 # in bytes (1 MB)
```

That’s quite strict. It means if your replica is just 2 MB behind, Patroni won’t fail over. This can cause the whole cluster to be stuck without a leader until the lag clears (or manual intervention).

Recommended approach for production:

- If your application is very sensitive to data loss (banking, payments, orders) → keep it low (1–16 MB).
- If your app can tolerate a few seconds of data loss, but must fail over quickly → set it higher (64MB–256MB). Some teams even set it to 0 (allow any lag), but that risks more data loss.

```yaml
maximum_lag_on_failover: 1048576  # in bytes (1 MB)
maximum_lag_on_failover: 67108864 # in bytes (64 MB) recommended for current setup 2 nodes only on cluster
```
