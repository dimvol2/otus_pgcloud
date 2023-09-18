# Курс `PostgreSQL Cloud Solutions`
### Проектная работа "Построение отказоустойчивого геораспределённого кластера PostgreSQL на инфраструктуре облачной платформы Google

- Реализуем схему отказоустойчивого географически распределённого кластера высокой доступности

КАРТИНКА

- Разворачиваю кластер в локациях: South Carolina, Oregon и London
```
#gcp_zones=("us-east1-b" "us-west1-a" "europe-west2-c")
gcp_zones=("us-east1-b" "us-west1-a" "us-central1-c")

```
#asia-south2 - Dehli
#us-east1 - South Carolina
#us-west1 - Oregon
#europe-west2 - London
#us-central1 - Iowa

- Создал три ВМ в GCP на основе Ubintu 22.04
```
for i in "${!gcp_zones[@]}";
  do gcloud compute instances create etcd$i \
    --machine-type=e2-small \
    --image-project=ubuntu-os-cloud --image=ubuntu-2204-jammy-v20230908 \
    --zone=${gcp_zones[$i]} &\
  done;  
```

- Установил на все ВМ распределённое хранилице данных etcd
```
for i in "${!gcp_zones[@]}";
  do gcloud compute ssh etcd$i \
    --command='sudo DEBIAN_FRONTEND=noninteractive apt update -q && \
    sudo DEBIAN_FRONTEND=noninteractive apt -qy install etcd && \
    sudo systemctl stop etcd' \
    --zone=${gcp_zones[$i]} &\
  done;
```
- Сконфигурировал etcd-кластер
```
for i in "${!gcp_zones[@]}";
  do gcloud compute ssh etcd$i \
    --command='cat > temp.cfg << EOF 
ETCD_NAME="$(hostname)"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://$(hostname -f):2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://$(hostname):2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd0=http://etcd0.us-east1-b.c.disco-ascent-385720.internal:2380,\
etcd1=http://etcd1.us-west1-a.c.disco-ascent-385720.internal:2380,\
etcd2=http://etcd2.us-central1-c.c.disco-ascent-385720.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
EOF
    cat temp.cfg | sudo tee -a /etc/default/etcd' \
    --zone=${gcp_zones[$i]} & \
  done;
```
- Запустил etcd-кластер
```
for i in "${!gcp_zones[@]}";
  do gcloud compute ssh etcd$i \
    --command='sudo systemctl start etcd' \
    --zone=${gcp_zones[$i]} &\
  done;
```
- Проверил его работу
```
root@etcd2:~# etcdctl cluster-health
member 4832398ddd50501d is healthy: got healthy result from http://etcd1.us-west1-a.c.disco-ascent-385720.internal:2379
member b09dbf7e15169004 is healthy: got healthy result from http://etcd0.us-east1-b.c.disco-ascent-385720.internal:2379
member dc639f37416712fe is healthy: got healthy result from http://etcd2.europe-west2-c.c.disco-ascent-385720.internal:2379
cluster is healthy

member 36f9d8e4b4d804a6 is healthy: got healthy result from http://etcd2.us-central1-c.c.disco-ascent-385720.internal:2379
member 4832398ddd50501d is healthy: got healthy result from http://etcd1.us-west1-a.c.disco-ascent-385720.internal:2379
member b09dbf7e15169004 is healthy: got healthy result from http://etcd0.us-east1-b.c.disco-ascent-385720.internal:2379
```
Кластер функционирует.

- Создал ещё три ВМ в GCP на основе Ubintu 22.04 в тех же геозонах
```
for i in ${!gcp_zones[@]};
  do gcloud compute instances create pgsql$i --zone=${gcp_zones[$i]} \
    --machine-type=e2-small \
    --image-project=ubuntu-os-cloud --image=ubuntu-2204-jammy-v20230908 &\
  done;  
```
- Установил на них PostgreSQL 15
```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i --zone=${gcp_zones[$i]} \
    --command='sudo DEBIAN_FRONTEND=noninteractive apt update -qy && \
    echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
    sudo tee -a /etc/apt/sources.list.d/pgdg.list && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && \
    sudo DEBIAN_FRONTEND=noninteractive apt -qy install postgresql-15' & \
  done;
```
- Удалил кластер PostgreSQL, установил patroni
```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i --zone=${gcp_zones[$i]} \
    --command='sudo DEBIAN_FRONTEND=noninteractive apt install -qy python3-pip && sudo systemctl stop postgresql@15-main &&\
    sudo -u postgres pg_dropcluster 15 main && sudo pip3 install psycopg2-binary patroni[etcd] &&\
    sudo ln -s /usr/local/bin/patroni /bin/patroni' & \
  done;
```

