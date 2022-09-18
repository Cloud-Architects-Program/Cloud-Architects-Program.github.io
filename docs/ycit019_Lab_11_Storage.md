# Kubernetes Storage Concepts

**Objective:**

  * Create a PersistentVolume (PV) referencing a disk in your environment.
  * Learn how to `dynamically provision` volumes.
  * Create a Single MySQL `Deployment` based on the `Volume Claim`
  * Deploy a Replicated MySQL (Master/Slaves) with a `StatefulSet` controller.

## 0 Create Regional GKE Cluster
**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:

```
gcloud container clusters create k8s-storage \
--region us-central1 \
--enable-network-policy \
--num-nodes 2 \
--machine-type "e2-standard-2" \
 --node-locations "us-central1-b","us-central1-c"
```

!!! note
    We created a Regional cluster with Nodes deployed in  "us-central1-b" and "us-central1-c" zones.

**Output:**
```
NAME          LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
k8s-storage  us-central1-c  1.19.9-gke.1400  34.121.222.83  e2-medium     1.19.9-gke.1400  2          RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-storage --region us-central1 --project jfrog2021
``` 



##  1 Dynamically Provision Volume

Our Lab already has provisioned Default Storageclass created by Cluster Administrator.

**Step 1** Verify what storage class is used in our lab:

```
kubectl get sc
```

**Output:**

```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
premium-rwo          pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   5m2s
standard (default)   kubernetes.io/gce-pd    Delete          Immediate              true                   5m2s
standard-rwo         pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   5m2s
```


!!! info
    The **PROVISIONER** field determines what volume plugin is used for provisioning PVs.
    
    * `standard` provisions  `standard` GCP PDs  (In-tree volume plugin)
    * `standard-rwo`  provisions `balanced` GCP persistent disk (CSI based)
    * `premium-rwo` provisions GCP SSD PDs (CSI based)


!!! info
    The **RECLAIMPOLICY** field tells the cluster what to do with the volume after it has been released of its claim. Currently, volumes can either be *Retained*, *Recycled*, or *Deleted*

      * `Delete` reclaim policy, deletion removes both the PersistentVolume object from Kubernetes, as well as the associated storage asset in the external infrastructure, such as an AWS EBS, GCE PD, Azure Disk
      * `Retain` reclaim policy allows for manual reclamation of the resource. When the persistent volume is released (this happens when you delete the claim that’s bound to it), Kubernetes retains the volume. The cluster administrator must manually reclaim the volume. This is the default policy for manually created persistent volumes aka Static Provisioners
      * `Recycle` - This option is deprecated and shouldn’t be used as it may not be supported by the underlying volume plugin. This policy typically causes all files on the volume to be deleted and makes the persistent volume available again without the need to delete and recreate it.

    If no `reclaimPolicy` is specified when a `StorageClass` object is created, it will default to Delete.

!!! info
    The **VOLUMEBINDINGMODE** field controls when volume binding and dynamic provisioning should occur. When unset, "Immediate" mode is used by default. 

    * Immediate
    
     The `Immediate` mode indicates that volume binding and dynamic provisioning occurs once the `PersistentVolumeClaim` is created. For storage backends that are topology-constrained and not globally accessible from all Nodes in the cluster, PersistentVolumes will be bound or provisioned without knowledge of the Pod's scheduling requirements. This may result in unschedulable Pods.

    * `WaitForFirstConsumer` The volume is provisioned and bound to the claim when the first pod that uses this claim is created. This mode is used for topology-constrained volume types. 

    The following plugins support `WaitForFirstConsumer` with dynamic provisioning:

    *  AWSElasticBlockStore
    *  GCEPersistentDisk
    *  AzureDisk

```
kubectl  describe sc
```
**Output:**
```
Name:                  premium-rwo
IsDefaultClass:        No
Annotations:           components.gke.io/component-name=pdcsi,components.gke.io/component-version=0.9.6,components.gke.io/layer=addon
Provisioner:           pd.csi.storage.gke.io
Parameters:            type=pd-ssd
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>


Name:                  standard
IsDefaultClass:        Yes
Annotations:           storageclass.kubernetes.io/is-default-class=true
Provisioner:           kubernetes.io/gce-pd
Parameters:            type=pd-standard
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>


Name:                  standard-rwo
IsDefaultClass:        No
Annotations:           components.gke.io/layer=addon,storageclass.kubernetes.io/is-default-class=false
Provisioner:           pd.csi.storage.gke.io
Parameters:            type=pd-balanced
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```


