### What is PgBouncer?

PgBouncer is a lightweight PostgreSQL connection pooler.

It sits between your application and PostgreSQL.

Instead of your app creating direct connections to Postgres, it connects to PgBouncer.

PgBouncer keeps a pool of database connections and reuses them for incoming client requests.

**Why?**
Because PostgreSQL connections are expensive (memory + startup time). If your app opens hundreds or thousands of short-lived connections, Postgres will choke. PgBouncer smooths this out.

## Why Use PgBouncer?

- Reduce connection overhead

  - Opening/closing a PostgreSQL connection can take ~20–30ms and consume ~5–10MB of memory.
  - With PgBouncer, the app always reuses existing connections.

- Handle high concurrency

  - Web apps often use connection pools per worker/thread. With 50 app servers × 20 workers, you can quickly hit 1000s of DB connections → Postgres slows down.
  - PgBouncer limits active connections (say 100), while queueing others.

- Stability & efficiency

  - Protects the database from overload.
  - Useful when scaling horizontally (many app instances).

## Pooling Modes in PgBouncer

1. PgBouncer supports 3 pooling modes:

   - Session pooling (default)
   - A server connection is assigned to a client connection for the whole session. Best for apps that rely on session features (e.g., temp tables, prepared statements).

2. Transaction pooling

   - A server connection is assigned only for the duration of a transaction, then returned to the pool.
   - Much more efficient (supports thousands of clients with fewer DB connections).
   - But not suitable if the app uses session-specific features (e.g., session variables, advisory locks).

3. Statement pooling
   - A server connection is assigned per SQL statement.
   - Most efficient, but very restrictive (breaks prepared statements, transactions, session features).

👉 In most production apps, people use transaction pooling for maximum performance.

## Where to Place PgBouncer

1. On each app server (sidecar mode)

   - Run PgBouncer alongside your NestJS backend.
   - Each backend only talks to its local PgBouncer.
   - PgBouncer then talks to HAProxy → Patroni cluster.
   - ✅ Best for scaling horizontally (more app servers = more PgBouncer pools).

2. As a separate service (shared pool)
   - Run PgBouncer on one or more dedicated servers.
   - Your NestJS backend connects to PgBouncer → PgBouncer connects to HAProxy → Patroni.
   - ✅ Centralized control, but a single PgBouncer node can become a bottleneck if not HA.

## Architecture

I will choose to install pgbouncer on the same haproxy server

```scss
NestJS App Servers
      │
      ▼
   PgBouncer (on same server as HAProxy)
      │
      ▼
   HAProxy (routes to leader/replicas via Patroni)
      │
      ▼
Patroni Cluster (1 Leader + 2 Replicas)
```

## Setup

```bash
sudo apt-get update
sudo apt-get install -y pgbouncer

```

Configure PgBouncer _(/etc/pgbouncer/pgbouncer.ini)_

Example _(assuming HAProxy listens on 127.0.0.1:**6432** for leader)_:

```bash
[databases]
# to allow access only for specific databases, you can create record for each one.

#contact_rewards = host=127.0.0.1 port=6432 dbname=contact_rewards

# Wildcard to accept any database name
* = host=127.0.0.1 port=6432

[pgbouncer]
listen_port = 5000
listen_addr = 0.0.0.0

auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction
max_client_conn = 5000
default_pool_size = 200

# Ignore specific startup parameters to avoid issues with client connections
ignore_startup_parameters = extra_float_digits,options
```

Add users for PgBouncer _(/etc/pgbouncer/userlist.txt)_

```bash
"myuser" "mypassword"
```

(This must match a valid Postgres role.)

PgBouncer runs as a service automatically:

```bash
sudo systemctl enable pgbouncer
sudo systemctl start pgbouncer
```

Connect NestJS backend to PgBouncer

```bash
postgres://myuser:mypassword@haproxy-server-ip:6432/mydb
```

## Official documentation

https://www.pgbouncer.org/config.html
