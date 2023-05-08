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
docker run -d --name pg14.7 -e POSTGRES_PASSWORD=supersecpasssword -v/var/lib/postgres:/var/lib/postgresql/data -p5432:5432 postgres:14.7


---
