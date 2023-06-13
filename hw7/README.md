


curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
bash-5.1$ su -c "install minikube-linux-amd64 /usr/local/bin/minikube"

gcloud compute instances create hw7  --machine-type=e2-medium  --image-project=ubuntu-os-cloud --image=ubuntu-2004-focal-v20230605 --zone=us-west4-b

gcloud compute ssh hw7
root@hw7:~# apt update && apt install docker.io -y && apt install joe elinks -y
sudo usermod -aG docker $USER && newgrp docker

root@hw7:~# apt -y install postgresql-client-common postgresql-client


curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
install kubectl  /usr/local/bin/

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube



minikube start
minikube dashboard ?? howto forward local port outside (kubectl port-forward
??)


bbc@hw7:~$ kubectl create namespace kub-hw7
namespace/kub-hw7 created

kubectl config set-context --current --namespace=kub-hw7

bbc@hw7:~$ eval $(minikube -p minikube docker-env)


gcloud compute scp les7-8/les2/postgres/postgres.yaml hw7:

joe postgres.yaml (change version)


bbc@hw7:~$ minikube service postgres -n kub-hw7 --url
http://192.168.49.2:31068

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


myapp=# create database hw7;
CREATE DATABASE
myapp=# \c hw7
psql (12.15 (Ubuntu 12.15-0ubuntu0.20.04.1), server 14.8 (Debian 14.8-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
You are now connected to database "hw7" as user "myuser".
hw7=# create table t(s text);
CREATE TABLE
hw7=# insert into t values ("pg with kubernetes manifest");
ERROR:  column "pg with kubernetes manifest" does not exist
LINE 1: insert into t values ("pg with kubernetes manifest");
                              ^
hw7=# insert into t values ('pg with kubernetes manifest');
INSERT 0 1
hw7=# 

kubectl delete -f postgres.yaml
bbc@hw7:~$ kubectl apply -f postgres.yaml
service/postgres created
statefulset.apps/postgres-statefulset created

bbc@hw7:~$ minikube service postgres -n kub-hw7 --url
http://192.168.49.2:30075

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


root@hw7:~# snap install helm --classic
helm 3.10.1 from Snapcrafters✪ installed
