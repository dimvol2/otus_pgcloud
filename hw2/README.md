##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #2 ("Postgres & Docker")

1. Создал и запустил VM в GCE otus-pgcloud2023-medium-ubuntu-2004 (Ubuntu 20.04 LTS).  

2. Вошёл в VM, обновил описания пакетов и установил в систему пакет docker:
```
apt update
apt install docker.io
```
Убедился, что docker-демон работает:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# systemctl status docker | grep Active
     Active: active (running) since Sun 2023-05-07 23:25:03 UTC; 6min ago
```

3. Создал директорию `/var/lib/postgres`.

Запустил контейнер с сервером PostgreSQL 14.7 со смонтированным в эту локальную
директорию каталогом кластера:
```
docker run -d --name pg14.7 \
  -e POSTGRES_PASSWORD=supersecretpass \
  -e POSTGRES_DB=otus \
  -e POSTGRES_USER=otus \
  -v /var/lib/postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:14.7 \
  -c ssl=on \
  -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem \
  -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
Unable to find image 'postgres:14.7' locally
14.7: Pulling from library/postgres
9e3ea8720c6d: Pull complete 
7782b3e1be4b: Pull complete 
247ec4ff783a: Pull complete 
f7ead6900700: Pull complete 
e7afdbe9a191: Pull complete 
3ef71fe7cece: Pull complete 
1459ebb56be5: Pull complete 
3595124f6861: Pull complete 
b4486f48017f: Pull complete 
a63c7ed11505: Pull complete 
0146526465ec: Pull complete 
039e22ce4453: Pull complete 
58942c7b9396: Pull complete 
Digest: sha256:5ac16ee311340b09e3670d660c76f77a611202fd07b05d486e934eece99bea7c
Status: Downloaded newer image for postgres:14.7
9aa93d6e85e7215342a0103901106974250f2bdc32e69429c90eb8eb3324fc9a
```
Запустил контейнер с клиентом:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker run -d --name pg14.7_client -e POSTGRES_PASSWORD=pass postgres:14.7
e5e8fc683ddade839db4d8a7ed30dce728c927f4d8c88c5f94e8049ade386b12
```
Убедился, что контейнеры работают:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker ps 
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS          PORTS                                       NAMES
e5e8fc683dda   postgres:14.7   "docker-entrypoint.s…"   17 seconds ago   Up 16 seconds   5432/tcp                                    pg14.7_client
9aa93d6e85e7   postgres:14.7   "docker-entrypoint.s…"   52 seconds ago   Up 49 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg14.7
```
Создал сеть:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker network create pg_net
bb4565fea5c4d45d00c6d3e7c1220c3810478ba7665f5ccaeabffcf84da544b1
```

Подключил контейнеры к сети:
```
docker network connect pg_net pg14.7
docker network connect pg_net pg14.7_client
```

4. Вошёл в контейнер, используемый в качестве клиентского:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker exec -it pg14.7_client bash
root@e5e8fc683dda:/# 
```

Соединился с сервером в другом контейнере, используя заданный при создании
контейнера с сервером пароль:
```
root@e5e8fc683dda:/# psql -h pg14.7 -U otus
Password for user otus: 
psql (14.7 (Debian 14.7-1.pgdg110+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

otus=# 
```

Создал таблицу и вставил в неё две строки:
```
otus=# create table otus_hw2 (i int, s text);
CREATE TABLE
otus=# insert into otus_hw2 values (3, 'three'), (11, 'eleven');
INSERT 0 2
```

Добавил правило в firewall GCE для пропуска трафика на стандартный порт сервера PostgreSQL:
```
gcloud compute firewall-rules create --allow=tcp:5432 --description="Allow incoming traffic to postgres" --direction=INGRESS
```
Подключился к БД извне GCP и проверил наличие данных в таблице:
```
bash-5.1$ psql -h 34.125.236.54 -U otus
Password for user otus: 
psql (14.7)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

otus=# select * from otus_hw2; 
 i  |   s    
----+--------
  3 | three
 11 | eleven
(2 rows)

otus=# 
```

5. Удалил контейнер с сервером:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker rm -f pg14.7
pg14.7
```
Создал контейнер с сервером заново, сразу с подключением к сети `pg_net`:
```
root@otus-pgcloud2023-medium-ubuntu-2004:~# docker run -d --name pg14.7 \
  --network=pg_net \
  -e POSTGRES_PASSWORD=supersecretpass \
  -e POSTGRES_DB=otus \
  -e POSTGRES_USER=otus \
  -v /var/lib/postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:14.7 \
  -c ssl=on \
  -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem \
  -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ebffae456043b1e9e6f48071f800e664f6a82585104d40ba461a2b9508325d91
```

Подключился из клиентского контейнера, убедился, что данные присутствуют:
```
root@e5e8fc683dda:/# psql -h pg14.7 -U otus
Password for user otus: 
psql (14.7 (Debian 14.7-1.pgdg110+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

otus=# select * from otus_hw2;
 i  |   s    
----+--------
  3 | three
 11 | eleven
(2 rows)

otus=# 
```


---
