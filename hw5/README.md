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

`otus-pgcloud2023-hw5-primary` - VM с ведущим сервером PostgreSQL,  
`otus-pgcloud2023-hw5-standby` - VM с резервным сервером PostgreSQL,  
`otus-pgcloud2023-hw5-backup` - VM с сервером резервных копий

2. Установил на три VM СУБД PostgreSQL и утилиту управления контрольными
суммами в кластере PostgreSQL:
```
for i in primary standby backup;
  do gcloud compute ssh otus-pgcloud2023-hw5-$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
    sudo tee -a /etc/apt/sources.list.d/pgdg.list && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && \
    sudo apt -y install postgresql-15 postgresql-15-pg-checksums' & \
  done;
```

3. Добавил правило в firewall GCE для пропуска трафика на стандартный порт сервера PostgreSQL:
```
gcloud compute firewall-rules create allow-postgres --allow=tcp:5432 --description="Allow incoming traffic to postgres" --direction=INGRESS
```

4. Остановил ведущий сервер, включил контрольные суммы в кластере:
```
root@otus-pgcloud2023-hw5-primary:~# systemctl stop postgresql

postgres@otus-pgcloud2023-hw5-primary:~$ /usr/lib/postgresql/15/bin/pg_checksums -D /var/lib/postgresql/15/main --enable
Checksum operation completed
Files scanned:   946
Blocks scanned:  2801
Files written:  778
Blocks written: 2801
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
```

5. На ведущем хосте согласно рекомендациям https://pgconfigurator.cybertec.at/
и параметрам VM (2 CPU, 4G RAM) определил оптимальные настройки СУБД и
добавил их в конфигурационный файл:
```
echo "listen_addresses = '*'" >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "shared_buffers = '1024 MB'" >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "work_mem = '32 MB'" >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "maintenance_work_mem = '320 MB'" >> /var/lib/postgresql/15/main/postgresql.auto.conf
echo "effective_cache_size = '3 GB'" >> /var/lib/postgresql/15/main/postgresql.auto.conf
```

Запустил ведущий сервер:
```
root@otus-pgcloud2023-hw5-primary:~# systemctl start postgresql
```

6. Создал роли для репликации и для резервного копирования:
```
postgres=# CREATE ROLE replicanto LOGIN REPLICATION ENCRYPTED PASSWORD 'strong_replicanto_password';
CREATE ROLE

postgres=# CREATE ROLE backup LOGIN REPLICATION PASSWORD 'strong_backup_password';
CREATE ROLE
```

и дал им доступ в `pg_hba.conf` на ведущем и резервном сервере PostgreSQL:
```
echo "hostssl replication replicanto 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf  
echo "hostssl replication backup 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf 
echo "hostssl backupdb backup 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
```

Перечитал конфигурацию ведущего сервера СУБД для применения параметров конфигурации
из `pg_hba.conf`: 
```
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```

Создал в ведущем кластере отдельную БД для подключения системы резервного копирования:
```
postgres=# CREATE DATABASE backupdb;
CREATE DATABASE
```

Настроил в этой БД права доступа для роли `backup` согласно документации
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

7. На резервном сервере остановил СУБД:
```
root@otus-pgcloud2023-hw5-standby:~# systemctl stop postgresql
```

Очистил директорию с данными кластера:
```
postgres@otus-pgcloud2023-hw5-standby:~$ rm -rf /var/lib/postgresql/15/main/* 
```

Настроил вход пользователя `replicanto` в кластер PostgreSQL на ведущем сервере без ввода пароля:
```
postgres@otus-pgcloud2023-hw5-standby:~$ echo "otus-pgcloud2023-hw5-primary:5432:replication:replicanto:strong_replicanto_password" > ~/.pgpass
postgres@otus-pgcloud2023-hw5-standby:~$ chmod 600 ~/.pgpass
```

Создал базовую копию ведущего кластера PostgreSQL на резервном:
```
postgres@otus-pgcloud2023-hw5-standby:~$ pg_basebackup -D /var/lib/postgresql/15/main -X s -S standby_slot \
  -R -h otus-pgcloud2023-hw5-primary -p 5432 -U replicanto -P
Password: 
30467/30467 kB (100%), 1/1 tablespace
```

Стартовал сервис резервного кластера PostgreSQL:
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

На ведущем сервере PostgreSQL проверил состояние репликации:
```
postgres=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 10649
usesysid         | 16388
usename          | replicanto
application_name | 15/main
client_addr      | 10.182.0.23
client_hostname  | 
client_port      | 40570
backend_start    | 2023-05-28 20:51:44.942703+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/3000060
write_lsn        | 0/3000060
flush_lsn        | 0/3000060
replay_lsn       | 0/3000060
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2023-05-28 20:53:05.152447+00
```

8. Создал БД на ведущем кластере PostgreSQL с тестовой таблицей и тестовыми данными:
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

Проверил наличие объектов на резервном кластере PostgreSQL:
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

Репликация кластера PostgreSQL работает!

