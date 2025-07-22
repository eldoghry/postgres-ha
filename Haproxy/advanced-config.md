## Advanced Haproxy Configuration

```bash
global
    maxconn 100 # Max concurrent connections
    log stdout local0

defaults
    mode tcp
    timeout client 30s
    timeout connect 5s
    timeout server 30s
    timeout check 5s # Health check timeout
    retries 2 # Retry health checks twice

# Statistics page
# [OPTIONAL] HAProxy stats page at http://<haproxy-host>:7000/stats
listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats auth admin:password # change me with strong password

# Primary PostgreSQL connection (writes) - HTTPS FIX
# Connects clients to the primary PostgreSQL server (for writes).
listen postgres_primary
    bind *:5000
    mode tcp
    option tcplog

    # HTTPS health check for Patroni
    option httpchk GET /primary
    http-check expect status 200
    # Add SSL verification disable for self-signed certs
    default-server check port 8008 check-ssl verify none
    default-server inter 2s fall 2 rise 1
    default-server on-marked-down shutdown-sessions
    default-server maxconn 100

    server primary-db 192.168.137.101:5432 check weight 100
    server standby-db 192.168.137.102:5432 check weight 100

# Read-only connections (for current standby)
# Load-balancing is round-robin across all healthy replicas.
# Routes read-only queries to replicas (including current primary).
listen postgres_replicas
    bind *:5001
    mode tcp
    option httpchk GET /replica # this will round robin only between replicas, If you want to add primary change /replica to /health or any endpoint return 200 if it run on all nodes
    http-check expect status 200
    balance roundrobin

    # HTTPS health check for Patroni
    default-server check port 8008 check-ssl verify none
    default-server inter 2s fall 2 rise 1
    default-server maxconn 100

    server primary-db 192.168.137.101:5432 check
    server standby-db 192.168.137.102:5432 check

# [optional] RECOMMENDED: Auto-primary endpoint using /leader
# This ensures HAProxy always routes connections to the actual leader, regardless of which node it is.
# This is a safer and simpler endpoint to use for write traffic or tools that want to ensure they connect to the current master without hardcoding it.
# Clients connecting to port 5002 are always routed to the current Patroni leader.
listen postgres_auto_primary
    bind *:5002
    mode tcp
    option tcplog

    # Use /leader endpoint which works on both nodes
    option httpchk GET /leader
    http-check expect status 200

    # HTTPS health check
    default-server check port 8008 check-ssl verify none
    default-server inter 1s fall 1 rise 1
    default-server on-marked-down shutdown-sessions
    default-server maxconn 100

    server primary-db 192.168.137.101:5432 check
    server standby-db 192.168.137.102:5432 check
```
