##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #12 "Parallel cluster" (Занятие "Массивно параллельные кластера PostgreSQL")

- Создал 4 VM в GCE (2 CPU, 8GB RAM, 50GB SSD)
```
for i in {1..4};
  do gcloud compute instances create gp$i --zone=us-east4-a \
    --machine-type=e2-standard-2 \
    --boot-disk-size=50GB --boot-disk-type=pd-ssd \
    --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230628 &\
  done;
```
- Добавил пользователя `gpadmin` и создал пару ssh-ключей:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='sudo groupadd gpadmin && \
    sudo useradd gpadmin -r -m -g gpadmin && \
    echo gpadmin:gpadmin123 | sudo chpasswd && \
    sudo usermod -aG sudo gpadmin && \
    sudo -u gpadmin ssh-keygen -t rsa -b 4096 -q -f /home/gpadmin/.ssh/id_rsa -N ""' \
    --zone=us-east4-a &\
  done;
```
- Добавил репозиторий Greenplum от Ubuntu версии 18.04
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='cat > repo.list << EOF 
deb http://ppa.launchpad.net/greenplum/db/ubuntu bionic main
deb http://ru.archive.ubuntu.com/ubuntu bionic main
EOF
    cat repo.list | sudo tee -a /etc/apt/sources.list.d/greenplum-ubuntu-db-bionic.list' \
    --zone=us-east4-a & \
  done;
```
- Закрепил этот репозиторий в качестве источника пакетов для Greenplum:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='cat > repo.pin << EOF 
Package: greenplum*
Pin: release v=18.04
Pin-Priority: 1
EOF
    cat repo.pin | sudo tee -a /etc/apt/preferences.d/99-greenplum' \
    --zone=us-east4-a & \
  done;
```
- Добавил gpg-ключ Ubuntu:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='gpg --keyserver keyserver.ubuntu.com --recv 3C6FDC0C01D86213 && \
    gpg --export --armor 3C6FDC0C01D86213 | sudo apt-key add -' \
    --zone=us-east4-a &\
  done;
```
- Обновил список пакетов, сами пакеты, установил sshpass и Greenplum:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && \
    sudo apt -y install sshpass greenplum-db-6' \
    --zone=us-east4-a &\
  done;
```
- Сменил владельца каталога с установленным Greenplun на `gpadmin`:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='sudo chown -R gpadmin.gpadmin /opt/greenplum* && \
    sudo chgrp -R gpadmin /opt/greenplum*' \
    --zone=us-east4-a &\
  done;
```
- Добавил инициализацию переменных окружения, необходимых для работы
Greenplum:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='sudo chsh -s /bin/bash gpadmin && \
    cat > bashrc << EOF
source /opt/greenplum-db-$(dpkg -s greenplum-db-6 | grep -i "^Version" | cut -d" " -f2 | cut -d"-" -f1)/greenplum_path.sh
EOF
    cat bashrc | sudo tee -a ~gpadmin/.bashrc' \
    --zone=us-east4-a &\
  done;
```
- Включил возможность входа по ssh с использованием пароля:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='sudo sed -i s@"PasswordAuthentication no"@"PasswordAuthentication yes"@ /etc/ssh/sshd_config && \
    sudo systemctl restart ssh' \
    --zone=us-east4-a &\
  done;
```
- Раскидал ssh-ключи по виртуальным машинам:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='for j in {1..4}; do sudo su -c "SSHPASS=gpadmin123 sshpass -v -e ssh-copy-id -o StrictHostKeyChecking=no gp$j" gpadmin & done' \
    --zone=us-east4-a &\
  done;
```

- Добавил репозиторий Ubuntu 16.04 для библиотеки libssl1.0.0 и установил её:
```
for i in {1..4};
  do gcloud compute ssh gp$i \
    --command='sudo echo "deb http://security.ubuntu.com/ubuntu xenial-security main">> /etc/apt/sources.list && \
    sudo apt update && sudo apt install libssl1.0.0 -y' \
    --zone=us-east4-a &\
  done;
```
- Создал директорию для данных на master- и standby-нодах с владельцем gpadmin:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo mkdir -p /data/master && \
    sudo chown -R gpadmin.gpadmin /data/master' \
    --zone=us-east4-a &\
  done;
```
- Подготовил структуру директорий на сегментах:
```
for i in {3..4};
  do gcloud compute ssh gp$i \
    --command='sudo mkdir -p /data/primary /data/mirror && \
    sudo chown -R gpadmin.gpadmin /data/primary /data/mirror' \
    --zone=us-east4-a &\
  done;
```