9. Установил на сервер резервных копий, на ведущий и резервный сервер PostgreSQL репозиторий софта резервного
копирования и восстановления `pg_probackup`:
```
echo "deb [arch=amd64] \
  https://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" \
  > /etc/apt/sources.list.d/pg_probackup.list \
  && sudo wget -O - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP \
  | sudo apt-key add - \
  && sudo apt-get update
```

Установил пакет `pg_probackup` управления резервным копированием и восстановлением кластеров баз данных
PostgreSQL версии соответствующий используемой мажорной версии СУБД:
```
root@otus-pgcloud2023-hw5-backup:~# apt install pg-probackup-15 -y
root@otus-pgcloud2023-hw5-primary:~# apt install pg-probackup-15 -y
root@otus-pgcloud2023-hw5-standby:~# apt install pg-probackup-15 -y
``` 

10. На сервере резервного копирования создал каталог для резервных копий с
правами доступа для пользователя `postgres`:
```
root@otus-pgcloud2023-hw5-backup:~# mkdir /home/pgbackups && chmod 777 /home/pgbackups \
  && chown postgres.postgres /home/pgbackups
```

Инициализировал каталог:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 init -B /home/pgbackups
INFO: Backup catalog '/home/pgbackups' successfully initialized
```

Сгенерировал пару ssh-ключей с пустой парольной фразой и добавил публичный
ключ в `~/.ssh/authorized_keys` пользователю `postgres` на ведущем и резервном
сервере PostgreSQL для работы `pg_probackup` в удалённом режиме.

Добавил определение копируемого экземпляра:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 add-instance -B /home/pgbackups --instance hw5 \
>   --remote-host=root@otus-pgcloud2023-hw5-standby --remote-user=postgres -D /var/lib/postgresql/15/main
The authenticity of host 'otus-pgcloud2023-hw5-standby (10.182.0.23)' can't be established.
ECDSA key fingerprint is SHA256:qTQW+ac4o/hUAA3O177Nnm/4AYSy0GWEdEOBflEr/30.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
INFO: Instance 'hw5' successfully initialized
```

Настроил конфигурацию параметров соединения и сжатия:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 set-config -B /home/pgbackups --instance hw5 \
  --compress-algorithm=zlib --compress-level=2 \
  --remote-host=otus-pgcloud2023-hw5-standby --remote-user=postgres -U backup -d backupdb
```

Настроил вход пользователя `backup` в кластер PostgreSQL на ведущий и
резервный сервер без ввода пароля:
```
postgres@otus-pgcloud2023-hw5-backup:~$ echo "otus-pgcloud2023-hw5-primary:5432:*:backup:strong_backup_password" > ~/.pgpass
postgres@otus-pgcloud2023-hw5-backup:~$ echo "otus-pgcloud2023-hw5-standby:5432:*:backup:strong_backup_password" >> ~/.pgpass
postgres@otus-pgcloud2023-hw5-backup:~$ chmod 600 ~/.pgpass
```

Остановил СУБД и очистил директорию с данными кластера:
```
root@otus-pgcloud2023-hw5-backup:~# systemctl stop postgresql

