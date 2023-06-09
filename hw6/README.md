##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #6 "Кластер Patroni" (Занятия "Кластер Patroni on-premise" 1 и 2)

1. Создал и запустил три VM в GCE для кластера `etcd`:
```
for i in {1..3};
  do gcloud compute instances create etcd$i \
    --machine-type=e2-small \
    --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230523 \
    --zone=us-east4-a &\
  done;  
```

2. Установил на три VM `etcd`:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && \
    sudo apt -y install etcd' \
    --zone=us-east4-a &\
  done;
```

Остановил сервисы `etcd`:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo systemctl stop etcd' \
    --zone=us-east4-a &\
  done;
```

Добавил конфиги на все три узла:
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

Запустил сервисы `etcd`:
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

Кластер `etcd` настроен и работает!

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

5. Удалил дефолтный кластер PostgreSQL, установил `patroni`:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo apt install -y python3-pip joe && sudo systemctl stop postgresql@15-main &&\
    sudo -u postgres pg_dropcluster 15 main && sudo pip3 install psycopg2-binary patroni[etcd] &&\
    sudo ln -s /usr/local/bin/patroni /bin/patroni' & \
  done;
```

Создал сервис `patroni`:
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

Распространил конфиг `patroni`:

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

Запустил `patroni` и активировал его сервис:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo systemctl enable patroni && sudo systemctl start patroni' & \
  done;
```

Проверил работу кластера `patroni`:
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

Кластер `patroni` настроен и работает!

6. На ВМ с кластерами PostgreSQL установил `pgbouncer`:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo apt -y install pgbouncer -y' & \
  done;
```

и записал конфиг:
```
for i in {1..3};
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
    --zone=us-west4-b & \
  done;
```

Получил хеш суперпользователя `postgres` СУБД PostgreSQL из представления pg_shadow:
```
postgres=# select passwd from pg_shadow where usename='postgres';
                                                                passwd                                                                 
---------------------------------------------------------------------------------------------------------------------------------------
 SCRAM-SHA-256$4096:L6YQ1rLqKXBEUBqErVQ5WA==$Wtpo1zWYe2C0+93AyyyS7zkNrD+DTJSX38Q0OVBxnWM=:OE0pBmeFisRqW+533BYkog+pfnRIxqJ6QWu9ItEiCAE=
(1 row)
```

Добавил аутентификационную информацию для `pgbouncer`:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i \
    --command='cat > temp.cfg <<\EOF 
"postgres" "SCRAM-SHA-256$4096:L6YQ1rLqKXBEUBqErVQ5WA==$Wtpo1zWYe2C0+93AyyyS7zkNrD+DTJSX38Q0OVBxnWM=:OE0pBmeFisRqW+533BYkog+pfnRIxqJ6QWu9ItEiCAE="
EOF
    cat temp.cfg | sudo tee /etc/pgbouncer/userlist.txt' \
    --zone=us-west4-b & \
  done;
```

Рестартовал сервис `pgbouncer`:
```
for i in {1..3};
  do gcloud compute ssh pgsql$i --zone=us-west4-b \
    --command='sudo systemctl restart pgbouncer' & \
  done;
```

На первом узле кластера `patroni` (лидер на текущий момент) создал БД:
```
postgres@pgsql1:~$ psql -h 127.0.0.1
psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# create database otus;
CREATE DATABASE
```

Подготовил набор данных для тестирования и запустил тест, используя
`pgbouncer`:

```
postgres@pgsql1:~$ pgbench -p 6432 -i -d otus -h 10.182.0.47
Password: 
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.56 s (drop tables 0.02 s, create tables 0.01 s, client-side generate 0.36 s, vacuum 0.07 s, primary keys 0.11 s).

postgres@pgsql1:~$ pgbench -p 6432 -c 20 -C -T 20 -P 1 -d otus -h 1
Password: 

[skipped output]
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 1
maximum number of tries: 1
duration: 20 s
number of transactions actually processed: 1875
number of failed transactions: 0 (0.000%)
latency average = 205.771 ms
latency stddev = 295.243 ms
average connection time = 6.732 ms
tps = 93.552142 (including reconnection times)
```