- Организовал systemd-сервис для patroni
```
for i in ${!gcp_zones[@]};
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
    --zone=${gcp_zones[$i]} & \
  done;
```

???
synchronous_mode: true
https://patroni.readthedocs.io/en/latest/ha_multi_dc.html#synchronous-replication
(либо асинхронную репликацию с двумя кластерами патрони в двух разных ДЦ)
тогда можно и
https://cloud.google.com/load-balancing/docs/tcp/#gcloud
попробовать, чтобы на чтениях из БД увидеть обращения к разным кластерам
https://cloud.google.com/load-balancing/docs/tcp/setting-up-tcp


- Сконфигурировал patroni
```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i \
    --command='cat > temp.cfg << EOF
scope: patroni
name: $(hostname)
restapi:
  listen: $(hostname -I | tr -d " "):8008
  connect_address: $(hostname -I | tr -d " "):8008
etcd:
  hosts: etcd0.us-east1-b.c.disco-ascent-385720.internal:2379,\
etcd1.us-west1-a.c.disco-ascent-385720.internal:2379,\
etcd2.us-central1-c.c.disco-ascent-385720.internal:2379
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
    --zone=${gcp_zones[$i]} & \
  done;
```
- Активировал автозапуск сервиса и запустил его

```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i --zone=${gcp_zones[$i]} \
    --command='sudo systemctl enable patroni && sudo systemctl start patroni' & \
  done;
```

- Проверил работу кластера patroni
```
root@pgsql0:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+-----------+----+-----------+
| Member | Host        | Role    | State     | TL | Lag in MB |
+--------+-------------+---------+-----------+----+-----------+
| pgsql0 | 10.142.0.13 | Replica | streaming |  1 |         0 |
| pgsql1 | 10.138.0.16 | Replica | streaming |  1 |         0 |
| pgsql2 | 10.128.0.35 | Leader  | running   |  1 |           |
+--------+-------------+---------+-----------+----+-----------+
```
- Установил пулер соединений pgbouncer
```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i --zone=${gcp_zones[$i]} \
    --command='sudo DEBIAN_FRONTEND=noninteractive apt -qy install pgbouncer' & \
  done;
```

- Сконфигурировал его
```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i \
    --command='cat > temp.cfg << EOF 
[databases]
otus = host=127.0.0.1 port=5432 dbname=otus 
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = postgres
EOF
    cat temp.cfg | sudo tee -a /etc/pgbouncer/pgbouncer.ini' \
    --zone=${gcp_zones[$i]} & \
  done;
```

- Добавил аутентификационную информацию для pgbouncer
```
postgres=# select passwd from pg_shadow where usename='postgres';
                                                                passwd                                                                 
---------------------------------------------------------------------------------------------------------------------------------------
 SCRAM-SHA-256$4096:+Sy9AZdtanyfESjIXKf6Ow==$Zk8cNII+cnCWwpzi8lGlR2xlqUB3UEMk0U3rRvuqcEU=:NTVWnxd8ZnSl2EBNRWloOtXpNN5x3Ev+gW7jnq07Qc4=
(1 row)

for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i \
    --command='cat > temp.cfg <<\EOF 
"postgres" "SCRAM-SHA-256$4096:+Sy9AZdtanyfESjIXKf6Ow==$Zk8cNII+cnCWwpzi8lGlR2xlqUB3UEMk0U3rRvuqcEU=:NTVWnxd8ZnSl2EBNRWloOtXpNN5x3Ev+gW7jnq07Qc4="
EOF
    cat temp.cfg | sudo tee /etc/pgbouncer/userlist.txt' \
    --zone=${gcp_zones[$i]} & \
  done;
```

