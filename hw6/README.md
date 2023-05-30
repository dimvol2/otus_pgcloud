##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #6 "Кластер Patroni" (Занятия "Кластер Patroni on-premise" 1 и 2)

1. Создал и запустил три VM в GCE для кластера etcd:
```
for i in {1..3};
  do gcloud compute instances create etcd$i --zone=us-west4-b \
    --machine-type=e2-medium \
    --image-project=ubuntu-os-cloud --image=ubuntu-minimal-2004-focal-v20230530a &\
  done;  
```

2. Установил на три VM etcd:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && \
    sudo apt -y install etcd' & \
  done;
```

Остановил сервисы:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo systemctl stop etcd' & \
  done;
```

Добавил конфиги на все три ноды:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='cat > temp.cfg << EOF 
ETCD_NAME="$(hostname)"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://$(hostname):2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://$(hostname):2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
EOF
     cat temp.cfg | sudo tee -a /etc/default/etcd' & \
  done;
```

Запустил сервисы:
```
for i in {1..3};
  do gcloud compute ssh etcd$i \
    --command='sudo systemctl start etcd' & \
  done;
```


---
