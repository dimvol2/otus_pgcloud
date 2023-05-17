##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #3 "Установка и настройка PostgteSQL в контейнере Docker" (Занятие "Настройка PostgreSQL")

1. Создал и запустил VM в GCE otus-pgcloud2023-medium-ubuntu-2004 (Ubuntu 20.04
LTS):
```bash
gcloud compute instances create otus-pgcloud2023-hw3 --zone=us-west4-b \
--machine-type=e2-medium \
--image-project=ubuntu-os-cloud --image=ubuntu-minimal-2004-focal-v20230427
```

2. Вошёл на VM:
```bash
gcloud compute ssh otus-pgcloud2023-hw3
```
из под пользователя `root` добавил репозиторий PostgreSQL, обновил базу
пакетов и установил пакет с PostgreSQL 15-й версии:
```bash
echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
apt update
apt install postgresql-15
```
Убедился, что кластер PostgreSQL запущен:
```
postgres@otus-pgcloud2023-hw3:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

```
postgres=# create table test(s text);
CREATE TABLE
postgres=# insert into test values ('twenty two');
INSERT 0 1
postgres=# 
```

```
postgres@otus-pgcloud2023-hw3:~$ pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
```

```
bash-5.1$ gcloud compute disks create hw3-additional-disk --size=10GB
WARNING: You have selected a disk size of under [200GB]. This may result in poor I/O performance. For more information, see: https://developers.google.com/compute/docs/disks#performance.
Created [https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west4-b/disks/hw3-additional-disk].
NAME                 ZONE        SIZE_GB  TYPE         STATUS
hw3-additional-disk  us-west4-b  10       pd-standard  READY

New disks are unformatted. You must format and mount a disk before it
can be used. You can find instructions on how to do this at:

https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting
```

```
bash-5.1$ gcloud compute instances attach-disk otus-pgcloud2023-hw3 --disk hw3-additional-disk
Updated [https://www.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west4-b/instances/otus-pgcloud2023-hw3].
```

```
root@otus-pgcloud2023-hw3:~# lsblk 
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0  55.6M  1 loop /snap/core18/2721
loop1     7:1    0  53.2M  1 loop /snap/snapd/18933
loop2     7:2    0 334.8M  1 loop /snap/google-cloud-cli/127
sda       8:0    0    10G  0 disk 
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part 
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk 
```

```
root@otus-pgcloud2023-hw3:~# parted /dev/sdb mklabel gpt
Information: You may need to update /etc/fstab.
root@otus-pgcloud2023-hw3:~# parted -a opt /dev/sdb mkpart primary xfs 0% 100%
Information: You may need to update /etc/fstab.
```
```
root@otus-pgcloud2023-hw3:~# mkfs.xfs -L pgpartition /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=655232 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=2620928, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

В `/etc/fstab` добавил
```
LABEL=pgpartition /mnt/pgdata   xfs     defaults        0 2
```
```
mkdir /mnt/pgdata
chown -R postgres.postgres /mnt/pgdata/
```
Перегрузил VM:
```
bash-5.1$ gcloud compute instances stop otus-pgcloud2023-hw3
Stopping instance(s) otus-pgcloud2023-hw3...done.                                                                                               
Updated [https://compute.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west4-b/instances/otus-pgcloud2023-hw3].
bash-5.1$ gcloud compute instances start otus-pgcloud2023-hw3
Starting instance(s) otus-pgcloud2023-hw3...done.                                                                                               
Updated [https://compute.googleapis.com/compute/v1/projects/disco-ascent-385720/zones/us-west4-b/instances/otus-pgcloud2023-hw3].
Instance internal IP is y.y.y.y
Instance external IP is x.x.x.x
```
Зашёл:
```
gcloud compute ssh otus-pgcloud2023-hw3
```
Убедился, что VM перезагружена и диск подмонтировался:
```
bbc@otus-pgcloud2023-hw3:~$ uptime
 22:33:43 up 1 min,  1 user,  load average: 0.26, 0.12, 0.04
bbc@otus-pgcloud2023-hw3:~$ mount |grep sdb1
/dev/sdb1 on /mnt/pgdata type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```
Перенёс
```
postgres@otus-pgcloud2023-hw3:~$ mv /var/lib/postgresql/15 /mnt/pgdata
```
Попробовал запусить, ан нет
```
postgres@otus-pgcloud2023-hw3:~$ pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
Поменял в конфигурационном файле `/etc/postgresql/15/main/postgresql.conf` директорию с данными на новую:
```
sed -i s@/var/lib/postgresql/@/mnt/pgdata/@ /etc/postgresql/15/main/postgresql.conf
```
Запустил и убедился в наличии данных:
```
postgres@otus-pgcloud2023-hw3:~$ pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Removed stale pid file.
postgres@otus-pgcloud2023-hw3:~$ psql -U postgres
psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
     s      
------------
 twenty two
(1 row)

```

Создал новую VM, перемонтировал диск

---