- Рестартовал сервис pgbouncer для применения конфигурации
```
for i in ${!gcp_zones[@]};
  do gcloud compute ssh pgsql$i --zone=${gcp_zones[$i]} \
    --command='sudo systemctl restart pgbouncer' & \
  done;
```

- Создал БД otus и протестировал работу ноды через pgbouncer тестом pgbench по умолчанию
```
root@pgsql2:~# psql -U postgres -h 127.0.0.1
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

postgres=# create database otus;
CREATE DATABASE
```

надо ли этот pgbench??

postgres=# create database otus;
CREATE DATABASE
```
```
root@pgsql2:~# pgbench -U postgres -p 6432 -i -d otus -h 127.0.0.1
Password: 
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.87 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.35 s, vacuum 0.07 s, primary keys 0.44 s).

postgres@pgsql0:~$ pgbench -p 6432 -c 20 -C -T 20 -P 1 -d otus -h 1
Password: 

[skipped output]
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 1396
number of failed transactions: 0 (0.000%)
latency average = 273.553 ms
latency stddev = 131.789 ms
average connection time = 11.193 ms
tps = 69.631026 (including reconnection times)
```
- Проверил работу pgbouncer в консоли
```
postgres@pgsql0:~$ psql -h 127.0.0.1 -p 6432 -d pgbouncer
Password for user postgres: 
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1), server 1.20.0/bouncer)
WARNING: psql major version 15, server major version 1.20.
         Some psql features might not work.
Type "help" for help.

pgbouncer=# show stats_totals;
 database  | xact_count | query_count | bytes_received | bytes_sent | xact_time | query_time | wait_time 
-----------+------------+-------------+----------------+------------+-----------+------------+-----------
 otus      |       1414 |        9804 |        2181618 |     388657 | 379267182 |  346347017 |    138773
 pgbouncer |          1 |           1 |              0 |          0 |         0 |          0 |         0
(2 rows)
```

- Развернул две ВМ под haproxy
```
gcloud compute instances create haproxy1 \
  --machine-type=e2-small \
  --image-project=ubuntu-os-cloud --image=ubuntu-2204-jammy-v20230908 \
  --zone=us-east1-b

gcloud compute instances create haproxy2 \
  --machine-type=e2-small \
  --image-project=ubuntu-os-cloud --image=ubuntu-2204-jammy-v20230908 \
  --zone=us-west1-a
```

#gcloud compute ssh haproxy1 --zone=us-east1-b

- Установил haproxy 2.8 и клиент PostgreSQL со вспомогательными утилитами на обоих ВМ (haproxy1, haproxy2)
```
DEBIAN_FRONTEND=noninteractive apt install -qy --no-install-recommends software-properties-common && \
add-apt-repository -y ppa:vbernat/haproxy-2.8 && apt install -qy haproxy=2.8.*

DEBIAN_FRONTEND=noninteractive apt -qy update && apt -yq install postgresql-client-common postgresql-client
```
#проверка, не нужна??
##psql -p 6432 -d otus -h 10.142.0.7 -U postgres
- Сконфигурировал haproxy на обоих ВМ
```
cat > /etc/haproxy/haproxy.cfg <<EOF
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:\
ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:\
ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
 
listen pgReadWrite
    bind *:5432
    mode tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /master
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql0 pgsql0.us-east1-b.c.disco-ascent-385720.internal:6432 check port 8008
    server pgsql1 pgsql1.us-west1-a.c.disco-ascent-385720.internal:6432 check port 8008
    server pgsql2 pgsql2.us-central1-c.c.disco-ascent-385720.internal:6432 check port 8008

listen pgReadOnly
    bind *:5433
    mode tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql0 pgsql0.us-east1-b.c.disco-ascent-385720.internal:6432 check port 8008
    server pgsql1 pgsql1.us-west1-a.c.disco-ascent-385720.internal:6432 check port 8008
    server pgsql2 pgsql2.us-central1-c.c.disco-ascent-385720.internal:6432 check port 8008
