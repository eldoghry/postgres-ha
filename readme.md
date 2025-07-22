# PostgreSQL High Availability with Patroni, etcd, and HAProxy

This repository provides a robust solution for achieving high availability (HA) for your PostgreSQL database using a combination of Patroni, etcd, and HAProxy. This setup ensures automatic failover, reliable service discovery, and efficient load balancing, minimizing downtime and maximizing database uptime.

## Table of Contents

- [PostgreSQL High Availability with Patroni, etcd, and HAProxy](#postgresql-high-availability-with-patroni-etcd-and-haproxy)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Architecture](#architecture)
  - [Nodes Ips](#nodes-ips)
  - [Required Firewall Ports](#required-firewall-ports)
  - [PostgreSQL](#postgresql)
  - [ETCD](#etcd)
      - [Certificates](#certificates)
      - [Running etcd](#running-etcd)
  - [PostgreSQL and Patroni](#postgresql-and-patroni)
    - [patroni](#patroni)
      - [Verifying Our Postgres Cluster](#verifying-our-postgres-cluster)
      - [Editing your pg\_hba after bootstrapping](#editing-your-pg_hba-after-bootstrapping)
  - [HAProxy](#haproxy)
- [Backup using pgbackrest toole](#backup-using-pgbackrest-toole)
- [🔍 Useful Monitoring \& Debugging Commands](#-useful-monitoring--debugging-commands)
  - [👁️ Patroni](#️-patroni)
  - [🔐 etcd](#-etcd)
  - [🧪 HAProxy](#-haproxy)

## Introduction

In modern applications, database availability is paramount. A single point of failure in your database infrastructure can lead to significant downtime and business disruption. This project addresses this challenge by providing a battle-tested architecture for PostgreSQL high availability. By leveraging Patroni for intelligent cluster management, etcd for distributed consensus, and HAProxy for seamless client redirection, we create a resilient and self-healing PostgreSQL cluster.

This project demonstrates how to set up a **highly available PostgreSQL cluster** using:

- [Patroni](https://github.com/zalando/patroni) for PostgreSQL replication and failover
- [etcd](https://etcd.io/) for distributed consensus
- [HAProxy](http://www.haproxy.org/) for load balancing and routing connections
- [pgBackRest](https://pgbackrest.org/) for robust backups and point-in-time recovery (PITR)

## Architecture

- **_Clients_**: Applications connect to the database through HAProxy.

- **_HAProxy_**: Acts as a load balancer and a single entry point for database connections. It intelligently routes traffic to the current PostgreSQL primary, it can be one node or cluster later.

- **_Patroni Cluster_**: A group of PostgreSQL instances managed by Patroni.

  - **_PostgreSQL Primary_**: The active read/write database instance.

  - **_PostgreSQL Replicas_**: Read-only copies of the primary, ready to take over in case of a primary failure, you can add more than one replicas.

- **etcd Cluster**: A distributed key-value store used by Patroni for leader election, cluster state management, and configuration storage, it can be on separate host but for seek of simplicity I will install it
  Haproxy, DB primary and DB replica hosts.

## Nodes Ips

```bash
192.168.137.100 # Node 1 [HAProxy + ETCD 1]
192.168.137.101 # Node 2 [Primary DB + ETCD 2]
192.168.137.102 # Node 3 [Standby DB + ETCD 3]
```

## Required Firewall Ports

Ensure the following ports are open between the nodes and allowed in any firewall configuration (UFW, iptables, security groups, etc.).

|Port |Protocol| Used By |Purpose| Direction|
|'5432'| TCP| PostgreSQL| Database client connections (internal use only)| Between Patroni nodes|
|'8008'| TCP |Patroni| REST API for status/health/administration| Internal & optional external for |monitoring
|'2379'| TCP| etcd| etcd client communication| From Patroni to etcd|
|'2380'| TCP| etcd| etcd peer communication| Between etcd nodes|
|'5000'| TCP| HAProxy| Public PostgreSQL access via HAProxy| From clients to HAProxy|
|'7000'| TCP| HAProxy| HAProxy statistics and health monitor (optional)| From admins to HAProxy|
|'22'| TCP| SSH| Remote access to servers (if applicable)| Admin access|

## PostgreSQL

In case postgres not installed on db nodes

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

Stop and disable the service because Patroni will handle the lifecycle of postgres

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

## ETCD

We can use one but for best performance we should use 3 for Quorum, so this should be installed on all nodes.

```bash
sudo apt update

## Optional: helper package may you need
sudo apt-get install -y wget curl iproute2 telnet iputils-ping nano openssl

# for latest release check https://github.com/etcd-io/etcd/releases
wget https://github.com/etcd-io/etcd/releases/download/v3.6.0/etcd-v3.6.0-linux-amd64.tar.gz
```

Uncompress and rename.

```bash
tar xvf etcd-v3.6.0-linux-amd64.tar.gz
```

Move all binaries into /usr/local/bin/ for later use.

```bash
sudo mv etcd/etcd* /usr/local/bin/
```

check etcd version

```bash
etcd --version
etcdctl version

```

Let’s create a user for etcd service.

```bash
sudo useradd --system --home /var/lib/etcd --shell /bin/false etcd

```

Before configuring etcd, we need to repeat all of these steps for all nodes that will should be used on etcd cluster.

Let’s configure etcd.

Make dir and edit file.

```bash
sudo mkdir -p /etc/etcd
sudo mkdir -p /etc/etcd/ssl
```

#### Certificates

We’ll be securing the communication between etcd nodes and postgres.
We will use **_self signed certificate_**.

**On our own machine (not servers!)**

Make sure openssl is installed

```bash
sudo apt install openssl
openssl version

mkdir certs
cd certs

```

Now let’s generate cert authority, this will be used to signed

```bash
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=etcd-ca" -days 7300 -out ca.crt

# 7300 will be valid for 20 years
```

Generate certificate each node. Note, pay attention to SANS, I am using IP, update with your IP and oh DNS/hostname.

Node 1 (192.168.137.100)

```bash
# Generate a private key
openssl genrsa -out etcd-node1.key 2048

# Create temp file for config
cat > temp.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[ req_distinguished_name ]
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 192.168.137.100
IP.2 = 127.0.0.1
EOF

# Create a csr
openssl req -new -key etcd-node1.key -out etcd-node1.csr \
  -subj "/C=US/ST=YourState/L=YourCity/O=YourOrganization/OU=YourUnit/CN=etcd-node1" \
  -config temp.cnf

# Sign the cert
openssl x509 -req -in etcd-node1.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd-node1.crt -days 7300 -sha256 -extensions v3_req -extfile temp.cnf

# Verify the cert and be sure you see Subject Name Alternative

openssl x509 -in etcd-node1.crt -text -noout | grep -A1 "Subject Alternative Name"

# Remove temp file
rm temp.cnf
```

Node 2 (192.168.137.101)

```bash
# Generate a private key
openssl genrsa -out etcd-node2.key 2048

# Create temp file for config
cat > temp.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[ req_distinguished_name ]
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 192.168.137.101
IP.2 = 127.0.0.1
EOF

# Create a csr
openssl req -new -key etcd-node2.key -out etcd-node2.csr \
  -subj "/C=US/ST=YourState/L=YourCity/O=YourOrganization/OU=YourUnit/CN=etcd-node2" \
  -config temp.cnf

# Sign the cert
openssl x509 -req -in etcd-node2.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd-node2.crt -days 7300 -sha256 -extensions v3_req -extfile temp.cnf

# Verify the cert and be sure you see Subject Name Alternative

openssl x509 -in etcd-node2.crt -text -noout | grep -A1 "Subject Alternative Name"

# Remove temp file
rm temp.cnf
```

Node 3 (192.168.137.102)

```bash
# Generate a private key
openssl genrsa -out etcd-node3.key 2048

# Create temp file for config
cat > temp.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[ req_distinguished_name ]
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 192.168.137.102
IP.2 = 127.0.0.1
EOF

# Create a csr
openssl req -new -key etcd-node3.key -out etcd-node3.csr \
  -subj "/C=US/ST=YourState/L=YourCity/O=YourOrganization/OU=YourUnit/CN=etcd-node3" \
  -config temp.cnf

# Sign the cert
openssl x509 -req -in etcd-node3.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd-node3.crt -days 7300 -sha256 -extensions v3_req -extfile temp.cnf

# Verify the cert and be sure you see Subject Name Alternative

openssl x509 -in etcd-node3.crt -text -noout | grep -A1 "Subject Alternative Name"

# Remove temp file
rm temp.cnf
```

Secure copy (scp) the certs to each node:

```bash
scp ca.crt etcd-node1.crt etcd-node1.key serveradmin@192.168.137.100:/tmp/
scp ca.crt etcd-node2.crt etcd-node2.key serveradmin@192.168.137.101:/tmp/
scp ca.crt etcd-node3.crt etcd-node3.key serveradmin@192.168.137.102:/tmp/
```

So continue etcd setup.

On each node we should do the following

We need to move certs from /tmp to ssl location (/etc/etcd/ssl)

```bash
sudo mkdir -p /etc/etcd/ssl
sudo mv /tmp/etcd-node*.crt /etc/etcd/ssl/
sudo mv /tmp/etcd-node*.key /etc/etcd/ssl/
sudo mv /tmp/ca.crt /etc/etcd/ssl/
sudo chown -R etcd:etcd /etc/etcd/
sudo chmod 600 /etc/etcd/ssl/etcd-node*.key
sudo chmod 644 /etc/etcd/ssl/etcd-node*.crt /etc/etcd/ssl/ca.crt
```

Create our config

```bash
sudo nano /etc/etcd/etcd.env
```

Node 1

```bash
ETCD_NAME="watcher-etcd"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="watcher-etcd=https://192.168.137.100:2380,primary-db-etcd=https://192.168.137.101:2380,standby-db-etcd=https://192.168.137.102:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.137.100:2380"
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.137.100:2379"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd-node1.crt"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-node1.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd-node1.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-node1.key"
```

Node 2

```bash
ETCD_NAME="primary-db-etcd"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="watcher-etcd=https://192.168.137.100:2380,primary-db-etcd=https://192.168.137.101:2380,standby-db-etcd=https://192.168.137.102:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.137.101:2380"
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.137.101:2379"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd-node2.crt"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-node2.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd-node2.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-node2.key"
```

Node 3

```bash
ETCD_NAME="standby-db-etcd"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="watcher-etcd=https://192.168.137.100:2380,primary-db-etcd=https://192.168.137.101:2380,standby-db-etcd=https://192.168.137.102:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.137.102:2380"
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.137.102:2379"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd-node3.crt"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-node3.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd-node3.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-node3.key"
```

Now let’s create a service for etcd on all 3 nodes

```bash
sudo nano /etc/systemd/system/etcd.service
```

Contents of service file, same for all 3 nodes

```bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
EnvironmentFile=/etc/etcd/etcd.env
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000
User=etcd
Group=etcd

[Install]
WantedBy=multi-user.target
```

We need to create a directory for etcd ETCD_DATA_DIR defined in service file.

```bash
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

#### Running etcd

Reload daemon and enable the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd

# Start etcd and check status
sudo systemctl start etcd
sudo systemctl status etcd

# You can also check the logs for etcd by running:
journalctl -xeu etcd.service
```

Once cluster is running, we should verify it’s working on each by running

```bash
 sudo etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ssl/ca.crt \
--cert=/etc/etcd/ssl/etcd-node1.crt \
--key=/etc/etcd/ssl/etcd-node1.key \
endpoint health
# https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 8.144438ms
```

check member list on etcd cluster

```bash
sudo etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ssl/ca.crt \
--cert=/etc/etcd/ssl/etcd-node1.crt \
--key=/etc/etcd/ssl/etcd-node1.key \
member list
```

should see something like that

```bash
3b39f09e1800676f, started, primary-db-etcd, https://192.168.137.101:2380, https://192.168.137.101:2379, false
82d45632c365ff47, started, watcher-etcd, https://192.168.137.100:2380, https://192.168.137.100:2379, false
8bc57ecd50273519, started, standby-db-etcd, https://192.168.137.102:2380, https://192.168.137.102:2379, false
```

Now we should now edit the variables since the cluster is bootstrapped

```bash
sudo systemctl restart etcd

 sudo etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ssl/ca.crt \
--cert=/etc/etcd/ssl/etcd-node1.crt \
--key=/etc/etcd/ssl/etcd-node1.key \
endpoint health

sudo etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ssl/ca.crt \
--cert=/etc/etcd/ssl/etcd-node1.crt \
--key=/etc/etcd/ssl/etcd-node1.key \
member list
```

You should see something like this, specifically one of the nodes IS LEADER = true

```bash
sudo etcdctl \
  --endpoints=https://192.168.137.100:2379,https://192.168.137.101:2379,https://192.168.137.102:2379 \
  --cacert=/etc/etcd/ssl/ca.crt \
  --cert=/etc/etcd/ssl/etcd-node1.crt \
  --key=/etc/etcd/ssl/etcd-node1.key \
  endpoint status --write-out=table
```

```bash
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.137.100:2379 | 59afb19d7cb2565d |  3.5.17 |   20 kB |      true |      false |         2 |         12 |                 12 |        |
| https://192.168.137.101:2379 | 6338565ebcb76aa2 |  3.5.17 |   20 kB |     false |      false |         2 |         12 |                 12 |        |
| https://192.168.137.102:2379 | 9d74b3125c745c74 |  3.5.17 |   20 kB |     false |      false |         2 |         12 |                 12 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

## PostgreSQL and Patroni

Once this is all set up and working, we can now configure postgres and patroni.

We need to create some dirs for postgres

```bash
sudo mkdir -p /var/lib/postgresql/data
sudo mkdir -p /var/lib/postgresql/ssl
```

Notice we are using ssl, we need to generate a certificate
Generate a self-signed certificate

**On our own machine (not servers!)**

```bash
openssl genrsa -out server.key 2048 # private key
openssl req -new -key server.key -out server.req # csr
openssl req -x509 -key server.key -in server.req -out server.crt -days 7300 # generate cert, valid for 20 years



# Move files to cert location
sudo mv server.crt server.key server.req /var/lib/postgresql/ssl
```

After doing this, we need to copy this to our other servers

Copy files locally from (your local machine!) to node1, node2, node3 using scp

scp them to node1, node2, and node3

```bash
scp server.crt server.key server.req serveradmin@192.168.137.100:/tmp
scp server.crt server.key server.req serveradmin@192.168.137.101:/tmp
scp server.crt server.key server.req serveradmin@192.168.137.102:/tmp
```

move certificate on each postgres node to its location

```bash
cd /tmp
sudo mv server.crt server.key server.req /var/lib/postgresql/ssl
```

update permission on each node

```bash
sudo chmod 600 /var/lib/postgresql/ssl/server.key
sudo chmod 644 /var/lib/postgresql/ssl/server.crt
sudo chmod 600 /var/lib/postgresql/ssl/server.req
sudo chown postgres:postgres /var/lib/postgresql/data
sudo chown postgres:postgres /var/lib/postgresql/ssl/server.*
```

create PEM file to be used later on patroni configration

```bash
sudo sh -c 'cat /var/lib/postgresql/ssl/server.crt /var/lib/postgresql/ssl/server.key > /var/lib/postgresql/ssl/server.pem'
sudo chown postgres:postgres /var/lib/postgresql/ssl/server.pem
sudo chmod 600 /var/lib/postgresql/ssl/server.pem
```

We can verify with:

```bash
sudo openssl x509 -in /var/lib/postgresql/ssl/server.pem -text -noout

```

We will need to give the postgres user read access to the etcd certificates using acls

```bash
sudo apt update
sudo apt install -y acl

## on each node update certificate name
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/etcd-node1.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/etcd-node1.key
```

### patroni

install patroni

```bash
sudo apt install -y patroni
```

make a dir for patroni

```bash
sudo mkdir -p /etc/patroni/
```

Create a config file and edit

```bash
sudo nano /etc/patroni/config.yml
```

then on each node add the following config with changing apporpiate congif

Node 2

```bash
scope: cn-postgresql-cluster
namespace: /service/
name: primary-db  # node2

etcd3:
  hosts: 192.168.137.100:2379,192.168.137.101:2379,192.168.137.102:2379  # etcd cluster nodes
  protocol: https
  cacert: /etc/etcd/ssl/ca.crt
  cert: /etc/etcd/ssl/etcd-node2.crt  # node2's etcd certificate
  key: /etc/etcd/ssl/etcd-node2.key  # node2's etcd key

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.137.101:8008  # IP for node2's REST API
  certfile: /var/lib/postgresql/ssl/server.pem

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # Failover parameters
    postgresql:
        parameters:
            ssl: 'on'  # Enable SSL
            ssl_cert_file: /var/lib/postgresql/ssl/server.crt  # PostgreSQL server certificate
            ssl_key_file: /var/lib/postgresql/ssl/server.key  # PostgreSQL server key
        pg_hba:  # Access rules
        - hostssl replication replicator 127.0.0.1/32 md5
        - hostssl replication replicator 192.168.137.101/32 md5
        - hostssl replication replicator 192.168.137.102/32 md5
        - hostssl all all 127.0.0.1/32 md5
        - hostssl all all 0.0.0.0/0 md5
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.137.101:5432  # IP for node2's PostgreSQL
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/16/bin  # Binary directory for PostgreSQL 16
  authentication:
    superuser:
      username: postgres
      password: postgres  # Superuser password - be sure to change
    replication:
      username: replicator
      password: replicatorpassword  # Replication password - be sure to change
  parameters:
    max_connections: 100
    shared_buffers: 256MB

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

Node 3

```bash
scope: cn-postgresql-cluster
namespace: /service/
name: standby-db  # node3

etcd3:
  hosts: 192.168.137.100:2379,192.168.137.101:2379,192.168.137.102:2379  # etcd cluster nodes
  protocol: https
  cacert: /etc/etcd/ssl/ca.crt
  cert: /etc/etcd/ssl/etcd-node3.crt  # node3's etcd certificate
  key: /etc/etcd/ssl/etcd-node3.key  # node3's etcd key

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.137.102:8008  # IP for node3's REST API
  certfile: /var/lib/postgresql/ssl/server.pem

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # Failover parameters
    postgresql:
        parameters:
            ssl: 'on'  # Enable SSL
            ssl_cert_file: /var/lib/postgresql/ssl/server.crt  # PostgreSQL server certificate
            ssl_key_file: /var/lib/postgresql/ssl/server.key  # PostgreSQL server key
        pg_hba:  # Access rules
        - hostssl replication replicator 127.0.0.1/32 md5
        - hostssl replication replicator 192.168.137.101/32 md5
        - hostssl replication replicator 192.168.137.102/32 md5
        - hostssl all all 127.0.0.1/32 md5
        - hostssl all all 0.0.0.0/0 md5
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.137.102:5432  # IP for node3's PostgreSQL
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/16/bin  # Binary directory for PostgreSQL 16
  authentication:
    superuser:
      username: postgres
      password: postgres  # Superuser password - be sure to change
    replication:
      username: replicator
      password: replicatorpassword  # Replication password - be sure to change
  parameters:
    max_connections: 100
    shared_buffers: 256MB

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

starting patroni

```bash
 sudo systemctl restart patroni
```

Check logs

```bash
 journalctl -u patroni -f
```

after patroni is started and worked normally we need to update etcd config on all nodes

```bash
sudo nano /etc/etcd/etcd.env

#update ETCD_INITIAL_CLUSTER_STATE from "new" to be "existing"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

#### Verifying Our Postgres Cluster

We now have a ha postgres cluster!

However we don’t always know who the leader is so we can’t use an IP

We can test the patroni endpoint to see who is leader

```bash
## this will result json, check role value should be "primary" | "replica"
curl -k https://192.168.137.101:8008/primary
curl -k https://192.168.137.102:8008/primary
```

#### Editing your pg_hba after bootstrapping

Now patroni is responsible to manage postgres and its config, assume after booting strap we want to update on pg_hba roles.

If you ever want to see your global config you can with

```bash
 sudo patronictl -c /etc/patroni/config.yml show-config
```

If you ever want to edit it, you can with:

```bash
 sudo patronictl -c /etc/patroni/config.yml edit-config
```

After saving these will be replicated to all nodes

## HAProxy

install and create HAproxy config

```bash
sudo apt -y install haproxy
sudo nano /etc/haproxy/haproxy.cfg
```

add the following configuration, you can adjust this based on your needs

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
    option httpchk GET /replica
    http-check expect status 200
    balance roundrobin

    # HTTPS health check for Patroni
    default-server check port 8008 check-ssl verify none
    default-server inter 2s fall 2 rise 1
    default-server maxconn 100

    server primary-db 192.168.137.101:5432 check
    server standby-db 192.168.137.102:5432 check

# RECOMMENDED: Auto-primary endpoint using /leader
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

start haproxy

```bash
sudo systemctl reload haproxy
```

check logs

```bash
sudo tail -f /var/log/syslog | grep haproxy
```

You can check state of current environment using http://192.168.137.100:7000/stats

# Backup using pgbackrest toole

coming soon

# 🔍 Useful Monitoring & Debugging Commands

## 👁️ Patroni

| Task                            | Command                                                          |
| ------------------------------- | ---------------------------------------------------------------- |
| Check Patroni member status     | `curl -k https://localhost:8008`                                 |
| Show full cluster status (JSON) | `curl -k https://localhost:8008/patroni`                         |
| Switchover primary              | `sudo patronictl -c /etc/patroni/config.yml switchover`          |
| Failover to a new leader        | `sudo patronictl -c /etc/patroni/config.yml failover`            |
| Restart node cleanly            | `sudo patronictl -c /etc/patroni/config.yml restart <node-name>` |
| Reload config                   | `sudo patronictl -c /etc/patroni/config.yml reload`              |
| Monitor logs                    | `journalctl -u patroni -f`                                       |

## 🔐 etcd

all command shoud start with

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ssl/ca.crt \
--cert=/etc/etcd/ssl/etcd-nodeX.crt \
--key=/etc/etcd/ssl/etcd-nodeX.key \
[COMMAND]
```

| Task                  | Command                               |
| --------------------- | ------------------------------------- |
| View cluster health   | ... endpoint health                   |
| View member list      | ... member list                       |
| View leader           | ... endpoint status --write-out=table |
| Watch keys live       | ... watch --prefix / --recursive      |
| View cluster keys     | ... get / --prefix                    |
| View logs             | journalctl -u etcd -f                 |
| Check etcdctl version | etcdctl version                       |

## 🧪 HAProxy

| Task                                 | Command                                                           |                |
| ------------------------------------ | ----------------------------------------------------------------- | -------------- |
| View HAProxy live stats (if enabled) | Open `http://<haproxy-ip>:7000` in browser                        |                |
| View active connections              | \`ss -tnlp                                                        | grep haproxy\` |
| Reload HAProxy config                | `sudo systemctl reload haproxy`                                   |                |
| Check HAProxy logs                   | `sudo tail -f /var/log/haproxy.log` or `journalctl -u haproxy -f` |                |
| Test connection to PostgreSQL        | `psql -h 127.0.0.1 -p 5000 -U postgres`                           |                |

#Referance

- https://technotim.live/posts/postgresql-high-availability
