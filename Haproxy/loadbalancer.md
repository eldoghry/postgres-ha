# Loadbalancing with Haproxy & pgbouncer

### Haproxy

add 2 interfaces front and backend (one for read connection | one for write connection)

- **port: 6432** will accept read/write connection
- **port: 6433** will accept read only connection

```bash
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 2000
    daemon
    # new
    ssl-server-verify none

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  300s
    timeout server  300s

frontend frontend_rw
    # you can use any port
    #bind *:5000
    bind *:6432
    mode tcp
    default_backend backend_rw
    # new config
    option tcplog
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }


# ----------------------------
# Backend used for real traffic
# ----------------------------
backend backend_rw
    mode tcp
    option tcp-check
    # patroni provides an endpoint to check node roles
    option httpchk GET /primary
    # expect 200 for the primary node
    http-check expect status 200

    server postgresql-01 192.168.137.31:5432 check port 8008 check-ssl
    server postgresql-02 192.168.137.32:5432 check port 8008 check-ssl



frontend frontend_read_only
    # you can use any port
    #bind *:5000
    bind *:6433
    mode tcp
    default_backend backend_read_only
    # new config
    option tcplog
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }


# ----------------------------
# Backend used for real traffic
# ----------------------------
backend backend_read_only
    mode tcp
    option tcp-check
    # patroni provides an endpoint to check node roles
    option httpchk GET /replica
    # expect 200 for the primary node
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server postgresql-01 192.168.137.31:5432 check port 8008 check-ssl
    server postgresql-02 192.168.137.32:5432 check port 8008 check-ssl


# ----------------------------
# Backend used only for monitoring
# ----------------------------
backend postgres_backend_monitor
    mode tcp
    option tcp-check
    option httpchk GET /health
    http-check expect status 200

    server postgresql-01 192.168.137.31:5432 check port 8008 check-ssl
    server postgresql-02 192.168.137.32:5432 check port 8008 check-ssl

# Statistics page
# [OPTIONAL] HAProxy stats page at http://<haproxy-host>:7000/stats
listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    # status credentials
    stats auth admin:password
```

### PGbouncer

```bash
sudo nano /etc/pgbouncer/pgbouncer.ini
```

add alias database ( one for read | one for write)
notice that we use **ports** that configured on haproxy

```bash
[databases]
# to allow access only some databases you can add record for each database
#contact_rewards = host=127.0.0.1 port=6432 dbname=contact_rewards
db_aliase_rw = host=127.0.0.1 port=6432 dbname=dbnameexample
db_aliase_read = host=127.0.0.1 port=6433 dbname=dbnameexample

# or if you want to allow all add Wildcard to accept any database name: * = host=127.0.0.1 port=6432
#* = host=127.0.0.1 port=6432

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

# (اختياري) منع PgBouncer من قبول أي non-SSL قبل handshake
#disable_plaintext_connections = 1

### 🔐 SSL from client to PgBouncer
client_tls_sslmode = verify-ca
client_tls_ca_file = /etc/pgbouncer/ssl/ca.crt
client_tls_cert_file = /etc/pgbouncer/ssl/pgbouncer.crt
client_tls_key_file  = /etc/pgbouncer/ssl/pgbouncer.key

### 🔐 SSL from PgBouncer to HAProxy/PostgreSQL
server_tls_sslmode = verify-ca
server_tls_ca_file = /etc/pgbouncer/ssl/ca.crt
server_tls_cert_file = /etc/pgbouncer/ssl/pgbouncer.crt
server_tls_key_file  = /etc/pgbouncer/ssl/pgbouncer.key

# Minimum pool size: Added to maintain idle connections for faster response after inactivity
min_pool_size = 10
# Reserve pool size: Added for burst capacity during spikes
reserve_pool_size = 50
# Reserve pool timeout: Time to wait before using reserve (in seconds)
reserve_pool_timeout = 5
# Maximum connections per database: Added to prevent overload on specific DBs
max_db_connections = 200
# Server lifetime: Close old connections periodically to refresh state (in seconds)
server_lifetime = 600
```

### Typeorm

```typescript
const bundle = fs.readFileSync(join(__dirname, '../../cert/test-bundle.pem')).toString();

{
			type: 'postgres',
			autoLoadEntities: true,
			entities: [`${join(__dirname, '..', '**/*.entity{.ts,.js}')}`],
			synchronize: false,
			replication: {
				master: {
					host: 'loadbalancer_ip',
					port: 5000, // configured on pgbouncer.ini
					username: 'username',
					password: 'password',
					database: 'db_aliase_rw', // configured on pgbouncer.ini
					ssl: {
						rejectUnauthorized: false, // use true if you have a CA cert,
						ca: bundle,
						cert: bundle,
						key: bundle
					}
				},
				slaves: [
					{
						host: 'loadbalancer_ip',
						port: 5000, // configured on pgbouncer.ini
						username: 'username',
						password: 'password',
						database: 'db_aliase_read', // configured on pgbouncer.ini
						ssl: {
							rejectUnauthorized: false,
							ca: bundle,
							cert: bundle,
							key: bundle
						}
					}
				]
			},
			logging: false
		};
```
