##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #12 "Parallel cluster" (Занятие "Массивно параллельные кластера PostgreSQL")

1. Создал 2 VM в GCE
```
for i in {1..2};
  do gcloud compute instances create gp$i --zone=us-east4-a \
    --machine-type=e2-standard-2 \
    --boot-disk-size=50GB --boot-disk-type=pd-ssd \
    --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230628 &\
  done;
```
Добавил пользователя `gpadmin` и создал пару ssh-ключей:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo groupadd gpadmin && \
    sudo useradd gpadmin -r -m -g gpadmin && \
    echo gpadmin:gpadmin123 | sudo chpasswd && \
    sudo usermod -aG sudo gpadmin && \
    sudo -u gpadmin ssh-keygen -t rsa -b 4096 -q -f /home/gpadmin/.ssh/id_rsa -N ""' \
    --zone=us-east4-a &\
  done;
```
Добавил репозиторий от Ubuntu версии 18.04
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='cat > repo.list << EOF 
deb http://ppa.launchpad.net/greenplum/db/ubuntu bionic main
deb http://ru.archive.ubuntu.com/ubuntu bionic main
EOF
    cat repo.list | sudo tee -a /etc/apt/sources.list.d/greenplum-ubuntu-db-bionic.list' \
    --zone=us-east4-a & \
  done;
```
Закрепил этот репозиторий в качестве источника пакетов для Greenplum:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='cat > repo.pin << EOF 
Package: greenplum*
Pin: release v=18.04
Pin-Priority: 1
EOF
    cat repo.pin | sudo tee -a /etc/apt/preferences.d/99-greenplum' \
    --zone=us-east4-a & \
  done;
```
Добавил gpg-ключ Ubuntu:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='gpg --keyserver keyserver.ubuntu.com --recv 3C6FDC0C01D86213 && \
    gpg --export --armor 3C6FDC0C01D86213 | sudo apt-key add -' \
    --zone=us-east4-a &\
  done;
```
Обновил список пакетов, сами пакеты, установил sshpass и Greenplum:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo apt update && \
    sudo apt upgrade -y -q && \
    sudo apt -y install sshpass greenplum-db-6' \
    --zone=us-east4-a &\
  done;
```
Сменил владельца каталога с установленным Greenplun на `gpadmin`:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo chown -R gpadmin.gpadmin /opt/greenplum* && \
    sudo chgrp -R gpadmin /opt/greenplum*' \
    --zone=us-east4-a &\
  done;
```
Добавил инициализацию переменных окружения, необходимых для работы
Greenplum:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo chsh -s /bin/bash gpadmin && \
    cat > bashrc << EOF
source /opt/greenplum-db-$(dpkg -s greenplum-db-6 | grep -i "^Version" | cut -d" " -f2 | cut -d"-" -f1)/greenplum_path.sh
EOF
    cat bashrc | sudo tee -a ~gpadmin/.bashrc' \
    --zone=us-east4-a &\
  done;
```
Включил возможность входа по ssh с использованием пароля:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo sed -i s@"PasswordAuthentication no"@"PasswordAuthentication yes"@ /etc/ssh/sshd_config && \
    sudo systemctl restart ssh' \
    --zone=us-east4-a &\
  done;
```
Раскидал ssh-ключи по виртуальным машинам:
```
for i in {1..2};
  do gcloud compute ssh gp$i \
    --command='sudo su -c "for j in {1..2}; do SSHPASS=gpadmin123 sshpass -v -e ssh-copy-id -o StrictHostKeyChecking=no gp$j & done" gpadmin' \
    --zone=us-east4-a &\
  done;
```

---
