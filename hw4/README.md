##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #4 "Тюнинг Постгреса" (Занятие "Углубленный анализ производительности. Профилирование. Мониторинг. Оптимизация")

1. Создал и запустил VM в GCE otus-pgcloud2023-hw4 (Ubuntu 20.04 LTS):
```
gcloud compute instances create otus-pgcloud2023-hw4 --zone=us-west4-b \
--machine-type=e2-medium \
--image-project=ubuntu-os-cloud --image=ubuntu-minimal-2004-focal-v20230518
```

2. Вошёл на VM:
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

postgres@otus-pgcloud2023-hw4:~$ psql -c "create database pgbench_test;"
CREATE DATABASE

Создал и инициализировал БД для тестов:
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

Проверил значение параметра `shared_buffers` по умолчанию:
```
postgres=# show shared_buffers;
 shared_buffers 
----------------
 128MB
(1 row)
```

Запустил тест `pgbench` трижды:
```
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 506.272905 (without initial connection time)

postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 507.978894 (without initial connection time)

postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 543.344801 (without initial connection time)
```

Увеличил значение параметра `shared_buffers` до рекомендованного для
запущенной VM [pgconfigurator](https://pgconfigurator.cybertec.at/),
рестартовал кластер для применения настройки, проверил новое значение:
```
postgres=# alter system set shared_buffers ='1024MB';
ALTER SYSTEM

root@otus-pgcloud2023-hw4:~# pg_ctlcluster 15 main restart

postgres=# show shared_buffers;
 shared_buffers 
----------------
 1GB
(1 row)
```

Прогнал тест повторно:
```
postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 1684.630180 (without initial connection time)

postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 1639.102592 (without initial connection time)

postgres@otus-pgcloud2023-hw4:~$ pgbench -c 100 -j 10 -T 30 pgbench_test|grep tps
starting vacuum...end.
tps = 1484.412508 (without initial connection time)
```

Число выполняемых транзакций выросло примерно в три раза.

---