!!! summary
    The StorageClass resource specifies which provisioner should be used for provisioning the persistent volume when a persistent volume claim requests this storage class. The parameters defined in the storage class definition are passed to the provisioner and are specific to each provisioner plugin.


Step 2 Let's create a new `StorageClass` for Regional PDs:

```
cat > regionalpd-sc.yaml << EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: regionalpd-storageclass
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
  replication-type: regional-pd
allowedTopologies:
  - matchLabelExpressions:
      - key: failure-domain.beta.kubernetes.io/zone
        values:
          - us-central1-b
          - us-central1-c
EOF
```

```
kubectl apply  -f regionalpd-sc.yaml
```


```
kubectl  describe sc regionalpd-storageclass
```

!!! result
    We've created a new `StorageClass` that uses GCP PD csi provisioner to create Regional Disks in GCP.


**Step 3** Create a Persistent Volume Claim (PVC) `pvc-demo-ssd.yaml` file that will Dynamically creates 30G GCP PD Persistent Volume (PV),  using SSD persistent disk Provisioner.


```
cat > pvc-demo-ssd.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hello-web-disk
spec:
  storageClassName: premium-rwo
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30G
EOF
```


**Step 3** Create a PVC:

```
kubectl create -f pvc-demo-ssd.yaml
```


**Step 4** Verify  `STATUS` of PVC

```
kubectl get pvc
```

**Output:**
```
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc   Pending                                      standard-rwo   5s
```

List PVs:
```
kubectl get pv
```

List GCP Disks:

```
gcloud compute disks list
```


!!! result 
    What we see is that:

    * `PVC` is in `Pending` 
    * `PV` is not created
    * GCP `PD` is not created


`Question`: why PVC is in Pending State ?



**Step 5** Let's review `VolumeBindingMode` of `premium-rwo` Storage Class:

```
kubectl  describe sc premium-rwo | grep VolumeBindingMode
```

!!! Info

    This StorageClass using VolumeBindingMode -  `WaitForFirstConsumer` that creates PV only, when the first pod that uses this claim is created. 


!!! result
    Ok so if we want PV created we actually need to create a `Pod` first. This mode is especially important in the Cloud, as `Pods` can be created in different zones and so `PV` needs to be created in the correct zone as well.



**Step 6** Create a `pod-volume-demo.yaml` manifest that will create a `Pod` and mount `Persistent Volume` from `hello-web-disk` `PVC`.



```
cat > pod-volume-demo.yaml << EOF
kind: Pod
apiVersion: v1
metadata:
  name: pvc-demo-pod
spec:
  containers:
    - name: frontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: pvc-demo-volume
  volumes:
    - name: pvc-demo-volume
      persistentVolumeClaim:
        claimName: hello-web-disk
EOF
```


**Step 3** Create a Pod


```
kubectl create -f pod-volume-demo.yaml
```


**Step 4** Verify  `STATUS` of PVC now

```
kubectl get pvc
```

