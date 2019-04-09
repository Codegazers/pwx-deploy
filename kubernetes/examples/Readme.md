## Create a PostgreSQL Portworx StorageClass ##

Create the filepx-postgres-sc.yaml with the following:
~~~
#### Portworx storage class
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
    name: px-postgres-sc
provisioner: kubernetes.io/portworx-volume
parameters:
   repl: "2"
~~~
Then run this command:
~~~
kubectl apply -f px-postgres-sc.yaml
~~~

## Create a PostgreSQL Portworx PersistentVolumeClaim ##
Create the file px-postgres-vol.yaml with the following contents:
~~~
#### Portworx persistent volume claim
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: postgres-data
   annotations:
     volume.beta.kubernetes.io/storage-class: px-postgres-sc
spec:
   accessModes:
     - ReadWriteOnce
   resources:
     requests:
       storage: 1Gi
~~~
Then run this command:

~~~
kubectl apply -f px-postgres-vol.yaml
~~~

## Deploy PostgreSQL using Kubernetes Deployment ##
We are recommending to use “Stork” for Postgres deployment as a scheduler. Stork is an opensource project that helps achieve even tighter integration of Portworx with Kubernetes. It allows users to co-locate pods with their data, provides seamless migration of pods in case of storage errors and makes it easier to create and restore snapshots of Portworx volumes. Stork consists of 2 components, the stork scheduler, and an extender. Both of these components run in HA mode with three replicas by default.

Note: You need to install the stork before deploying below Postgres spec file. It px-postgres-app.yaml uses schedulerName: stork instead of schedulerName: default-scheduler.

### Deploying PostgreSQL on Kubernetes have following prerequisites ###

We need to Define the Environment Variables for PostgreSQL

POSTGRES_USER (Super Username for PostgreSQL)

POSTGRES_PASSWORD (Super User password for PostgreSQL)

PGDATA (Data Directory for PostgreSQL Database)
~~~
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: postgres
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      schedulerName: stork
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: px/enabled
                operator: NotIn
                values:
                - "false"
      containers:
      - name: postgres
        image: postgres:9.5
        imagePullPolicy: "IfNotPresent"
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: pgbench
        - name: POSTGRES_PASSWORD
          value: superpostgres
        - name: PGBENCH_PASSWORD
          value: superpostgres
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgredb
      volumes:
      - name: postgredb
        persistentVolumeClaim:
          claimName: postgres-data
~~~
Now let’s deploy PostgreSQL using following commands:

~~~
kubectl apply -f px-postgres-app.yaml

~~~
Validate StorageClass, PersistentVolumeClaim and PostgreSQL Deployment
~~~
kubectl get sc
NAME             PROVISIONER                     AGE
px-postgres-sc   kubernetes.io/portworx-volume   1h

kubectl get pvc
NAME            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
postgres-data   Bound     pvc-60e43292-06e3-11e8-96b4-022185d04910   1Gi        RWO            px-postgres-sc   1h

kubectl get pod
NAME                        READY     STATUS    RESTARTS   AGE
postgres-86cb8587c4-l9r48   1/1       Running   0          1h

~~~
Validate PostgreSQL DB Server
To access via docker exec:

~~~
kubectl get pod
NAME                              READY     STATUS    RESTARTS   AGE
postgres-5d8767bb94-wdp2d         1/1       Running   0          1m

kubectl exec -it postgres-5d8767bb94-wdp2d bash
root@postgres-86cb8587c4-l9r48:/#
root@postgres-86cb8587c4-l9r48:/# psql
psql (9.5.10)
Type "help" for help.

pgbench=# \d
              List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+---------
 public | pgbench_accounts | table | pgbench
 public | pgbench_branches | table | pgbench
 public | pgbench_history  | table | pgbench
 public | pgbench_tellers  | table | pgbench
(4 rows)

pgbench=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 pgbench   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)

pgbench=# \q
~~~


Launch pgbench to create some tables

~~~

root@postgres-86cb8587c4-l9r48:/# PGUSER=pgbench pgbench -i -s 10 pgbench

~~~
