##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #8 "Разворачиваем и настраиваем БД с большими данными" (Занятие "Работа с большим объемом реальных данных")

1. Создал VM в GCE hw8 (Ubuntu 18.04 LTS):
```gcloud compute instances create hw8 --machine-type=e2-standard-2 \
--image-project=ubuntu-os-cloud --image=ubuntu-1804-bionic-v20230605 \
--boot-disk-size=200GB --boot-disk-type=pd-ssd --zone=us-east4-a
```
Вошёл на VM:
```
gcloud compute ssh hw8 --zone=us-east4-a
```

Установил репозиторий:
```
root@hw8:~# echo "deb https://packages.cloud.google.com/apt gcsfuse-$(lsb_release -c -s) main" \
| tee /etc/apt/sources.list.d/gcsfuse.list
root@hw8:~# curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

установил ПО для работы с бакетами и обновил ОС:
```
root@hw8:~# apt update && apt install fuse gcsfuse -y && apt upgrade -y
```

НЕНАДО ??? надо ли?
gcloud auth application-default login

Смонтировал каталог с набором данных в локальную директорию с read-only доступом для всех пользователей
и сделал локальную копию данных:
```
root@hw8:~# mkdir /tmp/taxi && gcsfuse -o allow_other,ro chicago10 /tmp/taxi
I0611 18:27:20.150658 2023/06/11 18:27:20.150621 Start gcsfuse/0.42.5 (Go version go1.20.3) for app "" using mount point: /home/bbc/data
postgres@hw8:~$ cp -av /tmp/taxi /tmp/taxi_local

```

Установил PostgreSQL:
```
root@hw8:~# echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
root@hw8:~# wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
root@hw8:~# apt update
root@hw8:~# apt install -y postgresql-15
```

Создал БД и таблицу для загружаемых данных:
```
postgres@hw8:~$ psql 
psql (15.3 (Ubuntu 15.3-1.pgdg18.04+1))
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

Залил данные в таблицу:
```
time for f in /tmp/taxi_local/taxi*
do
        echo -e "Processing $f file..."
        psql -d taxi -U postgres -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done

Processing /tmp/taxi_local/taxi.csv.000000000000 file...
COPY 668818
...
Processing /tmp/taxi_local/taxi.csv.000000000039 file...
COPY 629855

real	5m39.381s
user	0m10.673s
sys	0m39.159s
```

Для проверки загрузки в нежурналируемую таблицу удалил из таблицы `taxi_trips` данные и
сделал таблицу нежурналируемой:
```
taxi=# truncate table taxi_trips;
TRUNCATE TABLE
taxi=# alter table taxi_trips set unlogged;
ALTER TABLE
```

Повторил загрузку данных в таблицу:
```
time for f in /tmp/taxi_local/taxi*
do
        echo -e "Processing $f file..."
        psql -d taxi -U postgres -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done

Processing /tmp/taxi_local/taxi.csv.000000000000 file...
COPY 668818

Processing /tmp/taxi_local/taxi.csv.000000000039 file...
COPY 629855

real	4m17.274s
user	0m10.473s
sys	0m37.280s
```

Данные заливались примерно на 24% быстрее, а теперь сделаем таблицу журналируемой:
```
taxi=# \timing 
Timing is on.
taxi=# alter table taxi_trips set logged;
ALTER TABLE
Time: 190244.783 ms (03:10.245)
```

Плюс 3 минуты, в итоге вся процедура на 32% дольше, чем заливка данных сразу в журналируемую таблицу.


Настроил PostgreSQL под текущую конфигурацию VM (2 CPU, RAM 8GB, SSD):
```
postgres@hw8:~$ cat >> /var/lib/postgresql/15/main/postgresql.auto.conf<<EOF
# Memory Settings
shared_buffers = '2048 MB'
work_mem = '64 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '6 GB'
effective_io_concurrency = 200 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)
EOF
```