**Output:**
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
hello-web-disk   Bound    pvc-4c006173-284e-4786-b752-028bdae768e9   28Gi       RWO            premium-rwo    15m
```

!!! result
    PVC `STATUS` shows as Claim `hello-web-disk` as `Bound`, and that Claim has been attached to   VOLUME `pvc-4c006173-284e-4786-b752-028bdae768e9` with `CAPACITY` 28Gi and `ACCESS MODES` RWO via `STORAGECLASS` premium-rwo using SSD.

List PVs:

```
kubectl get pv
```

!!! result
    PV `STATUS` shows as `Bound` to the  `CLAIM` default/hello-web-disk, with `RECLAIM POLICY` Delete, 
    meaning that SSD Disk will be deleted after PVC is deleted from Kubernetes.

**Output:**
```
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
persistentvolume/pvc-4c006173-284e-4786-b752-028bdae768e9   28Gi       RWO            Delete           Bound    default/hello-web-disk   premium-rwo             14m
```


List GCP Disks:

```
gcloud compute disks list
```



**Output:**
```
pvc-4c006173-284e-4786-b752-028bdae768e9    us-central1-c  zone            28       pd-ssd       READ
```

!!! result
    We can see that CSI Provisioner created SSD disk on GCP infrastructure.
    


**Step 5** Verify  `STATUS` of Pod

```
kubectl get pod
```

**Output:**
```
NAME           READY   STATUS    RESTARTS   AGE
pvc-demo-pod   1/1     Running   0          21m
```

**Step 6** Delete PVC

```
kubectl delete pod pvc-demo-pod
kubeclt delete pvc hello-web-disk
```


**Step 6** Verify resources has been released:

```
kubectl get pv,pvc,pods
```


## 2 Deploy Single MySQL Database with Volume

You can run a stateful application by creating a Kubernetes Deployment
and connecting it to an existing PersistentVolume using a PersistentVolumeClaim.

**Step 1** Below Manifest file going to creates 3 Kubernetes resources:

  * `PersistentVolumeClaim` that looks for a 2G volume. This claim will be
  satisfied by dynamic provisioner `general` and appropriate PV going to be created
  * `Deployment` that runs MySQL and references the PersistentVolumeClaim that is
  mounted in /var/lib/mysql.
  * `Service` that depoyed as `ClusterIP:None` that lets the Service DNS name
  resolve directly to the Pod’s IP

```
kubectl create -f - <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6.37
        name: mysql
        env:
          # Use secret in real use case
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
EOF
```

!!! note
    The password is defined inside of the Manifest as environment,
    which is not insecure. See [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
    for a secure solution.

!!! result
    Single node MySQL database has been deployed with a Volume

**Step 3** Display information about the Deployment:

```
kubectl describe deployment mysql
```

**Output:**
```
 Name:                 mysql
 Namespace:            default
 CreationTimestamp:    Tue, 01 Nov 2016 11:18:45 -0700
 Labels:               app=mysql
 Annotations:          deployment.kubernetes.io/revision=1
 Selector:             app=mysql
 Replicas:             1 desired | 1 updated | 1 total | 0 available | 1 unavailable
 StrategyType:         Recreate
 MinReadySeconds:      0
 Pod Template:
   Labels:       app=mysql
   Containers:
    mysql:
     Image:      mysql:5.6
     Port:       3306/TCP
     Environment:
       MYSQL_ROOT_PASSWORD:      password
     Mounts:
       /var/lib/mysql from mysql-persistent-storage (rw)
   Volumes:
    mysql-persistent-storage:
     Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
     ClaimName:  mysql-pv-claim
     ReadOnly:   false
 Conditions:
   Type          Status  Reason
   ----          ------  ------
   Available     False   MinimumReplicasUnavailable
   Progressing   True    ReplicaSetUpdated
 OldReplicaSets:       <none>
 NewReplicaSet:        mysql-63082529 (1/1 replicas created)
 Events:
   FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
   ---------    --------    -----    ----                -------------    --------    ------            -------
   33s          33s         1        {deployment-controller }             Normal      ScalingReplicaSet Scaled up replica set mysql-63082529 to 1
```

**Step 4** List the pods created by the Deployment:


```
kubectl get pods -l app=mysql
```

**Output:**
```
NAME                   READY     STATUS    RESTARTS   AGE
mysql-63082529-2z3ki   1/1       Running   0          3m
```
**Step 5** Inspect the PersistentVolumeClaim:

```
kubectl describe pvc mysql-pv-claim
```

**Output:**
```
 Name:         mysql-pv-claim
 Namespace:    default
 StorageClass:
 Status:       Bound
 Volume:       mysql-pv
 Labels:       <none>
 Annotations:    pv.kubernetes.io/bind-completed=yes
                 pv.kubernetes.io/bound-by-controller=yes
 Capacity:     20Gi
 Access Modes: RWO
 Events:       <none>
