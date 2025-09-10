## Simple Haproxy Configuration

The following configuration will send all traffic (read & write) to current Patroni leader

```bash
frontend postgres_frontend
    bind *:5000 # can be any port
    mode tcp
    default_backend postgres_backend

backend postgres_backend
    mode tcp
    option tcp-check
    option httpchk GET /primary  # patroni provides an endpoint to check node roles
    http-check expect status 200  # expect 200 for the primary node
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    server postgresql-01 192.168.137.101:5432 port 8008 check check-ssl verify none
    server postgresql-02 192.168.137.102:5432 port 8008 check check-ssl verify none
```

To check current patroni leader run the following from any postgres node

```bash
sudo patronictl -c /etc/patroni/config.yml list
```

```bash
curl -I --cacert /etc/etcd/ssl/ca.crt https://127.0.0.1:8008/primary
# if leader: HTTP/1.0 200 OK
# if not leader: HTTP/1.0 503 Service Unavailable
# you can check /primary /replica /leader ...
```