Проверил работу консоли `pgbouncer`:


```
postgres@pgsql1:~$ psql -h 127.0.0.1 -p 6432 -d pgbouncer
psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1), server 1.19.0/bouncer)
WARNING: psql major version 15, server major version 1.19.
         Some psql features might not work.
Type "help" for help.

pgbouncer=# show stats_totals;
 database  | xact_count | query_count | bytes_received | bytes_sent | xact_time | query_time | wait_time 
-----------+------------+-------------+----------------+------------+-----------+------------+-----------
 otus      |       1898 |       13162 |        2384414 |     522341 | 381596349 |  353454480 |    169124
 pgbouncer |          1 |           1 |              0 |          0 |         0 |          0 |         0
(2 rows)
```

`pgbouncer` настроен и работает!


7. Создал и запустил VM для haproxy:
```
gcloud compute instances create haproxy \
  --machine-type=e2-small \
  --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230523 \
  --zone=us-west4-b
```

Зашёл на VM:
```
gcloud compute ssh haproxy --zone=us-west4-b
```

Установил haproxy 2.8:
```
root@haproxy:~# apt install -y --no-install-recommends software-properties-common &&
add-apt-repository -y ppa:vbernat/haproxy-2.8 &&
apt install -y haproxy=2.8.*
```

Установил клиент PostgreSQL для проверки соединения:
```
root@haproxy:~# apt update && apt upgrade -y -q && apt -y install postgresql-client-common postgresql-client
```

Проверил соединение с первым узлом кластера `patroni` через `pgbouncer`:
```
root@haproxy:~$ psql -p 6432 -d otus -h 10.182.0.47 -U postgres
Password for user postgres: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 15.3 (Ubuntu 15.3-1.pgdg20.04+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=#
```

Сконфигурировал haproxy:
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
    server pgsql1 10.182.0.47:6432 check port 8008
    server pgsql2 10.182.0.46:6432 check port 8008
    server pgsql3 10.182.0.48:6432 check port 8008
 
listen pgReadOnly
    bind *:5433
    mode tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql1 10.182.0.47:6432 check port 8008
    server pgsql2 10.182.0.46:6432 check port 8008
    server pgsql3 10.182.0.48:6432 check port 8008