Перезагрузил сервис для применения новых настроек:
```
root@hw8:~# systemctl restart postgresql
```

Установил `pgloader`:
```
root@hw8:~# apt install pgloader
```

/*
Установил `pg_bulkload`:
```
root@hw8:~# apt install make gcc postgresql-server-dev-15 -y
bbc@hw8:~$ curl -L https://github.com/ossc-db/pg_bulkload/releases/download/VERSION3_1_20/pg_bulkload-3.1.20.tar.gz -o pg_bulkload-3.1.20.tar.gz
bbc@hw8:~$ tar axf pg_bulkload-3.1.20.tar.gz 
bbc@hw8:~$ cd pg_bulkload-3.1.20/
```
*/

Протестировал запрос:
```
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

Time: 36742.624 ms (00:36.743)
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

Time: 39759.097 ms (00:39.759)
```

Время выполнения запроса около 38s.






Установил Greenplum DB по инструкции (https://greenplum.org/install-greenplum-oss-on-ubuntu/):
```
root@hw8:~# add-apt-repository ppa:greenplum/db -y
root@hw8:~# apt update && apt install greenplum-db-6 -y
```

```
bbc@hw8:~$ source /opt/greenplum-db-6.24.4/greenplum_path.sh

bbc@hw8:~$ cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_singlenode .

bbc@hw8:~$ cat >>gpinitsystem_singlenode<<EOF
MASTER_HOSTNAME=hw8
declare -a DATA_DIRECTORY=(/home/bbc/primary /home/bbc/primary)
MASTER_PORT=5433
MASTER_DIRECTORY=/home/bbc/master
EOF
```

```
bbc@hw8:~$ cat >>./hostlist_singlenode<<EOF
> hw8
> EOF
```

```
bbc@hw8:~$ gpssh-exkeys -f hostlist_singlenode 
[STEP 1 of 5] create local ID and authorize on local host

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] retrieving credentials from remote hosts

[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts

[INFO] completed successfully
```
///20230613:00:10:52:015733 gpinitsystem:hw8:bbc-[WARN]:-Host hw8 open files limit is 1024 should be >= 65535


echo "export MASTER_DATA_DIRECTORY=/home/bbc/master/gpsne-1" >> ~/.bashrc 

bbc@hw8:~$ createdb taxi -p 5433

bbc@hw8:~$ psql -p 5433 -d taxi
psql (9.4.26)
Type "help" for help.

taxi=# select version();
                                                                                                                  version                        
                                                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------
 PostgreSQL 9.4.26 (Greenplum Database 6.24.4 build commit:d1b94743b5e665888e0a72c921daf04bdbe77528 Open Source) on x86_64-unknown-linux-gnu, com
piled by gcc (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0, 64-bit compiled on Jun  7 2023 19:23:04
(1 row)

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
dropoff_location text)
WITH (
    appendonly = true,
    orientation = column,
    compresstype = zstd,
    compresslevel = 1
);
NOTICE:  Table doesn't have 'DISTRIBUTED BY' clause -- Using column named 'unique_key' as the Greenplum Database data distribution key for this table.
HINT:  The 'DISTRIBUTED BY' clause determines the distribution of data. Make sure column(s) chosen are the optimal data distribution key to minimize skew.
CREATE TABLE


time for f in /tmp/taxi_local/taxi*
do
        echo -e "Processing $f file..."
        psql -d taxi -p 5433 -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done
Processing /tmp/taxi_local/taxi.csv.000000000000 file...
COPY 668818
..
Processing /tmp/taxi_local/taxi.csv.000000000039 file...
COPY 629855

real	4m41.361s
user	0m15.615s
sys	0m37.629s


```
taxi=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c
taxi-# FROM taxi_trips
taxi-# group by payment_type
taxi-# order by 3;
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

Time: 13573.991 ms
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

Time: 13196.389 ms
```

Время выполнения запроса около 13s - в три раза быстрее, чем в PostgreSQL!
