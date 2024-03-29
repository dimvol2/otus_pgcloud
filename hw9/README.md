##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #9 "Развернуть HA кластер" (Занятие "Работа с кластером высокой доступности")

#### HA кластер на основе `pg_auto_failover`

- Создал 3 VM в GCE (2 CPU, 8GB RAM, 30GB SSD, Ubuntu 22.04):
```
for i in prim sec mon;
  do gcloud compute instances create $i --zone=us-east4-a \
    --machine-type=e2-standard-2 \
    --boot-disk-size=50GB --boot-disk-type=pd-ssd \
    --image-project=ubuntu-os-cloud --image=ubuntu-2204-jammy-v20230714 &\
  done;
```

- Установил PostgreSQL и pg_auto_failover на всех нодах (ведущая, ведомая и
  управляющая):
```
for i in prim sec mon;
  do gcloud compute ssh $i --zone=us-east4-a \
    --command='sudo mkdir /etc/postgresql-common && \
    echo "create_main_cluster = false" | sudo tee -a /etc/postgresql-common/createcluster.conf && \
    curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
    echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -c -s)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && \
    sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -q --no-install-recommends postgresql-15 && \
    sudo DEBIAN_FRONTEND=noninteractive apt-get install pg-auto-failover-cli postgresql-15-auto-failover -y -q' \
    --zone=us-east4-a &\
  done;
```

- Запустил сервер мониторинга HA кластера на ноде `mon`:
```
postgres@mon:~$ pg_autoctl create monitor --no-ssl --pgdata ~/monitor --auth trust --run 
```

- Сконфигурировал на ней доступ к PostgreSQL из локальной подсети:
```
postgres@mon:~$ echo "host all all 10.150.0.0/24 trust" >> ~/monitor/pg_hba.conf
postgres@mon:~$ psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```


- Получил URL мониторингового кластера для рабочих нод:
```
postgres@mon:~$ pg_autoctl show uri --formation monitor --pgdata ~/monitor
postgres://autoctl_node@mon:5432/pg_auto_failover?sslmode=prefer
```

- На ведущей ноде `prim` поднял PostgreSQL, включив её в HA кластер:
```
postgres@prim:~$ pg_autoctl create postgres \
   --auth trust \
   --ssl-self-signed \
   --pgdata ~/primary \
   --monitor postgres://autoctl_node@mon:5432/pg_auto_failover?sslmode=prefer \
   --run
```

- Посмотрел конфигурацию кластера, нода добавилась:
```
postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   1: 0/156E500 |   read-write |              single |              single
```

- Поднял PostgreSQL На ведомой ноде `sec`, включив её в HA кластер:
```
postgres@sec:~$ pg_autoctl create postgres \
   --auth trust \
   --ssl-self-signed \
   --pgdata ~/secondary \
   --monitor postgres://autoctl_node@mon:5432/pg_auto_failover?sslmode=prefer \
   --run
```

- Посмотрел состояние кластера, всё Ok, две ноды, ведущая и ведомая, подключены и работают:
```
postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   1: 0/3000110 |   read-write |             primary |             primary
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   1: 0/3000110 |    read-only |           secondary |           secondary
```

- Установил на ведущую ноду репозиторий ПО для работы с бакетами, само ПО и обновил ОС:
```
root@prim:~# echo "deb https://packages.cloud.google.com/apt gcsfuse-$(lsb_release -c -s) main" \
| tee /etc/apt/sources.list.d/gcsfuse.list
root@prim:~# curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

root@prim:~# apt update && apt install fuse gcsfuse -y && apt upgrade -y
```

- Смонтировал каталог с набором данных (10GB чикагского такси) в локальную директорию:
```
root@prim:~# mkdir /tmp/taxi && gcsfuse -o allow_other,ro chicago10 /tmp/taxi
I0725 20:47:01.794268 2023/07/25 20:47:01.794238 Start gcsfuse/1.0.0 (Go version go1.20.4) for app "" using mount point: /tmp/taxi
```

- Создал БД taxi и в ней таблицу taxi_trips для загружаемых данных:
```
postgres@prim:~$ psql
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database taxi;
CREATE DATABASE
postgres=# \c taxi 
You are now connected to database "taxi" as user "postgres".
taxi=# create table taxi_trips (
unique_key text, 
taxi_id text, 
trip_start_timestamp TIMESTAMP, 
trip_end_timestamp TIMESTAMP, 
trip_seconds bigint, 
trip_miles numeric, 
pickup_census_tract bigint, 
dropoff_census_tract bigint, 
pickup_community_area bigint, 
dropoff_community_area bigint, 
fare numeric, 
tips numeric, 
tolls numeric, 
extras numeric, 
trip_total numeric, 
payment_type text, 
company text, 
pickup_latitude numeric, 
pickup_longitude numeric, 
pickup_location text, 
dropoff_latitude numeric, 
dropoff_longitude numeric, 
dropoff_location text
);
CREATE TABLE
```

