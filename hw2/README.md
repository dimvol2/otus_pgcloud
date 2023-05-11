##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #2 ("Postgres & Docker")

1. Создал и запустил VM в GCE otus-pgcloud2023-medium-ubuntu-2004 (Ubuntu 20.04 LTS).  

2. Вошёл в VM, установил в систему пакеты docker и docker-compose (?? нужен
ли композитор):
```
apt update
apt install docker docker-compose
```
Убедился, что docker-демон работает:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# systemctl status docker | grep Active
     Active: active (running) since Sun 2023-05-07 23:25:03 UTC; 6min ago
```

3. Создал директорию `/var/lib/postgres`

4. Запустил контейнер с сервером PostgreSQL:
```
#docker run -d --name pg14.7 -e POSTGRES_PASSWORD=supersecpasssword -v/var/lib/postgres:/var/lib/postgresql/data -p5432:5432 postgres:14.8

docker run -d --name pg15 \
  -e POSTGRES_PASSWORD=nkikhdthnD6qf2btQ-kw \
  -e POSTGRES_DB=otus \
  -e POSTGRES_USER=otus \
  -v /var/lib/postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15\ 
  -c ssl=on \
  -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem \
  -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key

Запустил контейнер с клиентом:
```
```
Создал сеть:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker network create pg_net
3d953abc271485cfd151b179ec412bbadc0d5f140274f76f2b9c7609cfdb6897
```

Подключил контейнеры к сети:
```
docker network connect pg_net pg15
docker network connect pg_net pg15_client
```

Вошёл в контейнер с клиентом:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker exec -it pg15_client bash
```

Соединился с сервером в другом контейнере, используя заданный при создании
контейнера с сервером пароль:
```
root@e6237cf831c9:/# psql -h pg15 -U otus
Password for user otus: 
psql (15.2 (Debian 15.2-1.pgdg110+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

otus=# 
```

Создал таблицу и вставил в неё две строки:
```
otus=# create table otus_hw3 (i int, s text);
CREATE TABLE
otus=# insert into otus_hw3 values (3, 'three'), (11, 'eleven');
INSERT 0 2
```

```

Подключился к БД извне GCP и проверил наличие данных в таблице:
```
bash-5.1$ psql -h 34.125.158.66 -U otus
Password for user otus: 
psql (14.7, server 15.2 (Debian 15.2-1.pgdg110+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

otus=# select * from otus_hw3;
 i  |   s    
----+--------
  3 | three
 11 | eleven
(2 rows)
```

Удалил контейнер с сервером:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker rm -f pg15
pg15
```
Создал контейнер с сервером заново, сразу с подключением к сети `pg_net`:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker run -d --name pg15 \
>    --network=pg_net \
>    -e POSTGRES_PASSWORD=nkikhdthnD6qf2btQ-kw \
>    -e POSTGRES_DB=otus \
>    -e POSTGRES_USER=otus \
>    -v /var/lib/postgres:/var/lib/postgresql/data \
>    -p 5432:5432 \
>    postgres:15 \
>    -c ssl=on \
>    -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem \
>    -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
cacb2b43abd9244db0da88d98a746810be0ff4133ef5de472759684a80f452a7
```

Подключился из клиентского контейнера, убедился, что данные присуствуют:
```
root@e6237cf831c9:/# psql -h pg15 -U otus
Password for user otus: 
psql (15.2 (Debian 15.2-1.pgdg110+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

otus=# select * from otus_hw3;
 i  |   s    
----+--------
  3 | three
 11 | eleven
(2 rows)
```
---
