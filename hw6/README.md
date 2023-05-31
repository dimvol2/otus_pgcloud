##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #6 "Кластер Patroni" (Занятия "Кластер Patroni on-premise" 1 и 2)

1. Создал и запустил три VM в GCE для кластера etcd:
```
for i in {1..3};
  do gcloud compute instances create etcd$i \
    --machine-type=e2-small \
    --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230523 &\
    --zone=us-east4-a &\
  done;  
```

2. Установил на три VM etcd:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && \
    sudo apt -y install etcd' \
    --zone=us-east4-a &\
  done;
```

Остановил сервисы etcd:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo systemctl stop etcd' \
    --zone=us-east4-a &\
  done;
```

Добавил конфиги на все три ноды:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='cat > temp.cfg << EOF 
ETCD_NAME="$(hostname)"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://$(hostname -f):2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://$(hostname):2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
EOF
    cat temp.cfg | sudo tee -a /etc/default/etcd' \
    --zone=us-east4-a & \
  done;
```

Запустил сервисы etcd:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo systemctl start etcd' \
    --zone=us-east4-a &\
  done;
```

Проверил состояние кластера:
```
root@etcd1:~# etcdctl cluster-health
member 9a1f33941721f94d is healthy: got healthy result from http://etcd1:2379
member 9df0146dd9068bd2 is healthy: got healthy result from http://etcd3:2379
member f2aeb69aaf7ffcbf is healthy: got healthy result from http://etcd2:2379
cluster is healthy
```

ETCD_INITIAL_CLUSTER_STATE="new"->ETCD_INITIAL_CLUSTER_STATE="existing"


3. Создал и запустил три VM в GCE для кластера PostgreSQL в другой зоне,
чтобы обойти ограничение на 4 IP адреса:
```
for i in {1..3};
  do gcloud compute instances create pgsql$i --zone=us-west4-b \
    --machine-type=e2-small \
    --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230523 &\
  done;  
```

4. Установил на эти три VM PostgreSQL:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
    sudo tee -a /etc/apt/sources.list.d/pgdg.list && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && \
    sudo apt -y install postgresql-15' & \
  done;
```

5. Удалил дефолтный кластер PostgreSQL, установил patroni:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo apt install -y python3-pip joe && sudo systemctl stop postgresql@15-main &&\
    sudo -u postgres pg_dropcluster 15 main && sudo pip3 install psycopg2-binary patroni[etcd] &&\
    sudo ln -s /usr/local/bin/patroni /bin/patroni' & \
  done;
```

Создал systemctl сервис patroni:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i \
    --command='cat > temp.cfg << EOF 
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
    cat temp.cfg | sudo tee -a /etc/systemd/system/patroni.service' \
    --zone=us-west4-b & \
  done;
```

Распространил конфиг patroni:

```
for i in {1..3};
  do gcloud compute ssh pgsql$i \
    --command='cat > temp.cfg << EOF
scope: patroni
name: $(hostname)
restapi:
  listen: $(hostname -I | tr -d " "):8008
  connect_address: $(hostname -I | tr -d " "):8008
etcd:
  hosts: etcd1.us-east4-a.c.disco-ascent-385720.internal:2379,etcd2.us-east4-a.c.disco-ascent-385720.internal:2379,etcd3.us-east4-a.c.disco-ascent-385720.internal:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host replication replicanto 10.0.0.0/8 md5
  - host all all 10.0.0.0/8 md5
  users:
    admin:
      password: strong_admin_password
      options:
        - createrole
        - createdb
postgresql:
  listen: 127.0.0.1, $(hostname -I | tr -d " "):5432
  connect_address: $(hostname -I | tr -d " "):5432
  data_dir: /var/lib/postgresql/15/main
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicanto
      password: strong_replicanto_password
    superuser:
      username: postgres
      password: strong_superuser_password
    rewind:
      username: rewind
      password: strong_rewind_password
  parameters:
    unix_socket_directories: '.'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
EOF
    cat temp.cfg | sudo tee -a /etc/patroni.yml' \
    --zone=us-west4-b & \
  done;
```

Запустил patroni и активировал его сервис:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo systemctl enable patroni && sudo systemctl start patroni' & \
  done;
```

Проверил работу кластера patroni:
```
root@pgsql1:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pgsql1 | 10.182.0.47 | Leader  | running |  1 |           |
| pgsql2 | 10.182.0.46 | Replica | running |  1 |         0 |
| pgsql3 | 10.182.0.48 | Replica | running |  1 |         0 |
+--------+-------------+---------+---------+----+-----------+
```


---