- Наполнил таблицу тестовыми данными:
```
postgres@prim:~$ time for f in /tmp/taxi/taxi*
do
        echo -e "Processing $f file..."
        psql -d taxi -U postgres -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done

[skipped]

real	5m6.737s
user	0m4.970s
sys	0m19.193s
```

- Сделал аналитический запрос на обеих нодах:
```
postgres@prim:~$ psql -d taxi
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

taxi=# \timing 
Timing is on.
taxi=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
FROM taxi_trips
group by payment_type
order by 3;
 payment_type | tips_percent |    c     
--------------+--------------+----------
 Prepaid      |            0 |        6
 Way2ride     |           12 |       27
 Split        |           17 |      180
 Dispute      |            0 |     5596
 Pcard        |            2 |    13575
 No Charge    |            0 |    26294
 Mobile       |           16 |    61256
 Prcard       |            1 |    86053
 Unknown      |            0 |   103869
 Credit Card  |           17 |  9224956
 Cash         |            0 | 17231871
(11 rows)

Time: 71023.602 ms (01:11.024)
```

```
postgres@sec:~$ psql -d taxi
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

taxi=# \timing 
Timing is on.
taxi=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
FROM taxi_trips
group by payment_type
order by 3;
 payment_type | tips_percent |    c     
--------------+--------------+----------
 Prepaid      |            0 |        6
 Way2ride     |           12 |       27
 Split        |           17 |      180
 Dispute      |            0 |     5596
 Pcard        |            2 |    13575
 No Charge    |            0 |    26294
 Mobile       |           16 |    61256
 Prcard       |            1 |    86053
 Unknown      |            0 |   103869
 Credit Card  |           17 |  9224956
 Cash         |            0 | 17231871
(11 rows)

Time: 44485.784 ms (00:44.486)
```

- Проверил ведомую ноду, убедился, что PostgreSQL работает в режиме read-only:
```
postgres@sec:~$ psql 
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database ro;
ERROR:  cannot execute CREATE DATABASE in a read-only transaction
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery 
-------------------
 t
(1 row)
```

- Посмотрел состояние кластера:
```
postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |        TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+-----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   1: 5/FCCDA7C0 |   read-write |             primary |             primary
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   1: 5/FCCDA7C0 |    read-only |           secondary |           secondary
```

- Сэмулировал switchover:
```
postgres@mon:~$ pg_autoctl perform switchover --pgdata ~/monitor --wait 10
```

- Первая нода упала, вторая промоутилась до ведущей:
```
postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |        TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+-----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   1: 5/FCCDA7C0 |       none ! |             demoted |          catchingup
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   2: 5/FCCDA92A |   read-write |        wait_primary |        wait_primary
```

- Через некоторое время стало видно, что ноды поменялись ролями:
```
postgres@mon:~$ pg_autoctl show state
  Name |  Node |                                           Host:Port |        TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+-----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   2: 5/FCCDAB98 |    read-only |           secondary |           secondary
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   2: 5/FCCDAB98 |   read-write |             primary |             primary
```

- Ещё один switchover, успешно:
```
postgres@mon:~$ pg_autoctl perform switchover --pgdata ~/monitor --wait 30

postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |        TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+-----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   3: 5/FCCDAD98 |   read-write |        wait_primary |        wait_primary
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   2: 5/FCCDABD0 |         none |             demoted |          catchingup

postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |        TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+-----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   3: 5/FE000110 |   read-write |             primary |             primary
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   3: 5/FE000110 |    read-only |           secondary |           secondary
```

- Сымитировал failover: закрыл исходящие соединения по порту 5432 с текущей ведущей ноды
  (хост `prim`):
```
bbc@prim:~$ sudo ufw enable
bbc@prim:~$ sudo ufw allow 22/tcp
bbc@prim:~$ sudo ufw deny out 5432
```

- Видно, что соединение с одной нодой потеряно, вторая готовится стать
  ведущей:
```
postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |        TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+-----------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   3: 5/FE000148 | read-write ! |             primary |             demoted
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   4: 5/FE0022C0 |   read-write |        wait_primary |        wait_primary
```

- Отключаем firewall на ноде `prim`, чтобы управляющая нода опять стала
  получать информацию от рабочей ноды:
```
bbc@prim:~$ sudo ufw disable
```

- Через некоторое время кластер восстановил состояние, роли нод поменялись
  местами:
```
postgres@mon:~$ pg_autoctl show state --pgdata ~/monitor
  Name |  Node |                                           Host:Port |   TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+-----------------------------------------------------+------------+--------------+---------------------+--------------------
node_1 |     1 | prim.us-east4-a.c.disco-ascent-385720.internal:5432 |   4: 6/148 |    read-only |           secondary |           secondary
node_2 |     2 |  sec.us-east4-a.c.disco-ascent-385720.internal:5432 |   4: 6/148 |   read-write |             primary |             primary
```

Таким образом, HA кластер на основе `pg_auto_failover` был успешно
развёрнут, наполнен данными, а также было протестировано плановое переключение
роли ведущего сервера на другую ноду и сымитирован отказ ведущей ноды

---