```

**Step 5** Inspect created PersistentVolume:

```
kubectl get pv
kubectl describe pv
gcloud compute disks list
```


**Step 6**  Access the MySQL instance

The Service option `clusterIP: None` lets the Service DNS name resolve directly
to the Pod's IP address. This is optimal when you have only one Pod behind a
Service and you don't intend to increase the number of Pods.

Run a MySQL client to connect to the server:

```
kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h mysql -ppassword

```

This command creates a new Pod in the cluster running a MySQL client
and connects it to the server through the Service. If it connects, you
know your stateful MySQL database is up and running.

```
Waiting for pod default/mysql-client-274442439-zyp6i to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

mysql> exit
```

**Step 7**  Update the MySQL instance

The image or any other part of the Deployment can be updated as usual
with the `kubectl apply` command.

!!! important

    * Don't scale the app. This setup is for single-instance apps
      only. The underlying PersistentVolume can only be mounted to one
      Pod. For clustered stateful apps, see the
      [StatefulSet documentation](/docs/concepts/workloads/controllers/statefulset/).
    * Use `strategy:` `type: Recreate` in the Deployment configuration
      YAML file. This instructs Kubernetes to _not_ use rolling
      updates. Rolling updates will not work, as you cannot have more than
      one Pod running at a time. The `Recreate` strategy will stop the
      first pod before creating a new one with the updated configuration.

**Step 8** Delete the MySQL instance

Delete the deployed objects by name:

```
kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
```

Since we used a dynamic provisioner, it automatically deletes the
PersistentVolume when it sees that you deleted the PersistentVolumeClaim.

**Step 9** Check that GCP Volume has been deleted:


```
gcloud compute disks list
```


!!! note
    If PersistentVolume was manually provisioned, it is requrire to manually
    delete it, as well as release the underlying resource.



## 3 Deploying highly available PostgreSQL with GKE

### 3.1 Deploying PostgreSQL

**Step 1**  Create `regional persistent disk` StorageClass

```
cat <<EOF | kubectl create -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: regionalpd-storageclass
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
allowedTopologies:
  - matchLabelExpressions:
      - key: failure-domain.beta.kubernetes.io/zone
        values:
          - us-central1-b
          - us-central1-c
EOF
```

**Step 2**  Create PersistentVolumeClaim based on a `regional persistent disk` StorageClass

```
cat <<EOF | kubectl create -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-pv
spec:
  storageClassName: regionalpd-storageclass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 300Gi
EOF
```

**Step 3**  Create a PostgreSQL deployment:

```
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
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
         image: postgres:10
         resources:
           limits:
             cpu: "1"
             memory: "3Gi"
           requests:
             cpu: "1"
             memory: "2Gi"
         ports:
           - containerPort: 5432
         env:
           - name: POSTGRES_PASSWORD
             value: password
           - name: PGDATA
             value: /var/lib/postgresql/data/pgdata
         volumeMounts:
           - mountPath: /var/lib/postgresql/data
             name: postgredb
     volumes:
       - name: postgredb
         persistentVolumeClaim:
           claimName: postgresql-pv
EOF
```


**Step 4** Create PostgreSQL service:

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  ports:
    - port: 5432
  selector:
    app: postgres
  clusterIP: None
EOF
```

**Step 5** Check Regional PVs has been Provisioned based on PVC request
```
kubectl get pvc,pv
```

**Step 6** Check that Postgres is up and `Running`

```
kubectl get deploy,pods
```


### 3.2 Creating a test dataset


**Step 1**  Connect to your PostgreSQL instance:

```
POD=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $1}'`

kubectl exec -it $POD -- psql -U postgres
```

**Step 2**  Create a database and a table, and then insert some test rows:


```
create database gke_test_regional;

\c gke_test_regional;

CREATE TABLE test(
   data VARCHAR (255) NULL
);

insert into test values
  ('Learning GKE is fun'),
  ('Databases on GKE are easy');
```

**Step 3**  Verify that the test rows were inserted, select all rows:

```
select * from test;
```

**Step 4** Exit the PostgreSQL shell:

```
\q
```

### 3.3 Simulating database instance failover


**Step 0**  Identify the node that is currently hosting PostgreSQL


```
kubectl get pods -l app=postgres -o wide
```

!!! note
    Take a note on which of the nodes Pod is `Running`

**Step 1** Prepare that node to be CORDONED in other words Disabled for  scheduling:

```
CORDONED_NODE=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $7}'`