postgres@otus-pgcloud2023-hw5-backup:~$ rm -rf /var/lib/postgresql/15/main/* 
```

11. Проверил настройки кластеров PostgreSQL на возможность создания
резервной копии с реплики.

На резервном сервере:
```
postgres=# show hot_standby;
 hot_standby 
-------------
 on
(1 row)
```

На ведущем:
```
postgres=# show full_page_writes ;
 full_page_writes 
------------------
 on
(1 row)
```

Сделал полную резервную копию кластера с ведомого сервера в два параллельных потока:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b FULL -j 2 --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVE2UP, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Backup RVE2UP is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVE2UP/database/pg_wal/000000010000000000000003 to be streamed
INFO: PGDATA size: 37MB
INFO: Current Start LSN: 0/34211F8, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/34212E0
WARNING: Could not read WAL record at 0/34212E0: invalid record length at 0/34212E0: wanted 24, got 0
INFO: Wait for LSN 0/34212E0 in streamed WAL segment /home/pgbackups/backups/hw5/RVE2UP/database/pg_wal/000000010000000000000003
WARNING: Could not read WAL record at 0/34212E0: invalid record length at 0/34212E0: wanted 24, got 0
[n line stripped]
WARNING: Could not read WAL record at 0/34212E0: invalid record length at 0/34212E0: wanted 24, got 0
INFO: Getting the Recovery Time from WAL
INFO: Failed to find Recovery Time in WAL, forced to trust current_timestamp
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 1s
INFO: Validating backup RVE2UP
INFO: Backup RVE2UP data files are valid
INFO: Backup RVE2UP resident size: 28MB
INFO: Backup RVE2UP completed
```

Проверил наличие копии в каталоге резервных копий:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 show -B /home/pgbackups --instance hw5
================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI  Time  Data   WAL  Zratio  Start LSN  Stop LSN   Status 
================================================================================================================================
 hw5       15       RVE2UP  2023-05-28 22:08:52+00  FULL  STREAM    1/0   14s  12MB  16MB    2.94  0/34211F8  0/34212E0  OK     
```

Восстановил кластер PostgreSQL локально на сервере резервных копий:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 restore -B /home/pgbackups --instance hw5 -j 2 -D /var/lib/postgresql/15/main -i RVE2UP --remote-proto=none
INFO: Validating backup RVE2UP
INFO: Backup RVE2UP data files are valid
INFO: Backup RVE2UP WAL segments are valid
INFO: Backup RVE2UP is valid.
INFO: Restoring the database from backup at 2023-05-28 22:08:49+00
INFO: Start restoring backup files. PGDATA size: 53MB
INFO: Backup files are restored. Transfered bytes: 53MB, time elapsed: 0
INFO: Restore incremental ratio (less is better): 100% (53MB/53MB)
INFO: Syncing restored files to disk
INFO: Restored backup files are synced, time elapsed: 3s
INFO: Restore of backup RVE2UP completed.
```

Запустил сервер PostgreSQL:
```
root@otus-pgcloud2023-hw5-backup:~# systemctl start postgresql
```

Проверил наличие данных на восстановленном из резервной копии кластере:
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

Резервное копирование и восстановление из резервной копии кластера СУБД
PostgreSQL работает!


12. Проверяю работу резервного копирования на нагруженном кластере
PostgreSQL.

Подготовил на ведущем сервере БД и таблицы для программы тестирования
производительности `pgbench`:
```
postgres=# create database pgbench_test;
CREATE DATABASE

postgres@otus-pgcloud2023-hw5-primary:~$ pgbench -i -s 100 pgbench_test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 43.53 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 56.79 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 43.62 s, vacuum 1.62 s, primary keys 11.55 s).
```

Запустил тест производительности:
```
postgres@otus-pgcloud2023-hw5-primary:~$ pgbench -c 90 -T 300 pgbench_test
```

И параллельно сделал полную резервную копию с ведомого сервера:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b FULL -j 2 --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVE3WF, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Backup RVE3WF is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVE3WF/database/pg_wal/000000010000000000000045 to be streamed
INFO: PGDATA size: 1542MB
INFO: Current Start LSN: 0/45FFFAA8, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 31s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/65E56D30
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 1s
INFO: Validating backup RVE3WF
INFO: Backup RVE3WF data files are valid
INFO: Backup RVE3WF resident size: 693MB
INFO: Backup RVE3WF completed
```

Проверил наличие копии в каталоге резервных копий:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 show -B /home/pgbackups --instance hw5
====================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI  Time   Data    WAL  Zratio  Start LSN   Stop LSN    Status 
====================================================================================================================================
 hw5       15       RVE3WF  2023-05-28 22:32:00+00  FULL  STREAM    1/0   35s  149MB  544MB   10.36  0/45FFFAA8  0/65E56D30  OK     
 hw5       15       RVE2UP  2023-05-28 22:08:52+00  FULL  STREAM    1/0   14s   12MB   16MB    2.94  0/34211F8   0/34212E0   OK     
```

Запустил тест производительности ещё раз:
```
postgres@otus-pgcloud2023-hw5-primary:~$ pgbench -c 90 -T 300 pgbench_test
```

И параллельно сделал полную резервную копию теперь с ведущего сервера:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b FULL -j 2 --stream --temp-slot --remote-host=otus-pgcloud2023-hw5-primary
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVE48H, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
The authenticity of host 'otus-pgcloud2023-hw5-primary (10.182.0.22)' can't be established.
ECDSA key fingerprint is SHA256:+vPN4uiD5rb0CXEK7nRL57MB4tcJ+/5WUywjDweVfPA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVE48H/database/pg_wal/0000000100000000000000A5 to be streamed
INFO: PGDATA size: 1557MB
INFO: Current Start LSN: 0/A5F27A80, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 3m:44s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/DC41CFC8
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 1s
INFO: Validating backup RVE48H
INFO: Backup RVE48H data files are valid
INFO: Backup RVE48H resident size: 1054MB
INFO: Backup RVE48H completed
```

Проверил наличие новой копии в каталоге резервных копий:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 show -B /home/pgbackups --instance hw5
======================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI    Time   Data    WAL  Zratio  Start LSN   Stop LSN    Status 
======================================================================================================================================
 hw5       15       RVE48H  2023-05-28 22:43:54+00  FULL  STREAM    1/0  5m:21s  158MB  896MB    9.87  0/A5F27A80  0/DC41CFC8  OK     
 hw5       15       RVE3WF  2023-05-28 22:32:00+00  FULL  STREAM    1/0     35s  149MB  544MB   10.36  0/45FFFAA8  0/65E56D30  OK     
 hw5       15       RVE2UP  2023-05-28 22:08:52+00  FULL  STREAM    1/0     14s   12MB   16MB    2.94  0/34211F8   0/34212E0   OK     
```

Резервное копирование кластера СУБД PostgreSQL под нагрузкой работает!


---
