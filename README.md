# Postgres HA with Patroni, etcd, HAProxy + Keepalived, and Grafana HA

## Purpose
Step-by-step operational guide to build a production-like PostgreSQL HA stack using:
- **Patroni** (Postgres clustering)
- **etcd** (DCS)
- **HAProxy + Keepalived** (VIP for a stable DB endpoint)
- **Grafana** (running in Kubernetes, using HA Postgres DB)

This document captures setup, configs, verification, and troubleshooting so your team can reproduce the setup for production.

---

## Architecture (Quick View)
- VIP `10.67.74.137` is the single DB endpoint apps use.
- HAProxy decides which backend (Patroni node) is primary via Patroni REST API (`/master` or `/health`).
- Patroni + etcd manage cluster state and automatic failover.
- Grafana uses Postgres DB at the VIP; pods are stateless and DB persists dashboards/users.

---

## Inventory & IP

B. Install & Configure etcd (DCS)
1. Download etcd binaries
ETCD_VER=v3.5.15
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}.tar.gz
tar xvf /tmp/etcd-${ETCD_VER}.tar.gz -C /tmp
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/

2. Config /etc/default/etcd (postgres-1 example)
ETCD_NAME=postgres-1
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://10.67.74.134:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.67.74.134:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.67.74.134:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.67.74.134:2379"
ETCD_INITIAL_CLUSTER="postgres-1=http://10.67.74.134:2380,postgres-2=http://10.67.74.135:2380,postgres-3=http://10.67.74.136:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="pg-etcd-cluster"

3. Systemd service /etc/systemd/system/etcd.service
[Unit]
Description=etcd key-value store
After=network.target

[Service]
Type=notify
EnvironmentFile=/etc/default/etcd
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.yml
Restart=always
RestartSec=5s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target

4. Start & verify cluster
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
export ETCDCTL_API=3
etcdctl --endpoints="http://10.67.74.134:2379,http://10.67.74.135:2379,http://10.67.74.136:2379" endpoint health
etcdctl --endpoints="http://10.67.74.134:2379" member list

C. Install PostgreSQL & Patroni
Patroni config /etc/patroni.yml (postgres-1 example)
scope: postgres-ha
namespace: /db/
name: postgres-1

restapi:
  listen: 10.67.74.134:8008
  connect_address: 10.67.74.134:8008

etcd:
  host: 10.67.74.134:2379,10.67.74.135:2379,10.67.74.136:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      parameters:
        max_connections: 100
        shared_buffers: 256MB
        wal_level: replica
        hot_standby: "on"
  initdb:
  - encoding: UTF8
  - data-checksums

postgresql:
  listen: 10.67.74.134:5432
  connect_address: 10.67.74.134:5432
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/14/bin
  authentication:
    superuser:
      username: postgres
      password: postgres
    replication:
      username: replicator
      password: replpass


Patroni service
[Unit]
Description=Patroni PostgreSQL HA Cluster
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 /usr/local/bin/patroni /etc/patroni.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

Start & verify
sudo systemctl enable --now patroni
curl http://10.67.74.134:8008/cluster

D. HAProxy + Keepalived
Keepalived /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface ens18
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        10.67.74.137/24
    }
}

HAProxy /etc/haproxy/haproxy.cfg
frontend postgres_write
    bind 10.67.74.137:5432
    mode tcp
    default_backend patroni_postgres

backend patroni_postgres
    option httpchk GET /master
    http-check expect status 200
    default-server inter 2s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres-1 10.67.74.134:5432 check port 8008
    server postgres-2 10.67.74.135:5432 check port 8008
    server postgres-3 10.67.74.136:5432 check port 8008

listen stats
    bind 10.67.74.137:7000
    mode http
    stats enable
    stats uri /
    stats refresh 5s
    stats auth admin:admin123

Verify HAProxy
psql -h 10.67.74.137 -U grafana -d grafanadb -c "SELECT 1;"

E. Grafana on Kubernetes
Secrets
kubectl create secret generic grafana-db-secret -n postgres-test \
  --from-literal=GF_DATABASE_PASSWORD='postgres'

kubectl create secret generic grafana-admin-secret -n postgres-test \
  --from-literal=admin-user=admin --from-literal=admin-password='AdminPass@123'

values-grafana.yaml
replicas: 2
persistence:
  enabled: false

admin:
  existingSecret: grafana-admin-secret
  userKey: admin-user
  passwordKey: admin-password

env:
  GF_DATABASE_TYPE: postgres
  GF_DATABASE_HOST: 10.67.74.137:5432
  GF_DATABASE_NAME: grafanadb
  GF_DATABASE_USER: grafana
  GF_DATABASE_SSL_MODE: disable

envValueFrom:
  GF_DATABASE_PASSWORD:
    secretKeyRef:
      name: grafana-db-secret
      key: GF_DATABASE_PASSWORD

service:
  type: NodePort

Deploy Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install grafana grafana/grafana -n postgres-test -f values-grafana.yaml

F. Tests & Failover Validation

Dashboard persistence: Create dashboard → delete pods → confirm still exists.

User persistence: Create user → delete pods → login works.

Patroni failover: Stop Patroni on leader → new leader elected → Grafana works.

HAProxy node failover: If single HAProxy dies → DB unavailable. Use 2x HAProxy + Keepalived for resilience.


G. Troubleshooting

etcd not found → check /usr/local/bin/etcd.

HAProxy backends down → check Patroni REST API /master.

Grafana using SQLite → check GF_DATABASE_* env vars.

Grafana CrashLoop → missing GF_DATABASE_PASSWORD secret.

Invalid Grafana login → check grafana-admin-secret.

H. Final Notes

Backups: Use WAL archiving, pg_basebackup, or logical backups.

Security: Enable TLS for etcd, Patroni, and Postgres replication. Replace demo passwords.

Monitoring: Use HAProxy stats, Patroni REST, pg metrics.

Scaling: Grafana pods are stateless → scale via Helm. Run multiple HAProxy + Keepalived for HA.

---

