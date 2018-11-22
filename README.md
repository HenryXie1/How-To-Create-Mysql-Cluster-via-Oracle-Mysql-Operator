## How To Create Mysql Cluster via Oracle Mysql-Operator

### Preparation
* Install helm. Please refer official [github tutorial](https://github.com/oracle/mysql-operator/blob/master/docs/tutorial.md)
* $ kubectl create ns mysql-operator
* git clone the repository
```
$ git clone https://github.com/oracle/mysql-operator
```
* At the root of the local mysql-operator, use helm to install the operator
```
helm install --name mysql-operator mysql-operator
.....
==> v1beta1/CustomResourceDefinition
mysqlclusters.mysql.oracle.com         0s
mysqlbackupschedules.mysql.oracle.com  0s
mysqlbackups.mysql.oracle.com          0s
mysqlrestores.mysql.oracle.com         0s
NOTES:
Thanks for installing the MySQL Operator.
Check if the operator is running with
kubectl -n mysql-operator get po
```
* $kubectl logs < mysql-operator-podname >  , see some logs like
```
....
I1115 05:08:48.354994       1 cache.go:37] Caches are synced for mysql cluster controller
I1115 05:08:48.355004       1 controller.go:247] Starting Cluster controller workers
I1115 05:08:48.355018       1 controller.go:253] Started Cluster controller workers
I1115 05:08:48.355094       1 shared_informer.go:123] caches populated
I1115 05:08:48.355131       1 cache.go:37] Caches are synced for operator-backup-controller controller
I1115 05:08:48.355140       1 operator_controller.go:196] Caches are synced
I1115 05:08:48.355228       1 shared_informer.go:123] caches populated
I1115 05:08:48.355236       1 cache.go:37] Caches are synced for operator-restore-controller controller
I1115 05:08:48.355244       1 operator_controller.go:183] Caches are synced
```

### Create a mysql cluster
* create role binding for mysql-agent. Yaml file is like

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql-agent
  namespace: default
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mysql-agent
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mysql-agent
subjects:
- kind: ServiceAccount
  name: mysql-agent
  namespace: default
```
* create root password for mysql Cluster
```
kubectl create secret generic mysql-root-user-secret --from-literal=password=****
```
* We create 2 local PVs for each mysql db (total 4 PVs). One is for data,one is for backup. Yaml file is like

```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-0
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/u01/build/mysqldata-0"
  storageClassName: data
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-1
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/u01/build/mysqldata-1"
  storageClassName: data
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: backup-data-0
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/u01/build/mysqlbackup-data-0"
  storageClassName: backupdata
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: backup-data-1
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/u01/build/mysqlbackup-data-1"
  storageClassName: backupdata
```

* create mysql cluster with 2 members (1 primary,1 secondary)

```
---
apiVersion: mysql.oracle.com/v1alpha1
kind: Cluster
metadata:
  name: mysql-cluster-localfs
spec:
  members: 2
  rootPasswordSecret:
    name: mysql-root-user-secret
  volumeClaimTemplate:
    metadata:
      name: data
    spec:
      storageClassName: data
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
  backupVolumeClaimTemplate:
    metadata:
      name: backup-data
    spec:
      storageClassName: backupdata
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
```          
* run mysql client to test the db is up
```
kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never    -- mysql -h mysql-cluster-localfs -uroot -p**** -e 'SELECT 1'
mysql: [Warning] Using a password on the command line interface can be insecure.
+---+
| 1 |
+---+
| 1 |
+---+
```

### Clean up
* $ helm del --purge mysql-operator
* $ helm ls --all mysql-operator  (verify it is purged)
