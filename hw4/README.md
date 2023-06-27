##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #4 "Тюнинг Постгреса" (Занятие "Углубленный анализ производительности. Профилирование. Мониторинг. Оптимизация")

1. Создал и запустил VM в GCE otus-pgcloud2023-hw4 (Ubuntu 20.04 LTS):
```
gcloud compute instances create otus-pgcloud2023-hw4 --zone=us-west4-b \
--machine-type=e2-medium \
--image-project=ubuntu-os-cloud --image=ubuntu-minimal-2004-focal-v20230518
```

Вошёл на VM:
```
gcloud compute ssh otus-pgcloud2023-hw4
```
из под пользователя `root` добавил репозиторий PostgreSQL, обновил базу
пакетов и установил пакет с PostgreSQL 15-й версии:
```
echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
apt update
apt install -y postgresql-15
```

Убедился, что кластер PostgreSQL запущен:
```
postgres@otus-pgcloud2023-hw4:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

2. Создал и инициализировал БД для тестов:
```
postgres=# create database pgbench_test ;
CREATE DATABASE
postgres@otus-pgcloud2023-hw4:~$ pgbench -i -s 100 pgbench_test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 28.12 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 38.83 s (drop tables 0.03 s, create tables 0.02 s, client-side generate 28.23 s, vacuum 0.28 s, primary keys 10.28 s).
```

Проверил значение параметров `shared_buffers` и `work_mem` по умолчанию:
```
postgres=# show shared_buffers;
 shared_buffers 
----------------
 128MB
(1 row)
```
postgres=# show work_mem ;
 work_mem 
----------
 4MB
(1 row)

Запустил тест `pgbench` трижды:
```
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 538.181562 (without initial connection time)
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 595.535230 (without initial connection time)
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 563.728169 (without initial connection time)
```

Среднее значение 565 транзакций в секунду.

3. Увеличил значение параметров до рекомендованных для
тестовой VM (2CPU, 4GB RAM) [pgconfigurator](https://pgconfigurator.cybertec.at/),
рестартовал кластер для применения настройки, проверил новые значения:
```
postgres=# alter system set shared_buffers ='1024MB';
ALTER SYSTEM

postgres=# alter system set work_mem ='32MB';
ALTER SYSTEM

root@otus-pgcloud2023-hw4:~# pg_ctlcluster 15 main restart

postgres=# show shared_buffers;
 shared_buffers 
----------------
 1GB
(1 row)

postgres=# show work_mem;
 work_mem 
----------
 32MB
(1 row)
```

Прогнал тесты повторно:
```
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 540.972958 (without initial connection time)
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 646.145052 (without initial connection time)
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 778.322371 (without initial connection time)
```

Среднее значение 654 транзакций в секунду, повышение около 15%

4. Для дальнейшего увеличения производительности, отключил параметры,
обеспечивающие надёжную запись данных на диск:

```
postgres=# alter system set synchronous_commit='off';
ALTER SYSTEM
postgres=# alter system set fsync='off';
ALTER SYSTEM
postgres=# alter system set full_page_writes='off';
ALTER SYSTEM
```
и перезагрузил кластер PostgreSQL для применения настроек:
```
root@otus-pgcloud2023-hw4:~# pg_ctlcluster 15 main restart
```

Провёл тестирование:

```
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 875.357943 (without initial connection time)
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 1127.124786 (without initial connection time)
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 1556.152374 (without initial connection time)
```

Среднее значение 1186 транзакций в секунду, примерно двухкратное повышение


---
