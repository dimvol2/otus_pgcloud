##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #7 "Постгрес в minikube" (Занятие "Введение в Kubernetes")

1. Создал и запустил VM в GCE (Ubuntu 20.04 LTS):
```
gcloud compute instances create hw7 --machine-type=e2-medium \
--image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230605 --zone=us-west4-b
```

2. Вошёл на VM:
```
gcloud compute ssh hw7 --zone=us-west4-b
```

Обновил базу пакетов, установил докер:
```
root@hw7:~# apt update && apt install docker.io -y
sudo usermod -aG docker $USER && newgrp docker
```

Установил клиент PostgreSQL:
```
root@hw7:~# apt -y install postgresql-client-common postgresql-client
```

Установил последние версии `minikube` и `kubectl`:
```
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
install kubectl  /usr/local/bin/
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Запустил `minikube`:
```
minikube start
```

3. Создал namespace и прописал его по умолчанию:
```
bbc@hw7:~$ kubectl create namespace kub-hw7
namespace/kub-hw7 created

kubectl config set-context --current --namespace=kub-hw7
```

Настроил переменные окружения для работы с контейнерами в `minikube`:
```
bbc@hw7:~$ eval $(minikube -p minikube docker-env)
```

С помощью манифеста:
```
cat >>postgres.yaml<<EOF
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
   - port: 5432
  selector:
    app: postgres

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14.8
        ports:
        - containerPort: 5432
          name: postgredb
        env:
          - name: POSTGRES_DB
            value: myapp
          - name: POSTGRES_USER
            value: myuser
          - name: POSTGRES_PASSWORD
            value: passwd
        volumeMounts:
        - name: postgredb
          mountPath: /var/lib/postgresql/data
          subPath: postgres
  volumeClaimTemplates:
  - metadata:
      name: postgredb
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
EOF
```

Развернул PostgreSQL:
```
bbc@hw7:~$ kubectl apply -f postgres.yaml
```

```
bbc@hw7:~$ minikube service postgres -n kub-hw7 --url
http://192.168.49.2:31068
```

Соединился, проверил версию:
```
bbc@hw7:~$ psql -h 192.168.49.2 -p 31068 -d myapp -U myuser 
Password for user myuser: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 14.8 (Debian 14.8-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

myapp=# select version();
                                                           version                                                           
-----------------------------------------------------------------------------------------------------------------------------
 PostgreSQL 14.8 (Debian 14.8-1.pgdg110+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
(1 row)
```

Создал БД и таблицу в ней, занёс тестовую запись:
```
myapp=# create database hw7;
CREATE DATABASE
myapp=# \c hw7
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 14.8 (Debian 14.8-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
You are now connected to database "hw7" as user "myuser".
hw7=# create table t(s text);
CREATE TABLE
hw7=# insert into t values ('pg with kubernetes manifest');
INSERT 0 1
```

Удалил:
```
kubectl delete -f postgres.yaml

Развернул PostgreSQL через манифест заново:
```
bbc@hw7:~$ kubectl apply -f postgres.yaml
service/postgres created
statefulset.apps/postgres-statefulset created
```

```
bbc@hw7:~$ minikube service postgres -n kub-hw7 --url
http://192.168.49.2:30075
```

Соединился, проверил наличие данных в таблице:
```
bbc@hw7:~$ psql -h 192.168.49.2 -p 30075 -d myapp -U myuser 
Password for user myuser: 
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 14.8 (Debian 14.8-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

myapp=# \c hw7
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 14.8 (Debian 14.8-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
You are now connected to database "hw7" as user "myuser".
hw7=# select * from t;
              s              
-----------------------------
 pg with kubernetes manifest
(1 row)
```
