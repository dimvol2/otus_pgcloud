##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #10 "Multi master" (Занятие "Работа с горизонтально масштабируемым кластером")

- Создал 3 VM в GCE (2 CPU, 8GB RAM, 50GB SSD):
```
for i in {1..3};
  do gcloud compute instances create cc$i --zone=us-east4-a\
    --machine-type=e2-standard-2 \
    --boot-disk-size=20GB --boot-disk-type=pd-ssd \
    --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230715 &\
  done;
```

- Установил на них последнюю версию CockroachDB:
```
for i in {1..3};
  do gcloud compute ssh cc$i \
    --command='wget -qO- https://binaries.cockroachdb.com/cockroach-v23.1.5.linux-amd64.tgz | tar zxf - && \
      sudo cp -i cockroach-v23.1.5.linux-amd64/cockroach /usr/local/bin/ && \
      sudo mkdir -p /opt/cockroach && \
      sudo chown bbc.bbc /opt/cockroach' \
    --zone=us-east4-a & \
  done;
```

//TODO replace to openssl
- На локальной машине установил CockroachDB и сгенерировал сертификаты
```
wget -qO- https://binaries.cockroachdb.com/cockroach-v23.1.5.linux-amd64.tgz | tar zxf -
cd cockroach-v23.1.5.linux-amd64
mkdir certs my-safe-directory
./cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
./cockroach cert create-node cc1 cc2 cc3 --certs-dir=certs --ca-key=my-safe-directory/ca.key --overwrite
```

- Раскатал сертификаты на все ноды:
```
for i in {1..3};
  do scp -r certs cc$i \
    --command='... && \
      sudo ... && \
      sudo .. ' \
    --zone=us-east4-a & \
  done;
```

- Стартовал ноды:
```
cockroach start --certs-dir=certs --advertise-addr=cc$i \
 --join=cc1,cc2,cc3 --cache=.25 --max-sql-memory=.25 --background
```

- Инициализировал кластер
```
bbc@cc1:~$ cockroach init --certs-dir=certs --host=cc1
```

- Статус:
```
bbc@cc1:~$ cockroach node status --certs-dir=certs
```

- Установил ПО для бакетов

- Примонтировал chicago10

- Залил БД

- Запросы
- Выводы

---