- На master-ноде в новую директорию скопировал конфиг СУБД по умолчанию
```
gpadmin@gp1:~$ mkdir gpconfigs
gpadmin@gp1:~$ cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config \
  /home/gpadmin/gpconfigs/
```
- Определил необходимые параметры конфигурации:
```
gpadmin@gp1:~$ cat >>gpconfigs/gpinitsystem_config<<EOF
MASTER_HOSTNAME=gp1
declare -a DATA_DIRECTORY=(/data/primary)
MASTER_PORT=5432
MASTER_DIRECTORY=/data/master
MIRROR_PORT_BASE=7000
declare -a MIRROR_DATA_DIRECTORY=(/data/mirror)
EOF
```
- Создал конфигурационный файл со списком имен сегментов:
```
gpadmin@gp1:~$ cat >>gpconfigs/hostfile_gpinitsystem<<EOF
gp3
gp4
EOF
```
- Инициализировал Greenplum с поддержкой мирроринга командой:
```
gpinitsystem -c gpconfigs/gpinitsystem_config -h gpconfigs/hostfile_gpinitsystem -s gp2 --mirror-mode=spread
```
- Проверил состояние:
```
gpadmin@gp1:~$ export MASTER_DATA_DIRECTORY=/data/master/gpseg-1
gpadmin@gp1:~$ gpstate
20230717:22:55:53:031407 gpstate:gp1:gpadmin-[INFO]:-Starting gpstate with args: 
20230717:22:55:53:031407 gpstate:gp1:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 6.25.0 build commit:a7ff2cf4eeff7456d41f7d5ca5f20d6b008ad819 Open Source'
20230717:22:55:53:031407 gpstate:gp1:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 9.4.26 (Greenplum Database 6.25.0 build commit:a7ff2cf4eeff7456d41f7d5ca5f20d6b008ad819 Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0, 64-bit compiled on Jul 13 2023 13:46:16'
20230717:22:55:53:031407 gpstate:gp1:gpadmin-[INFO]:-Obtaining Segment details from master...
20230717:22:55:53:031407 gpstate:gp1:gpadmin-[INFO]:-Gathering data from segments...
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-Greenplum instance status summary
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Master instance                                           = Active
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Master standby                                            = gp2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Standby master state                                      = Standby host passive
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total segment instance count from metadata                = 4
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Primary Segment Status
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total primary segments                                    = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total primary segment valid (at master)                   = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total primary segment failures (at master)                = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files missing              = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files found                = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs found                 = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files missing                   = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files found                     = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes missing                 = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes found                   = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Mirror Segment Status
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total mirror segments                                     = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total mirror segment valid (at master)                    = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total mirror segment failures (at master)                 = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files missing              = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files found                = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs found                 = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files missing                   = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files found                     = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes missing                 = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes found                   = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number mirror segments acting as primary segments   = 0
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-   Total number mirror segments acting as mirror segments    = 2
20230717:22:55:54:031407 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
```

- Установил на master-ноде репозиторий ПО для работы с бакетами:
```
root@gp1:~# echo "deb https://packages.cloud.google.com/apt gcsfuse-$(lsb_release -c -s) main" \
| tee /etc/apt/sources.list.d/gcsfuse.list
root@gp1:~# curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

- Установил ПО для работы с бакетами и обновил ОС:
```
root@gp1:~# apt update && apt install fuse gcsfuse -y && apt upgrade -y
```

- Смонтировал каталог с набором данных в локальную директорию с read-only доступом для всех пользователей:
```
root@gp1:~# mkdir /tmp/taxi && gcsfuse -o allow_other,ro chicago10 /tmp/taxi
I0611 18:27:20.150658 2023/06/11 18:27:20.150621 Start gcsfuse/0.42.5 (Go version go1.20.3) for app "" using mount point: /home/bbc/data
```

- Сделал локальную копию данных:
```
gpadmin@gp1:~$ cp -av /tmp/taxi /tmp/taxi_local
```

- Создал БД, подключился:
```
gpadmin@gp1:~$ createdb taxi

gpadmin@gp1:~$ psql -d taxi
psql (9.4.26)
Type "help" for help.

taxi=# 
```

- Создал колоночную таблицу:
```
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
```

- Загрузил в неё данные из бакета с поездками чикагского такси:
```
bbc@hw8:~$ time for f in /tmp/taxi_local/taxi*
do
        echo -e "Processing $f file..."
        psql -d taxi -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"
done
Processing /tmp/taxi_local/taxi.csv.000000000000 file...
COPY 668818
..
Processing /tmp/taxi_local/taxi.csv.000000000039 file...
COPY 629855

real	2m53.801s
user	0m14.679s
sys	0m42.515s
```

Выполнил пару раз аналитический запрос:
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

Time: 7054.400 ms
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

Time: 7377.038 ms
```

Запрос выполняется почти в два раза быстрее, чем при развёртывании Greenplum
на одной ВМ с такими же характеристиками и более чем в пять раз быстрее,
чем в ванильном PostgreSQL (см. [ДЗ#8](https://github.com/dimvol2/otus_pgcloud/tree/main/hw8))

---