echo ${CORDONED_NODE}

gcloud compute instances list --filter="name=${CORDONED_NODE}"
```


**Step 2**  Disable scheduling of any new pods on this node:

```
kubectl cordon ${CORDONED_NODE}

kubectl get nodes
```

!!! result
    The node is cordoned, so scheduling is disabled on the node that the database instance resides on.


**Step 3**  Delete the existing PostgreSQL pod

```
POD=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $1}'`

kubectl delete pod ${POD}
```

**Step 4**  Verify that a new pod is created on the other node.

```
kubectl get pods -l app=postgres -o wide
```

!!! important
    It might take a while for the new pod to be ready (usually around 30 seconds).


**Step 5**  Verify the node's zone

```
NODE=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $7}'`

echo ${NODE}

gcloud compute instances list --filter="name=${NODE}"
```

!!! result
    Notice that the pod is deployed in a different zone from where the node was created at the beginning of this procedure.


**Step 6** Connect to the database instance

```
POD=`kubectl get pods -l app=postgres -o wide | grep -v NAME | awk '{print $1}'`

kubectl exec -it $POD -- psql -U postgres
```

**Step 7** Verify that the test dataset exists 

```
\c gke_test_regional;

select * from test;

\q
```

**Step 8** Re-enable scheduling for the node for which scheduling was disabled:

```
kubectl uncordon $CORDONED_NODE

```

**Step 9** Check that the node is ready again:

```
kubectl get nodes
```


**Step 10** Cleanup Postgres `Deployment` and `PVC`

```
kubectl delete pvc postgresql-pv
kubectl delete deploy postgres
```

## 4 Deploy StatefulSet


Scale up our GKE cluster to `4` nodes:

```
gcloud container clusters resize  k8s-storage --node-pool=default-pool --num-nodes=2 --region us-central1
```

### 4.1 Deploy Replicated MySQL (Master/Slaves) Cluster using StatefulSet.
Our Replicated MySQL deployment going to consists of:

  * 1 ConfigMap
  * 2 Services
  * 1 StatefulSet

**Step 1** Create the ConfigMap (just copy paste below):

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  primary.cnf: |
    # Apply this config only on the primary.
    [mysqld]
    log-bin
  replica.cnf: |
    # Apply this config only on replicas.
    [mysqld]
    super-read-only
EOF
```

!!! result
    This ConfigMap provides overrides that let you independently control
    configuration on the MySQL master and slaves. In this case:

    * master going to be able to serve replication logs to slaves
    * slaves to reject any writes that don't come via replication.

    There's nothing special about the ConfigMap itself that causes different
    portions to apply to different Pods. Each Pod decides which portion to look
    at as it's initializing, based on information provided by the StatefulSet
    controller.

**Step 2** Create 2 Services (just copy paste below):

```
cat <<EOF | kubectl create -f -
# Headless service for stable DNS entries of StatefulSet members.
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the primary: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
EOF
```

!!! info
    The Headless Service provides a home for the DNS entries that the StatefulSet
    controller creates for each Pod that's part of the set. Because the Headless
    Service is named mysql, the Pods are accessible by resolving <pod-name>.mysql
    from within any other Pod in the same Kubernetes cluster and namespace.

    The Client Service, called mysql-read, is a normal Service with its own
    cluster IP that distributes connections across all MySQL Pods that report
    being Ready. The set of potential endpoints includes the MySQL master and
    all slaves.

!!! note
    Only read queries can use the load-balanced Client Service. Because there is
    only one MySQL master, clients should connect directly to the MySQL master
    Pod (through its DNS entry within the Headless Service) to execute writes.


**Step 3** Create StatefulSet  `mysql-statefulset.yaml`  manifest:

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing replica. (Need to remove the tailing semicolon!)
            cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_slave_info xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from primary. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm -f xtrabackup_binlog_info xtrabackup_slave_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            mysql -h 127.0.0.1 \
                  -e "$(<change_master_to.sql.in), \
                          MASTER_HOST='mysql-0.mysql', \
                          MASTER_USER='root', \
                          MASTER_PASSWORD='', \
                          MASTER_CONNECT_RETRY=10; \
                        START SLAVE;" || exit 1
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard-rwo"
      resources:
        requests:
          storage: 10Gi