EOF
```
- Рестартовал сервисы на обоих ВМ
```
systemctl restart haproxy
```


HAProxy_screenshot.png
http://34.131.44.32:7000/


root@haproxy1:~# psql -h 127.0.0.1 -d otus -U postgres -p 5433
Password for user postgres: 
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1), server 15.4 (Ubuntu 15.4-1.pgdg22.04+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=# create database haproxy;
ERROR:  cannot execute CREATE DATABASE in a read-only transaction


root@haproxy1:~# psql -h 127.0.0.1 -d otus -U postgres -p 5432
Password for user postgres: 
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1), server 15.4 (Ubuntu 15.4-1.pgdg22.04+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=# create database haproxy;
CREATE DATABASE
otus=# \l haproxy 
                           List of databases
  Name   |  Owner   | Encoding | Collate |  Ctype  | Access privileges 
---------+----------+----------+---------+---------+-------------------
 haproxy | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
(1 row)

gcloud compute instances stop pgsql0 --zone=us-east1-b

HAProxy_degrade_screenshot.png
http://34.131.44.32:7000/


- Настроил Load Balancer по инструкции

https://cloud.google.com/load-balancing/docs/tcp/setting-up-tcp

gcloud compute instance-groups unmanaged create us-ig-east --zone us-east1-b
gcloud compute instance-groups unmanaged create us-ig-west --zone us-west1-a

gcloud compute instance-groups set-named-ports us-ig-east --zone=us-east1-b --named-ports=haproxy-rw:5432,haproxy-ro:5433
gcloud compute instance-groups set-named-ports us-ig-west --zone=us-west1-a --named-ports=haproxy-rw:5432,haproxy-ro:5433

gcloud compute instance-groups unmanaged add-instances us-ig-east --zone=us-east1-b --instances=haproxy1
gcloud compute instance-groups unmanaged add-instances us-ig-west --zone=us-west1-a --instances=haproxy2

gcloud compute firewall-rules create allow-haproxy-ro-rw --allow=tcp:5432,tcp:5433 \
  --description="Allow incoming traffic to haproxy service" --direction=INGRESS

- Настроил health-check
```
gcloud compute health-checks create tcp haproxy-rw-health-check --port 5432
gcloud compute health-checks create tcp haproxy-ro-health-check --port 5433
```
/*
bash-5.1$ gcloud compute backend-services create haproxy-lb --global-health-checks --global --protocol TCP --health-checks haproxy-rw-health-check,haproxy-ro-health-check --timeout 5s --port-name haproxy-rw,haproxy-ro
ERROR: (gcloud.compute.backend-services.create) Could not fetch resource:
 - Value for field 'resource.healthChecks' is too large: maximum size 1 element(s); actual size 2.
*/

!one port by backend
gcloud compute backend-services create lb-haproxy-rw --global-health-checks --global --protocol TCP \
--health-checks haproxy-rw-health-check --timeout 5s --port-name haproxy-rw

gcloud compute backend-services create lb-haproxy-ro --global-health-checks --global --protocol TCP \
--health-checks haproxy-ro-health-check --timeout 5s --port-name haproxy-ro \
--tracking-mode=PER_SESSION --session-affinity CLIENT_IP --idle-timeout-sec=60


gcloud compute backend-services add-backend lb-haproxy-rw --global --instance-group us-ig-west --instance-group-zone us-west1-a \
--balancing-mode UTILIZATION #?? --max-utilization 0.9
gcloud compute backend-services add-backend lb-haproxy-rw --global --instance-group us-ig-east --instance-group-zone us-east1-b \
--balancing-mode UTILIZATION #?? --max-utilization 0.9

gcloud compute backend-services add-backend lb-haproxy-ro --global --instance-group us-ig-west --instance-group-zone us-west1-a \
--balancing-mode UTILIZATION #--max-utilization 0.9
gcloud compute backend-services add-backend lb-haproxy-ro --global --instance-group us-ig-east --instance-group-zone us-east1-b \
--balancing-mode UTILIZATION #--max-utilization 0.9

gcloud compute target-tcp-proxies create proxy-lb-haproxy-rw --backend-service lb-haproxy-rw --proxy-header NONE
/*
gcloud compute addresses list 
NAME                ADDRESS/RANGE   TYPE      PURPOSE  NETWORK  REGION  SUBNET  STATUS
ipv4-lb-haproxy-rw  35.244.159.129  EXTERNAL                                    RESERVED
*/

gcloud compute addresses create ipv4-lb-haproxy-rw --ip-version=IPV4 --global
gcloud compute forwarding-rules create ipv4-lb-haproxy-rw-forward-rule --global \
--target-tcp-proxy proxy-lb-haproxy-rw --address ipv4-lb-haproxy-rw --ports 5432



/*
>gcloud compute addresses describe ipv4-lb-haproxy-rw --global
address: 35.244.159.129
addressType: EXTERNAL
creationTimestamp: '2023-09-16T14:12:29.780-07:00'
description: ''
id: '293204127522727122'
ipVersion: IPV4
kind: compute#address
labelFingerprint: 42WmSpB8rSM=
name: ipv4-lb-haproxy-rw
networkTier: PREMIUM
selfLink: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/global/addresses/ipv4-lb-haproxy-rw
status: IN_USE
users:
- https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/global/forwardingRules/ipv4-lb-haproxy-rw-forward-rule
*/

gcloud compute target-tcp-proxies create proxy-lb-haproxy-ro --backend-service lb-haproxy-ro --proxy-header NONE
gcloud compute addresses create ipv4-lb-haproxy-ro --ip-version=IPV4 --global
gcloud compute forwarding-rules create ipv4-lb-haproxy-ro-forward-rule --global \
--target-tcp-proxy proxy-lb-haproxy-ro --address ipv4-lb-haproxy-ro --ports 5433

/*
bash-5.1$ gcloud compute backend-services get-health lb-haproxy-ro --global
---
backend: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west1-a/instanceGroups/us-ig-west
status:
  healthStatus:
  - healthState: UNHEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west1-a/instances/haproxy2
    ipAddress: 10.138.0.9
    port: 5433
  kind: compute#backendServiceGroupHealth
---
backend: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-east1-b/instanceGroups/us-ig-east
status:
  healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-east1-b/instances/haproxy1
    ipAddress: 10.190.0.3
    port: 5433
  kind: compute#backendServiceGroupHealth


bash-5.1$ gcloud compute backend-services get-health lb-haproxy-ro --global
---
backend: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west1-a/instanceGroups/us-ig-west
status:
  healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west1-a/instances/haproxy2
    ipAddress: 10.138.0.9
    port: 5433
  kind: compute#backendServiceGroupHealth
---
backend: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-east1-b/instanceGroups/us-ig-east
status:
  healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-east1-b/instances/haproxy1
    ipAddress: 10.190.0.3
    port: 5433
  kind: compute#backendServiceGroupHealth


*/


bash-5.1$ pgbench -p 5432 -i -d otus -h 35.244.159.129 -U postgres
Password: 
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.49 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 25.02 s (drop tables 0.80 s, create tables 3.18 s, client-side generate 14.98 s, vacuum 3.25 s, primary keys 2.80 s).


через LB->Dehli->London
bash-5.1$ pgbench -p 5432 -c 20 -C -T 300 -P 1 -d otus -h 35.244.159.129 -U postgres
..
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
duration: 300 s
number of transactions actually processed: 37
latency average = 8911.781 ms
latency stddev = 709.599 ms
average connection time = 5735.139 ms
tps = 0.119890 (including reconnection times)
pgbench: fatal: Run was aborted; the above results are incomplete.

не впечатляет

с haproxy тоже:

Dehli->London
bash-5.1$ pgbench -p 5432 -c 20 -C -T 300 -P 1 -d otus -h 127.0.0.1 -U postgres
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
duration: 300 s
number of transactions actually processed: 121
latency average = 32549.858 ms
latency stddev = 15861.208 ms
average connection time = 1958.430 ms
tps = 0.392332 (including reconnection times)
pgbench: fatal: Run was aborted; the above results are incomplete.


с haproxy
Oregon->London
bash-5.1$ pgbench -p 5432 -c 20 -C -T 300 -P 1 -d otus -h 127.0.0.1 -U postgres
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
duration: 300 s
number of transactions actually processed: 357
latency average = 14157.009 ms
latency stddev = 8342.394 ms
average connection time = 647.713 ms
tps = 1.173501 (including reconnection times)
pgbench: fatal: Run was aborted; the above results are incomplete.


перевёл лидера в Oregon

root@pgsql2:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+-----------+----+-----------+
| Member | Host        | Role    | State     | TL | Lag in MB |
+--------+-------------+---------+-----------+----+-----------+
| pgsql0 | 10.142.0.9  | Replica | streaming |  1 |         0 |
| pgsql1 | 10.138.0.11 | Replica | streaming |  1 |         0 |
| pgsql2 | 10.154.0.10 | Leader  | running   |  1 |           |
+--------+-------------+---------+-----------+----+-----------+
root@pgsql2:~# patronictl -c /etc/patroni.yml switchover --leader pgsql2 --candidate pgsql1 --scheduled now --force
Current cluster topology
+ Cluster: patroni ----+---------+-----------+----+-----------+
| Member | Host        | Role    | State     | TL | Lag in MB |
+--------+-------------+---------+-----------+----+-----------+
| pgsql0 | 10.142.0.9  | Replica | streaming |  1 |         0 |
| pgsql1 | 10.138.0.11 | Replica | streaming |  1 |         0 |
| pgsql2 | 10.154.0.10 | Leader  | running   |  1 |           |
+--------+-------------+---------+-----------+----+-----------+
2023-09-17 00:43:14.21773 Successfully switched over to "pgsql1"
+ Cluster: patroni ----+---------+-----------+----+-----------+
| Member | Host        | Role    | State     | TL | Lag in MB |
+--------+-------------+---------+-----------+----+-----------+
| pgsql0 | 10.142.0.9  | Replica | streaming |  1 |         0 |
| pgsql1 | 10.138.0.11 | Leader  | running   |  1 |           |
| pgsql2 | 10.154.0.10 | Replica | stopped   |    |   unknown |
+--------+-------------+---------+-----------+----+-----------+
root@pgsql2:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+-----------+----+-----------+
| Member | Host        | Role    | State     | TL | Lag in MB |
+--------+-------------+---------+-----------+----+-----------+
| pgsql0 | 10.142.0.9  | Replica | streaming |  1 |         0 |
| pgsql1 | 10.138.0.11 | Leader  | running   |  2 |           |
| pgsql2 | 10.154.0.10 | Replica | stopped   |    |   unknown |
+--------+-------------+---------+-----------+----+-----------+
root@pgsql2:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+-----------+----+-----------+
| Member | Host        | Role    | State     | TL | Lag in MB |
+--------+-------------+---------+-----------+----+-----------+
| pgsql0 | 10.142.0.9  | Replica | streaming |  1 |         0 |
| pgsql1 | 10.138.0.11 | Leader  | running   |  2 |           |
| pgsql2 | 10.154.0.10 | Replica | streaming |  2 |         0 |
+--------+-------------+---------+-----------+----+-----------+

О! Внутри одного ДЦ:

>pgbench -p 5432 -c 20 -C -T 300 -P 1 -d otus -h 127.0.0.1 -U postgres
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
duration: 300 s
number of transactions actually processed: 8506
latency average = 677.491 ms
latency stddev = 605.263 ms
average connection time = 27.697 ms
tps = 28.336286 (including reconnection times)


остановили haproxy в Азии
root@haproxy1:~# systemctl stop haproxy

проверили LB
bash-5.1$ gcloud compute backend-services get-health lb-haproxy-rw --global
---
backend: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west1-a/instanceGroups/us-ig-west
status:
  healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west1-a/instances/haproxy2
    ipAddress: 10.138.0.9
    port: 5432
  kind: compute#backendServiceGroupHealth
---
backend: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-east1-b/instanceGroups/us-ig-east
status:
  healthStatus:
  - healthState: UNHEALTHY
    instance: https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-east1-b/instances/haproxy1
    ipAddress: 10.190.0.3
    port: 5432
  kind: compute#backendServiceGroupHealth

тест
LB->Oregon->Oregon
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
duration: 300 s
number of transactions actually processed: 238
latency average = 4639.274 ms
latency stddev = 1258.132 ms
average connection time = 955.030 ms
tps = 0.786657 (including reconnection times)
pgbench: fatal: Run was aborted; the above results are incomplete.


----
stub
