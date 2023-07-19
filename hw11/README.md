##### otus_pgcloud
# Курс `PostgreSQL Cloud Solutions`
### ДЗ #11 "Работа c PostgreSQL в Kubernetes" (Занятие "PostgreSQL и Google Kubernetes Engine")

- Сконфигурировал и запустил k8s кластер на платформе GKE:
```
gcloud beta container --project "disco-ascent-385720" clusters create "hw11" --zone "us-central1-f" --no-enable-basic-auth --cluster-version "1.26.5-gke.1200" --release-channel "regular" --machine-type "e2-standard-2" --image-type "COS_CONTAINERD" --disk-type "pd-ssd" --disk-size "30" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/disco-ascent-385720/global/networks/default" --subnetwork "projects/disco-ascent-385720/regions/us-central1/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --security-posture=standard --workload-vulnerability-scanning=disabled --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --no-enable-managed-prometheus --enable-shielded-nodes --node-locations "us-central1-f"
```
- Проверил статус кластера:
```
bash-5.1$ gcloud container clusters list 
NAME  LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE   NODE_VERSION     NUM_NODES  STATUS
hw11  us-central1-f  1.26.5-gke.1200  35.223.104.24  e2-standard-2  1.26.5-gke.1200  3          RUNNING
``` 
и наличие трёх нод:
```
bash-5.1$ kubectl get nodes
NAME                                  STATUS   ROLES    AGE     VERSION
gke-hw11-default-pool-993396e2-d07v   Ready    <none>   3m21s   v1.26.5-gke.1200
gke-hw11-default-pool-993396e2-hxxg   Ready    <none>   3m21s   v1.26.5-gke.1200
gke-hw11-default-pool-993396e2-jrlj   Ready    <none>   3m19s   v1.26.5-gke.1200
```

- Используя конфигурацию

secrets.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: citus-secrets
type: Opaque
data:
  password: c3Ryb25nX3Bhc3M=
```

master.yaml
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: citus-master-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: citus-master
  labels:
    app: citus-master
spec:
  selector:
    app: citus-master
  type: NodePort
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-master
spec:
  selector:
    matchLabels:
      app: citus-master
  replicas: 1
  template:
    metadata:
      labels:
        app: citus-master
    spec:
      containers:
      - name: citus
        image: citusdata/citus:8.0.0
        ports:
        - containerPort: 5432
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        volumeMounts:
        - name: storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - ./pg_healthcheck
          initialDelaySeconds: 60
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: citus-master-pvc
```
запустил мастер-ноду на версии CitusDB 8.0.0 (чтобы далее собирать кластер по
hostname)
```
bash-5.1$ kubectl create -f secrets.yaml -f master.yaml 
secret/citus-secrets created
persistentvolumeclaim/citus-master-pvc created
service/citus-master created
deployment.apps/citus-master created
```

- Используя конфиг worker-нод с версией CitusDB 11.3

workers.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: citus-workers
  labels:
    app: citus-workers
spec:
  selector:
    app: citus-workers
  clusterIP: None
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-worker
spec:
  selector:
    matchLabels:
      app: citus-workers
  serviceName: citus-workers
  replicas: 3
  template:
    metadata:
      labels:
        app: citus-workers
    spec:
      containers:
      - name: citus-worker
        image: citusdata/citus:11.3-pg14
        lifecycle:
          postStart:
            exec:
              command: 
              - /bin/sh
              - -c
              - if [ ${POD_IP} ]; then psql --host=citus-master --username=postgres --command="SELECT * from master_add_node('${HOSTNAME}.citus-workers', 5432);" ; fi
        ports:
        - containerPort: 5432
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - ./pg_healthcheck
          initialDelaySeconds: 60
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      storageClassName: premium-rwo
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

запустил три worker-ноды:
```
bash-5.1$ kubectl create -f workers.yaml 
service/citus-workers created
statefulset.apps/citus-worker created
```

- Проверил наличие нод:
```
bash-5.1$ kubectl get pods 
NAME                            READY   STATUS    RESTARTS   AGE
citus-master-5bff7bc99c-dk8tj   1/1     Running   0          5m27s
citus-worker-0                  1/1     Running   0          86s
citus-worker-1                  1/1     Running   0          61s
citus-worker-2                  1/1     Running   0          38s
```

- Зашёл в контейнер на master-ноду:
```
bash-5.1$ kubectl exec -ti citus-master-5bff7bc99c-dk8tj bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@citus-master-5bff7bc99c-dk8tj:/# 
```

- Проверил подключение worker-нод к базе:
```
root@citus-master-5bff7bc99c-dk8tj:/# psql -U postgres
psql (10.3 (Debian 10.3-1.pgdg90+1))
Type "help" for help.

postgres=# select * from master_get_active_worker_nodes();
          node_name           | node_port 
------------------------------+-----------
 citus-worker-2.citus-workers |      5432
 citus-worker-0.citus-workers |      5432
 citus-worker-1.citus-workers |      5432
(3 rows)
```

- Скопировал набор данныз для тестирования на диск:
```
curl -O https://storage.googleapis.com/chicago10/taxi.csv.0000000000[00-39]
```



---
