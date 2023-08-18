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

- На локальной машине установил CockroachDB и сгенерировал сертификаты
```
wget -qO- https://binaries.cockroachdb.com/cockroach-v23.1.5.linux-amd64.tgz | tar zxf -
cd cockroach-v23.1.5.linux-amd64
mkdir certs my-safe-directory
./cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
./cockroach cert create-node localhost cc1 cc2 cc3 --certs-dir=certs --ca-key=my-safe-directory/ca.key --overwrite
./cockroach cert create-client root --certs-dir=certs --ca-key=my-safe-directory/ca.key
```

- Раскатал сертификаты на все ноды:
```
for i in {1..3};
  do gcloud compute scp ./certs cc$i: \
    --recurse \
    --zone=us-east4-a & \
  done;
```

- Настроил права доступа:
```
for i in {1..3};
  do gcloud compute ssh cc$i \
    --command='chmod 700 certs' \
    --zone=us-east4-a & \
  done;
```

- Стартовал ноды:
```
for i in {1..3};
  do gcloud compute ssh cc$i \
    --command='cockroach start --certs-dir=certs --advertise-addr=$HOSTNAME \
      --join=cc1,cc2,cc3 --cache=.25 --max-sql-memory=.25 --background' \
    --zone=us-east4-a & \
  done;
```

- Инициализировал кластер
```
bbc@cc1:~$ cockroach init --certs-dir=certs --host=cc1
Cluster successfully initialized
```

- Проверил статус:
```
bbc@cc1:~$ cockroach node status --certs-dir=certs
  id |  address  | sql_address |  build  |              started_at              |              updated_at              | locality | is_available | is_live
-----+-----------+-------------+---------+--------------------------------------+--------------------------------------+----------+--------------+----------
   1 | cc1:26257 | cc1:26257   | v23.1.5 | 2023-07-23 18:21:06.013899 +0000 UTC | 2023-07-23 18:21:09.047332 +0000 UTC |          | true         | true
   2 | cc3:26257 | cc3:26257   | v23.1.5 | 2023-07-23 18:21:07.316887 +0000 UTC | 2023-07-23 18:21:07.428046 +0000 UTC |          | true         | true
   3 | cc2:26257 | cc2:26257   | v23.1.5 | 2023-07-23 18:21:07.541426 +0000 UTC | 2023-07-23 18:21:07.63175 +0000 UTC  |          | true         | true
(3 rows)
```
- Зашёл встроенным sql-клиентом, создал таблицу для датасета чикагского такси:
```
bbc@cc2:~$ cockroach sql --certs-dir=certs
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v23.1.5 (x86_64-pc-linux-gnu, built 2023/07/01 01:33:00, go1.19.10) (same version as client)
# Cluster ID: 4ad7ab25-710c-4d63-a5aa-80bd3000c553
#
# Enter \? for a brief introduction.
#
root@localhost:26257/defaultdb> CREATE DATABASE taxi;                                                                                          
CREATE DATABASE

Time: 26ms total (execution 26ms / network 0ms)

root@localhost:26257/defaultdb> \c taxi                                                                                                        
using new connection URL: postgresql://root@localhost:26257/taxi?application_name=%24+cockroach+sql&connect_timeout=15&sslcert=certs%2Fclient.root.crt&sslkey=certs%2Fclient.root.key&sslmode=verify-full&sslrootcert=certs%2Fca.crt
root@localhost:26257/taxi>create table taxi_trips (                                                                                      
                              unique_key text,                                                                                               
                              taxi_id text,                                                                                                  
                              trip_start_timestamp TIMESTAMP,                                                                                
                              trip_end_timestamp TIMESTAMP,                                                                                  
                              trip_seconds bigint,                                                                                           
                              trip_miles numeric,                                                                                            
                              pickup_census_tract bigint,                                                                                    
                              dropoff_census_tract bigint,                                                                                   
                              pickup_community_area bigint,                                                                                  
                              dropoff_community_area bigint,                                                                                 
                              fare numeric,                                                                                                  
                              tips numeric,                                                                                                  
                              tolls numeric,                                                                                                 
                              extras numeric,                                                                                                
                              trip_total numeric,                                                                                            
                              payment_type text,                                                                                             
                              company text,                                                                                                  
                              pickup_latitude numeric,                                                                                       
                              pickup_longitude numeric,                                                                                      
                              pickup_location text,                                                                                          
                              dropoff_latitude numeric,                                                                                      
                              dropoff_longitude numeric,                                                                                     
                              dropoff_location text                                                                                          
                              );                                                                                                             
                                                                                                                                             
CREATE TABLE

Time: 23ms total (execution 23ms / network 0ms)
```

Залил 10GB данных из бакета GCP:
```
root@localhost:26257/taxi> import into taxi_trips                                                                                              
                         (unique_key,taxi_id,trip_start_timestamp,trip_end_timestamp,trip_seconds,trip_miles,pickup_census_tract,
                         dropoff_census_tract,pickup_community_area,dropoff_community_area,fare,tips,tolls,extras,trip_total,
                         payment_type,company,pickup_latitude,pickup_longitude,pickup_location,dropoff_latitude,dropoff_longitude,
                         dropoff_location)                      
                         CSV DATA ('gs://chicago10/taxi.csv.0000000000*?AUTH=implicit')                                                      
                         WITH DELIMITER = ',', SKIP = '1';                                                                                   

        job_id       |  status   | fraction_completed |   rows   | index_entries |   bytes
---------------------+-----------+--------------------+----------+---------------+-------------
  884971446587785219 | succeeded |                  1 | 26753683 |             0 | 9785973702
(1 row)

Time: 317.714s total (execution 317.713s / network 0.001s)
```

- Потестировал аналитический запрос:
```
root@localhost:26257/taxi> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c                     
                         FROM taxi_trips                                                                                                     
                         group by payment_type                                                                                               
                         order by 3;                                                                                                         
                                                                                                                                             
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Prepaid      |            0 |        6
  Way2ride     |           12 |       27
  Split        |           17 |      180
  Dispute      |            0 |     5596
  Pcard        |            2 |    13575
  No Charge    |            0 |    26294
  Mobile       |           16 |    61256
  Prcard       |            1 |    86053
  Unknown      |            0 |   103869
  Credit Card  |           17 |  9224956
  Cash         |            0 | 17231871
(11 rows)

Time: 27.895s total (execution 27.894s / network 0.001s)

root@localhost:26257/taxi> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c                     
                         FROM taxi_trips                                                                                                     
                         group by payment_type                                                                                               
                         order by 3;                                                                                                         
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Prepaid      |            0 |        6
  Way2ride     |           12 |       27
  Split        |           17 |      180
  Dispute      |            0 |     5596
  Pcard        |            2 |    13575
  No Charge    |            0 |    26294
  Mobile       |           16 |    61256
  Prcard       |            1 |    86053
  Unknown      |            0 |   103869
  Credit Card  |           17 |  9224956
  Cash         |            0 | 17231871
(11 rows)

Time: 23.706s total (execution 23.706s / network 0.001s)
```

Запрос выполняется ~25s. Это на треть быстрее, чем в ванильном PostgreSQL на одной ВМ с такими же характеристиками
(см. в [ДЗ#8](https://github.com/dimvol2/otus_pgcloud/tree/main/hw8))


---