```

Deploy `StatefulSet`:

```
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-statefulset.yaml
```


**Step 4** Monitor Deployment Process/Sequence

```
watch kubectl get statefulset,pvc,pv,pods -l app=mysql
```

Press `Ctrl+C` to cancel the watch when all pods, pvc and statefulset provisioned.

```
kubectl get  pods
```

**Output:**
```
NAME      READY   STATUS     RESTARTS   AGE
mysql-0   2/2     Running    0          2m6s
mysql-1   2/2     Running    0          77s
```


!!! result
    The StatefulSet controller started Pods one at a time, in order by their
    *ordinal index*. It waits until each Pod reports being Ready before starting
    the next one.

    In addition, the controller assigned each Pod a *unique*, stable name of the
    form `<statefulset-name>-<ordinal-index>`. In this case, that results in Pods
    named `mysql-0`, `mysql-1`, and `mysql-2`.

    The Pod template in the above StatefulSet manifest takes advantage of these
    properties to perform orderly startup of MySQL replication.

    **Generating configuration**
    Before starting any of the containers in the Pod spec, the Pod first runs any
    [Init Containers] in the order defined.

    The first Init Container, named `init-mysql`, generates special MySQL config
    files based on the ordinal index.

    The script determines its own ordinal index by extracting it from the end of
    the Pod name, which is returned by the `hostname` command.
    Then it saves the ordinal (with a numeric offset to avoid reserved values)
    into a file called `server-id.cnf` in the MySQL `conf.d` directory.
    This translates the unique, stable identity provided by the StatefulSet
    controller into the domain of MySQL server IDs, which require the same
    properties.

    The script in the `init-mysql` container also applies either `master.cnf` or
    `slave.cnf` from the ConfigMap by copying the contents into `conf.d`.
    Because the example topology consists of a single MySQL master and any number of
    slaves, the script simply assigns ordinal `0` to be the master, and everyone
    else to be slaves.
    Combined with the StatefulSet controller's `deployment order guarantee`
    ensures the MySQL master is Ready before creating slaves, so they can begin
    replicating.

    **Cloning existing data**
    In general, when a new Pod joins the set as a slave, it must assume the MySQL
    master might already have data on it. It also must assume that the replication
    logs might not go all the way back to the beginning of time.
    These conservative assumptions are the key to allow a running StatefulSet
    to scale up and down over time, rather than being fixed at its initial size.

    The second Init Container, named `clone-mysql`, performs a clone operation on
    a slave Pod the first time it starts up on an empty PersistentVolume.
    That means it copies all existing data from another running Pod,
    so its local state is consistent enough to begin replicating from the master.

    MySQL itself does not provide a mechanism to do this, so the example uses a
    popular open-source tool called Percona XtraBackup.
    During the clone, the source MySQL server might suffer reduced performance.
    To minimize impact on the MySQL master, the script instructs each Pod to clone
    from the Pod whose ordinal index is one lower.
    This works because the StatefulSet controller always ensures Pod `N` is
    Ready before starting Pod `N+1`.

    **Starting replication**
    After the Init Containers complete successfully, the regular containers run.
    The MySQL Pods consist of a `mysql` container that runs the actual `mysqld`
    server, and an `xtrabackup` container that acts as a
    [sidecar](http://blog.kubernetes.io/2015/06/the-distributed-system-toolkit-patterns.html).

    The `xtrabackup` sidecar looks at the cloned data files and determines if
    it's necessary to initialize MySQL replication on the slave.
    If so, it waits for `mysqld` to be ready and then executes the
    `CHANGE MASTER TO` and `START SLAVE` commands with replication parameters
    extracted from the XtraBackup clone files.

    Once a slave begins replication, it remembers its MySQL master and
    reconnects automatically if the server restarts or the connection dies.
    Also, because slaves look for the master at its stable DNS name
    (`mysql-0.mysql`), they automatically find the master even if it gets a new
    Pod IP due to being rescheduled.

    Lastly, after starting replication, the `xtrabackup` container listens for
    connections from other Pods requesting a data clone.
    This server remains up indefinitely in case the StatefulSet scales up, or in
    case the next Pod loses its PersistentVolumeClaim and needs to redo the clone.

### 4.2 Test the MySQL cluster app and running
**Step 1** Create Database, Table and message on Master MySQL database

Send test queries to the MySQL master (hostname `mysql-0.mysql`)
by running a temporary container with the `mysql:5.7` image and running the
`mysql` client binary.

```shell
kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
```

**Step 2** Verify that recorded data has been replicated to the slaves:

Use the hostname `mysql-read` to send test queries to any server that reports
being Ready:

```shell
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
```

You should get output like this:

```
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

