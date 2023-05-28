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
echo "hostssl replication replicanto 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
echo "hostssl replication backup 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
echo "hostssl backupdb backup 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf

Перечитал конфигурацию ведущего сервер СУБД для применения параметров конфигурации
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
postgres@otus-pgcloud2023-hw5-backup:~$ echo "otus-pgcloud2023-hw5-standby:5432:*:backup:strong_backup_password" > ~/.pgpass
postgres@otus-pgcloud2023-hw5-backup:~$ chmod 600 ~/.pgpass
```

Остановил СУБД и очистил директорию с данными кластера:
```
root@otus-pgcloud2023-hw5-backup:~# systemctl stop postgresql

postgres@otus-pgcloud2023-hw5-backup:~$ rm -rf /var/lib/postgresql/15/main/* 
```

11. Сделал полную копию:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b FULL -j 2 --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVCAIX, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
INFO: Backup RVCAIX is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVCAIX/database/pg_wal/00000001000000000000009A to be streamed
INFO: PGDATA size: 37MB
INFO: Current Start LSN: 0/9A84BD00, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/9A86BE98
WARNING: Could not read WAL record at 0/9A86BE98: invalid record length at 0/9A86BE98: wanted 24, got 0
INFO: Wait for LSN 0/9A86BE98 in streamed WAL segment /home/pgbackups/backups/hw5/RVCAIX/database/pg_wal/00000001000000000000009A
WARNING: Could not read WAL record at 0/9A86BE98: invalid record length at 0/9A86BE98: wanted 24, got 0
[n lines stripped]
WARNING: Could not read WAL record at 0/9A86BE98: invalid record length at 0/9A86BE98: wanted 24, got 0
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup RVCAIX
INFO: Backup RVCAIX data files are valid
INFO: Backup RVCAIX resident size: 28MB
INFO: Backup RVCAIX completed
```

Сделал инкрементальную копию в режиме `DELTA`:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b DELTA -j 2 --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVCAXP, backup mode: DELTA, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
INFO: Backup RVCAXP is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Parent backup: RVCAIX
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVCAXP/database/pg_wal/00000001000000000000009A to be streamed
INFO: PGDATA size: 37MB
INFO: Current Start LSN: 0/9A86BE98, TLI: 1
INFO: Parent Start LSN: 0/9A84BD00, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/9A86BF80
WARNING: Could not read WAL record at 0/9A86BF80: invalid record length at 0/9A86BF80: wanted 24, got 0
INFO: Wait for LSN 0/9A86BF80 in streamed WAL segment /home/pgbackups/backups/hw5/RVCAXP/database/pg_wal/00000001000000000000009A
WARNING: Could not read WAL record at 0/9A86BF80: invalid record length at 0/9A86BF80: wanted 24, got 0
[n lines stripped]
WARNING: Could not read WAL record at 0/9A86BF80: invalid record length at 0/9A86BF80: wanted 24, got 0
INFO: Getting the Recovery Time from WAL
INFO: Failed to find Recovery Time in WAL, forced to trust current_timestamp
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup RVCAXP
INFO: Backup RVCAXP data files are valid
INFO: Backup RVCAXP resident size: 16MB
INFO: Backup RVCAXP completed
```

Проверил наличие в каталоге резернвый копий:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 show -B /home/pgbackups 

BACKUP INSTANCE 'hw5'
===================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI   Time  Data   WAL  Zratio  Start LSN   Stop LSN    Status 
===================================================================================================================================
 hw5       15       RVCAXP  2023-05-27 23:08:16+00  DELTA  STREAM    1/1   11s  146kB  16MB    1.88  0/9A86BE98  0/9A86BF80  OK
 hw5       15       RVCAIX  2023-05-27 22:44:35+00  FULL   STREAM    1/0   31s   12MB  16MB    2.95  0/9A84BD00  0/9A86BE98  OK     
```

Восстановил кластер локально на сервере резервных копий:
```
postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 restore -B /home/pgbackups --instance hw5 -j 2 -D /var/lib/postgresql/15/main -i RVCAIX --remote-proto=none
INFO: Validating backup RVCAIX
INFO: Backup RVCAIX data files are valid
INFO: Backup RVCAIX WAL segments are valid
INFO: Backup RVCAIX is valid.
INFO: Restoring the database from backup at 2023-05-27 22:59:21+00
INFO: Start restoring backup files. PGDATA size: 53MB
INFO: Backup files are restored. Transfered bytes: 53MB, time elapsed: 0
INFO: Restore incremental ratio (less is better): 100% (53MB/53MB)
INFO: Syncing restored files to disk
INFO: Restored backup files are synced, time elapsed: 4s
INFO: Restore of backup RVCAIX completed.
```

Запустил СУБД:
```
root@otus-pgcloud2023-hw5-backup:~# systemctl start postgresql
```

Проверил наличие данных:
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

postgres=# create database pgbench_test;
CREATE DATABASE
postgres=# \q
postgres@otus-pgcloud2023-hw5-primary:~$ pgbench -i -s 100 pgbench_test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 34.68 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 50.24 s (drop tables 0.02 s, create tables 0.01 s, client-side generate 34.79 s, vacuum 3.35 s, primary keys 12.08 s).


postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b FULL -j 2 --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVCC04, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
INFO: Backup RVCC04 is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVCC04/database/pg_wal/0000000100000000000000DC to be streamed
INFO: PGDATA size: 1550MB
INFO: Current Start LSN: 0/DCFFEF00, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 29s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 1/16C20AF8
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 2s
INFO: Validating backup RVCC04
INFO: Backup RVCC04 data files are valid
INFO: Backup RVCC04 resident size: 1113MB
INFO: Backup RVCC04 completed


с ведущего:

postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 backup -B /home/pgbackups --instance hw5 -b FULL -j 2 --stream --temp-slot --remote-host=otus-pgcloud2023-hw5-primary
INFO: Backup start, pg_probackup version: 2.5.12, instance: hw5, backup ID: RVCCKX, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zlib, compress-level: 2
WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/pgbackups/backups/hw5/RVCCKX/database/pg_wal/000000010000000100000063 to be streamed
INFO: PGDATA size: 1558MB
INFO: Current Start LSN: 1/63082078, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 4m:35s
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 1/A58DA830
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 1s
INFO: Validating backup RVCCKX
INFO: Backup RVCCKX data files are valid
INFO: Backup RVCCKX resident size: 1231MB
INFO: Backup RVCCKX completed


postgres@otus-pgcloud2023-hw5-backup:~$ pg_probackup-15 show -B /home/pgbackups 

BACKUP INSTANCE 'hw5'
========================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI    Time   Data     WAL  Zratio  Start LSN   Stop LSN    Status 
========================================================================================================================================
 hw5       15       RVCCKX  2023-05-27 23:48:50+00  FULL   STREAM    1/0  5m:10s  159MB  1072MB    9.81  1/63082078  1/A58DA830  OK     
 hw5       ----     RVCCHB  ----                    FULL   STREAM    0/0       0      0       0    1.00  0/0         0/0         ERROR  
 hw5       15       RVCCEH  2023-05-27 23:40:30+00  FULL   STREAM    1/0     43s  161MB    16MB    9.73  1/4D000028  1/4D003C98  OK     
 hw5       15       RVCCB9  ----                    FULL   STREAM    0/0      1s      0       0    1.00  0/0         0/0         ERROR  
 hw5       15       RVCC04  2023-05-27 23:31:52+00  FULL   STREAM    1/0     40s  153MB   960MB   10.16  0/DCFFEF00  1/16C20AF8  OK     
 hw5       15       RVCAXP  2023-05-27 23:08:16+00  DELTA  STREAM    1/1     11s  146kB    16MB    1.88  0/9A86BE98  0/9A86BF80  OK     
 hw5       15       RVCAIX  2023-05-27 22:44:35+00  FULL   STREAM    1/0     31s   12MB    16MB    2.95  0/9A84BD00  0/9A86BE98  OK     


---
