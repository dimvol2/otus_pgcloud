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


---