**Step 3** Demonstrate that the `mysql-read` Service distributes connections across
servers, you can run `SELECT @@hostname` in a loop:

```shell
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@hostname,NOW()'; done"
```

You should see the reported `@@hostname` change randomly, because a different
endpoint might be selected upon each connection attempt:

```
+-------------+---------------------+
| @@hostname  | NOW()               |
+-------------+---------------------+
|         100 | 2006-01-02 15:04:05 |
+-------------+---------------------+
+-------------+---------------------+
| @@hostname  | NOW()               |
+-------------+---------------------+
|         102 | 2006-01-02 15:04:06 |
+-------------+---------------------+
+-------------+---------------------+
| @@hostname  | NOW()               |
+-------------+---------------------+
|         101 | 2006-01-02 15:04:07 |
+-------------+---------------------+
```

You can press **Ctrl+C** when you want to stop the loop, but it's useful to keep
it running in another window so you can see the effects of the following steps.



### 4.3 Delete Pods

The StatefulSet recreates Pods if they're deleted, similar to what a
ReplicaSet does for stateless Pods.

**Step 1** Try to fail Mysql cluster by deleting `mysql-1` pod:

```shell
kubectl delete pod mysql-1
```

The StatefulSet controller notices that no `mysql-1` Pod exists anymore,
and creates a new one with the same name and linked to the same
PersistentVolumeClaim.

**Step 5** Monitor Deployment Process/Sequence

```
watch kubectl get statefulset,pvc,pv,pods -l app=mysql
```

!!! result
    You should see server ID `102` disappear from the loop output for a while
    and then return on its own.

### 4.4 Scaling the number of slaves
With MySQL replication, you can scale your read query capacity by adding slaves.
With StatefulSet, you can do this with a single command:

**Step 1** Scale up statefulset:

```shell
kubectl scale statefulset mysql  --replicas=5
```


**Step 2** Watch the new Pods come up by running:

```shell
kubectl get pods -l app=mysql --watch
```

**Step 3** Watch the new Pods come up by running:

```shell
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
```

```
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

**Step 4** Scaling back down is also seamless:

```shell
kubectl scale statefulset mysql --replicas=3
```

Note, however, that while scaling up creates new PersistentVolumeClaims
automatically, scaling down does not automatically delete these PVCs.
This gives you the choice to keep those initialized PVCs around to make
scaling back up quicker, or to extract data before deleting them.

You can see this by running:

```shell
kubectl get pvc -l app=mysql
```

Which shows that all 3 PVCs still exist, despite having scaled the
StatefulSet down to 1:

```
NAME           STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
data-mysql-0   Bound     pvc-8acbf5dc-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-1   Bound     pvc-8ad39820-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-2   Bound     pvc-8ad69a6d-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
```

If you don't intend to reuse the extra PVCs, you can delete them:

```shell
kubectl delete pvc data-mysql-0
kubectl delete pvc data-mysql-1
kubectl delete pvc data-mysql-2
kubectl delete pvc data-mysql-3
kubectl delete pvc data-mysql-4
```

**Step 5** Cancel the `SELECT @@server_id` loop by pressing **Ctrl+C** in its terminal,
   or running the following from another terminal:

```shell
kubectl delete pod mysql-client-loop --now
```

**Step 6** Delete the StatefulSet. This also begins terminating the Pods.

```shell
kubectl delete statefulset mysql
```


## 5 Cleaning Up

**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-storage
```