EOF
```

Перезагрузил сервис для применения конфигурации:
```
systemctl restart haproxy
```

Проверил подключение через haproxy в режиме read-write:
```
root@haproxy:~# psql -h 127.0.0.1 -d otus -U postgres -p 5432
Password for user postgres: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 15.3 (Ubuntu 15.3-1.pgdg20.04+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=# create database haproxy;
CREATE DATABASE
```

Проверил подключение через haproxy в режиме read-only:
```
root@haproxy:~# psql -h 127.0.0.1 -d otus -U postgres -p 5433
Password for user postgres: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 15.3 (Ubuntu 15.3-1.pgdg20.04+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=# create database haproxy;
ERROR:  cannot execute CREATE DATABASE in a read-only transaction
otus=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 haproxy   | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(5 rows)
```

haproxy настроен и работает!

8. Тестируем отказоустойчивость.

Посмотрел состояние кластера на текущий момент:
```
root@pgsql1:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pgsql1 | 10.182.0.47 | Leader  | running |  3 |           |
| pgsql2 | 10.182.0.46 | Replica | running |  3 |         0 |
| pgsql3 | 10.182.0.48 | Replica | running |  3 |         0 |
+--------+-------------+---------+---------+----+-----------+
```

Перевёл лидера на третий узел кластера:
```
root@pgsql1:~# patronictl -c /etc/patroni.yml switchover --leader pgsql1 --candidate pgsql3 --scheduled now --force
Current cluster topology
+ Cluster: patroni ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pgsql1 | 10.182.0.47 | Leader  | running |  3 |           |
| pgsql2 | 10.182.0.46 | Replica | running |  3 |         0 |
| pgsql3 | 10.182.0.48 | Replica | running |  3 |         0 |
+--------+-------------+---------+---------+----+-----------+
2023-06-04 20:02:52.94051 Successfully switched over to "pgsql3"
+ Cluster: patroni ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pgsql1 | 10.182.0.47 | Replica | stopped |    |   unknown |
| pgsql2 | 10.182.0.46 | Replica | running |  3 |         0 |
| pgsql3 | 10.182.0.48 | Leader  | running |  3 |           |
+--------+-------------+---------+---------+----+-----------+
```

Посмотрел состояние кластера, переключение прошло:
```
root@pgsql1:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pgsql1 | 10.182.0.47 | Replica | running |  4 |         0 |
| pgsql2 | 10.182.0.46 | Replica | running |  4 |         0 |
| pgsql3 | 10.182.0.48 | Leader  | running |  4 |           |
+--------+-------------+---------+---------+----+-----------+
```

Подключился через haproxy, проверил работу:
```
root@haproxy:~# psql -h 127.0.0.1 -d otus -U postgres -p 5432
Password for user postgres: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 15.3 (Ubuntu 15.3-1.pgdg20.04+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=# create database switchover;
CREATE DATABASE
```

Остановил VM с третьим узлом кластера `patroni`:
```
bash-5.1$ gcloud compute instances stop pgsql3 --zone=us-west4-b
Stopping instance(s) pgsql3...done.                                                                                                              
```

Посмотрел состояние кластера, узел исчез из кластера, новый лидер на
первом узле:
```
root@pgsql1:~# patronictl -c /etc/patroni.yml list
+ Cluster: patroni ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pgsql1 | 10.182.0.47 | Leader  | running |  5 |           |
| pgsql2 | 10.182.0.46 | Replica | running |  5 |         0 |
+--------+-------------+---------+---------+----+-----------+
```

Подключился через haproxy, проверил работу:
```
root@haproxy:~# psql -h 127.0.0.1 -d otus -U postgres -p 5432
Password for user postgres: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 15.3 (Ubuntu 15.3-1.pgdg20.04+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
Type "help" for help.

otus=# create database leader_node_out;
CREATE DATABASE
```

Отказоустойчивость протестирована!

9. Резервное копирование.

Установил на узлы кластера `patroni` репозиторий софта резервного
копирования и восстановления `pg_probackup` и саму программу:
```
for i in haproxy pgsql1 pgsql2 pgsql3;
  do gcloud compute ssh $i --zone=us-west4-b \
    --command='echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" |\
    sudo tee -a /etc/apt/sources.list.d/pg_probackup.list \
  && sudo wget -O - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP \
  | sudo apt-key add - \
  && sudo apt-get update && sudo apt install pg-probackup-15 -y' & \
  done;
```

Создал в кластере `patroni` отдельную роль для резервного копирования:
```
postgres=# CREATE ROLE backup LOGIN REPLICATION PASSWORD 'strong_backup_password';
CREATE ROLE
```

Настроил права доступа в БД `otus` для роли `backup` согласно документации
`pg_probackup`:

```
BEGIN;
GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
COMMIT;
```

создал каталог для резервных копий:
```
root@haproxy:~# mkdir /home/pgbackups
```

Инициализировал его:
```
root@haproxy:~# pg_probackup-15 init -B /home/pgbackups
INFO: Backup catalog '/home/pgbackups' successfully initialized
```

Сгенерировал пару ssh-ключей с пустой парольной фразой и добавил публичный
ключ в `~/.ssh/authorized_keys` пользователю `postgres` на три узла
кластера `patroni` для работы `pg_probackup` в удалённом режиме.

Добавил определение копируемого экземпляра:
```
root@haproxy:~# pg_probackup-15 add-instance -B /home/pgbackups --instance hw6 --remote-host=pgsql3 \
--remote-user=postgres -D /var/lib/postgresql/15/main
INFO: Instance 'hw6' successfully initialized
```

Настроил конфигурацию параметров соединения и сжатия:
```
root@haproxy:~# pg_probackup-15 set-config -B /home/pgbackups --instance hw6 \
  --compress-algorithm=zlib --compress-level=2 \
  --remote-host=pgsql3 --remote-user=postgres -U backup -d otus
```

Настроил вход пользователя `backup` в кластер PostgreSQL на ведущий и
резервный сервер без ввода пароля:
```
root@haproxy:~# echo "pgsql1:5432:*:backup:strong_backup_password" > ~/.pgpass
root@haproxy:~# echo "pgsql2:5432:*:backup:strong_backup_password" >> ~/.pgpass
root@haproxy:~# echo "pgsql3:5432:*:backup:strong_backup_password" >> ~/.pgpass
root@haproxy:~# chmod 600 ~/.pgpass
```

Для резервного копирования пользователю backup необходимо настроить доступ к
PostgreSQL к БД replication, для этого добавил эти настройки:
```
  pg_hba:
    - host all all 127.0.0.1/32 trust
    - host replication replicanto 10.0.0.0/8 md5
    - host replication backup 10.0.0.0/8 md5
    - host all all 10.0.0.0/8 md5
```
с помощью `patronictl -c /etc/patroni.yml edit-config`

Сделал полную резервную копию с третьего узла кластера (реплика на текущий
момент) в два параллельных потока:
```
root@haproxy:~# pg_probackup-15 backup -B /home/pgbackups --instance hw6 -b FULL -j 2 --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw6, backup ID: RVR5UB, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Backup RVR5UB is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw6/RVR5UB/database/pg_wal/00000005000000000000000B to be streamed
INFO: PGDATA size: 73MB
INFO: Current Start LSN: 0/B3FD580, TLI: 5
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 3s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/C000060
WARNING: Could not read WAL record at 0/C000060: invalid record length at 0/C000060: wanted 24, got 0
INFO: Wait for LSN 0/C000060 in streamed WAL segment /home/pgbackups/backups/hw6/RVR5UB/database/pg_wal/00000005000000000000000C
WARNING: Could not read WAL record at 0/C000060: invalid record length at 0/C000060: wanted 24, got 0
WARNING: Could not read WAL record at 0/C000060: invalid record length at 0/C000060: wanted 24, got 0
INFO: Getting the Recovery Time from WAL
INFO: Failed to find Recovery Time in WAL, forced to trust current_timestamp
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 5s
INFO: Validating backup RVR5UB
INFO: Backup RVR5UB data files are valid
INFO: Backup RVR5UB resident size: 53MB
INFO: Backup RVR5UB completed
```

Сделал полную резервную копию с первого узла кластера (лидер на текущий
момент) в два параллельных потока:
```
root@haproxy:~# pg_probackup-15 backup -B /home/pgbackups --instance hw6 -b FULL -j 2 --stream --temp-slot -h pgsql1
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw6, backup ID: RVR63P, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw6/RVR63P/database/pg_wal/00000005000000000000000E to be streamed
INFO: PGDATA size: 73MB
INFO: Current Start LSN: 0/E000028, TLI: 5
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 3s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/E00DA08
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 6s
INFO: Validating backup RVR63P
INFO: Backup RVR63P data files are valid
INFO: Backup RVR63P resident size: 37MB
INFO: Backup RVR63P completed
```
Проверил наличие копий в каталоге резервных копий:
```
root@haproxy:~# pg_probackup-15 show -B /home/pgbackups --instance hw6 
==================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI    Time  Data   WAL  Zratio  Start LSN  Stop LSN   Status 
==================================================================================================================================
 hw6       15       RVR63P  2023-06-04 23:47:54+00  FULL  STREAM    5/0     12s  21MB  16MB    3.47  0/E000028  0/E00DA08  OK     
 hw6       15       RVR5UB  2023-06-04 23:42:16+00  FULL  STREAM    5/0     17s  21MB  32MB    3.47  0/B3FD580  0/C000060  OK     
```

Резервное копирование настроено!

---
