##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #5 "Бэкапы Постгреса" (Занятие "Углубленное изучение бэкапов и репликации")

1. Создал и запустил три VM в GCE (для ведущего и резервного хоста PostgreSQL
и сервера резервных копий):
```
for i in primary standby backup;
  do gcloud compute instances create otus-pgcloud2023-hw5-$i --zone=us-west4-b \
    --machine-type=e2-medium \
    --image-project=ubuntu-os-cloud --image=ubuntu-minimal-2004-focal-v20230524 &\
  done;  
```

2. Установил на три VM СУБД PostgreSQL:
```
for i in primary standby backup;
  do gcloud compute ssh otus-pgcloud2023-hw5-$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
    sudo tee -a /etc/apt/sources.list.d/pgdg.list && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && \
    sudo apt -y install postgresql-15' & \
  done;
```

3. Добавил правило в firewall GCE для пропуска трафика на стандартный порт сервера PostgreSQL:
```
gcloud compute firewall-rules create allow-postgres --allow=tcp:5432 --description="Allow incoming traffic to postgres" --direction=INGRESS
```

На ведущем хосте согласно
https://pgconfigurator.cybertec.at/
и параметрам VM (2CPU 4G RAM) определил оптимальные настройки СУБД и
добавляем их в конфигурационный файл `/var/lib/postgresql/15/main/postgresql.auto.conf`

"alter system set listen_addresses ='*';"

echo "listen_addresses = '*'" >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "shared_buffers = '1024 MB' >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "work_mem = '32 MB' >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "maintenance_work_mem = '320 MB' >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "effective_cache_size = '3 GB' >> /var/lib/postgresql/15/main/postgresql.auto.conf

Создал роли для репликации и для резервного копирования:
```
postgres=# CREATE ROLE replicanto LOGIN REPLICATION ENCRYPTED PASSWORD 'strong_replicanto_password';
CREATE ROLE

postgres=# CREATE ROLE backup LOGIN REPLICATION PASSWORD 'strong_backup_password';
CREATE ROLE
```
и дал им доступ в `pg_hba.conf`:
echo "hostssl replication replicanto 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
echo "hostssl replication backup 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
echo "hostssl postgres backup 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf

Настроил в БД `postgres` права доступа для роли `backup` согласно документации
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

Создал слот репликации:
```
postgres=# SELECT pg_create_physical_replication_slot('standby_slot');
 pg_create_physical_replication_slot 
-------------------------------------
 (standby_slot,)
(1 row)
```

Перезагрузил сервер СУБД для применения параметров конфигурации: 
```
root@otus-pgcloud2023-hw5-primary:~# systemctl restart postgresql
```

На резервном сервере остановил СУБД:

root@otus-pgcloud2023-hw5-standby:~# systemctl stop postgresql

Очистил директорию с данными кластера:
```
postgres@otus-pgcloud2023-hw5-standby:~$ rm -rf /var/lib/postgresql/15/main/* 
```

Создал копию ведущего кластера:
```
postgres@otus-pgcloud2023-hw5-standby:~$ pg_basebackup -D /var/lib/postgresql/15/main -X s -S standby_slot -R -h otus-pgcloud2023-hw5-primary -p 5432 -U replicanto -P
Password: 
22980/22980 kB (100%), 1/1 tablespace
```
Стартовал сервис PostgreSQL:
```
root@otus-pgcloud2023-hw5-standby:~# systemctl start postgresql
```

Проверил результат (сервер должен быть в режиме восстановления):
```
postgres=# SELECT pg_is_in_recovery();
 pg_is_in_recovery 
-------------------
 t
(1 row)
```

На ведущем проверил состояние репликации:
```
postgres=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 1236
usesysid         | 16644
usename          | replicanto
application_name | 15/main
client_addr      | 10.182.0.18
client_hostname  | 
client_port      | 40174
backend_start    | 2023-05-27 20:21:44.624015+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/9A001CF8
write_lsn        | 0/9A001CF8
flush_lsn        | 0/9A001CF8
replay_lsn       | 0/9A001CF8
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2023-05-27 20:22:43.245559+00
```

Создал БД на ведущем кластере PostgreSQL с тестовой таблицей:
```
postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# create table t(s text);
CREATE TABLE
otus=# insert into t values ('eleven'), ('extinction');
INSERT 0 2
otus=# 
```

Проверил их наличие на резервном кластере:
```
postgres=# \c otus 
You are now connected to database "otus" as user "postgres".
otus=# select * from t;
     s      
------------
 eleven
 extinction
(2 rows)
```



---
