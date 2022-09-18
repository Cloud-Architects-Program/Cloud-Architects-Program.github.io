Lab 2 Creating Production GKE Cluster

**Objective:**

  
## 1 Creating Production GKE Cluster


### 1.1 Locate Assignment 2

**Step 1**  Clone `ycit020` repo with Kubernetes manifests, which going to use for our work:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit020
cd ~/ycit020/Assignment2/
ls
```

!!! result 
    You can see Kubernetes manifests with Assignment tasks.


**Step 2** Go into your personal Google Cloud Source Repository:

```
MY_REPO=your_student_id-notepad
```

!!! note
    Replace $student_id with your ID

```
cd ~/$MY_REPO
```

```
git pull                              # Pull latest code from you repo
```

**Step 3** Copy Assignment 2 `deploy_a2` folder to your repo:

```
cp -r ~/ycit020/Assignment2/deploy_a2 deploy_ycit020_a2
```


**Step 4** Create `Document` for `Assignment 2`, that will be used for grading:

```
cd ~/$MY_REPO
mkdir docs
cat > docs/production_gke.md << EOF
# Creating Production GKE Cluster

**Step 1** Create a vpc network:

**Step 2** Create a subnet VPC design:

**Step 3** Create a subnet with primary and secondary ranges:

**Step 4** Create a Private GKE Cluster:

**Step 5** Create a Cloud Router:

**Step 6** Create a Cloud Nat Gateway:

EOF
```

**Step 5** Commit `deploy` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding documentation for ycit020 assignment 2"
```

**Step 6** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

### 1.2 Creating a GCP project


!!! note
    We going to create a temporarily project for this assignment.
    While you going to store code in existing project that we've used so far in class

Set variables:

```
export ORG=<student-name>
```

```
export PRODUCT=notepad
export ENV=dev
export PROJECT_PREFIX=1   # Project has to be unique
export PROJECT_ID=$ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```

Create a project:

```
gcloud projects create $ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```


List billings and take note of `ACCOUNT_ID`:

```
gcloud alpha billing accounts list
```

Define a variable for `ACCOUNT_ID`:

```
export ACCOUNT_ID=<Your Billing Account>
```

Attach your project to the correct billing account:

```
gcloud alpha billing accounts projects link $PROJECT_ID --billing-account=$ACCOUNT_ID
```


Set the newly created project as default for gcloud:

```
gcloud config set project $PROJECT_ID
gcloud config set compute/region us-central1
```


Enable `compute`, `container`, `cloudresourcemanager` APIs:

```
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com 
gcloud services enable cloudresourcemanager.googleapis.com
```

!!! note
    Enabling GCP Service APIs very important step in automation (e.g. terraform)


Get a list of services that enabled in your project:

```
gcloud services list
```

!!! note
    Some services APIs enabled by default during project creation

### 1.2 Deleting Default VPC

Observe Asset Inventory in GCP UI:

```
Products -> IAM & Admin -> Asset Inventory -> Overview
```

!!! result
    You can see that `vpc` default network spans across all GCP Regions, which
    for many companies will not be acceptable practice (e.g. GDPR)


List all networks in a project:

```
gcloud compute networks list
```

**Output:**
```
NAME     SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
default  AUTO         REGIONAL
```

Review existing firewall rules for `default` vpc:


```
gcloud compute firewall-rules list
```

Also check in Google cloud UI:

```
Networking->Firewalls
```


Delete firewall rules associated with `default` vpc network:

```
gcloud compute firewall-rules delete default-allow-internal
gcloud compute firewall-rules delete default-allow-ssh
gcloud compute firewall-rules delete default-allow-rdp
gcloud compute firewall-rules delete default-allow-icmp
```


Delete the `Default` network, following best practices:


```
gcloud compute networks delete default
```

### 1.3 Creating a custom mode network (VPC)


**Task N1:** Using reference doc: [Creating a custom mode network](https://cloud.google.com/vpc/docs/using-vpc#create-custom-network). Create a new custom mode VPC network using gcloud command with following parameters:

  * Network name: `$ORG-$PRODUCT-$ENV-vpc`
  * Subnet mode: `custom`
  * Bgp routing mode: `regional`
  * MTUs: `default`

Document a command to create `vpc` network in `docs/production_gke.md` doc under `step 1`.

**Step 1:** Create a new custom mode VPC network:


with name `ORG-$PRODUCT-$ENV-vpc`, with subnet mode `custom`,
and `regional`.

**Step 1:** Create a new custom mode VPC network:

Set variables:

```
export PRODUCT=notepad
export ENV=dev
```

```
TODO: gcloud create network
```


Review created network:

```
gcloud compute networks list
```

Also check in Google cloud UI:

```
Networking->VPC Networks
```

**Step 2:** Create firewall rules `default-allow-internal` and `default-allow-ssh`:

```
gcloud compute firewall-rules create $ORG-$PRODUCT-$ENV-allow-tcp-ssh-icmp --network $ORG-$PRODUCT-$ENV-vpc --allow tcp:22,tcp:3389,icmp
gcloud compute firewall-rules create allow-internal --network $ORG-$PRODUCT-$ENV-vpc --allow tcp,udp,icmp --source-ranges 10.128.0.0/22
```

Reference: https://cloud.google.com/kubernetes-engine/docs/concepts/firewall-rules


Review created firewall rules:

```
gcloud compute firewall-rules list
```

Also check in Google cloud UI:

```
Networking->Firewalls
```


### 1.4 Design and create a `user-managed` subnet 

After you've created VPC network, it is require to add subnet to it.

**Task N2:** In order to ensure that GKE clusters in you organization doesn't overlap each other,
design VPC subnet ranges for `dev`, `stg`, `prd` VPC-native GKE clusters, where

  * Nodes *primary range* belongs to Class A (10.0.0.0/x)
  * Pods  *secondary ranges*  belongs to Class B (172.20.0.0/x) 
  * Services  *secondary ranges*   belongs to Class B (172.100.0.0/x)
  * CIDR range for master-ipv4-cidr (k8s api range) belongs to Class B (172.16.0.0/x)

In your design assume that each cluster will have maximum of 59 Nodes with max 110 Pods per node and 1700 Service per cluster.

Use following reference docs:

  1. [VPC-native clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)
  2. [GKE address management](https://cloud.google.com/architecture/gke-address-management-options) 


Using tables from [VPC-native clusters document](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips) and online subnet [calculator](https://www.subnet-calculator.com/), create a table for `dev`, `stg`, `prd` in the following format and store result under `docs/production_gke.md`:

```
env |     subnet    |    pod range     |    srv range   | kubectl api range
dev |     IP range  |     IP range     |     IP range   |     IP range    
stg |     IP range  |     IP range     |     IP range   |     IP range     
prd |     IP range  |     IP range     |     IP range   |     IP range     
```

Document a subnet VPC design in `docs/production_gke.md` doc under `step 2`.


**Task N3:** Create a `user-managed` subnet.

Create a subnet for `dev` cluster, taking in consideration VPC subnet ranges created in above table, where:

  * Subnet name: `$ORG-$PRODUCT-$ENV-vpc`
  * Node Range: See column `subnet` in above table for `dev` cluster
  * Secondary service range with name: `services`
  * Secondary Ranges:
    * Service range name: `services`
    * Service range CIDR: See column `srv range` in above table for `dev` cluster
    * Pods range name: `pods`
    * Pods range CIDR: See column `pod range` in above table for `dev` cluster
  * Features:
    * Flow Logs
    * Private IP Google Access


```
TODO: gcloud subnet create 
```

Reference docs:

  1. [Creating a private cluster with custom subnet](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#custom_subnet)
  2. [VPC-native clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)
  3. Use `gcloud subnets create` command [reference](https://cloud.google.com/sdk/gcloud/reference/compute/networks/subnets/create) for all available options.


Review created subnet:

```
gcloud compute networks subnets list
```

Also check in Google cloud UI:

```
Networking->VPC Networks -> Click VPC network and check `Subnet` tab
```


Document a subnet VPC creation command in `docs/production_gke.md` doc under `step 3`.


### 1.5 Creating a Private, Regional and VPC Native GKE Cluster


**Task N4:** Create `dev` *Private* GKE Cluster with no client access to the public endpoint and following values:

  * Cluster name: `$ORG-$PRODUCT-$ENV-cluster`
  * Secondary pod range with name: `pods`
  * Secondary service range with name: `services`
  * VM size: `e2-small`
  * Node count: 1 per zone
  * GKE Control plane is replicated across three zones of a region: `us-central1`
  * GKE version: "1.20.8-gke.700"
  * Cluster Node Communication: `VPC Native`
  * Cluster Nodes access: *Private* Node GKE Cluster with *Public API* endpoint
  * Cluster K8s API access: *with limited access to the public endpoint via authorized network*


Use following reference docs:

  1. [Creating a private cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters)
  2. [GKE Release Channels](https://cloud.google.com/kubernetes-engine/docs/concepts/release-channels#channels)
  3. Use `gcloud container clusters create` command [reference](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create) for all available options.

```
#TODO gcloud clusters create $ORG-$PRODUCT-$ENV-cluster
```

Document a GKE cluster creation command in `docs/production_gke.md` doc under `step 4`.


!!! result 
    The private cluster is now created. `gcloud` has set up `kubectl` to authenticate with the private cluster but the cluster's K8s API server will only accept connections from the primary range of your subnetwork, and the secondary range of your subnetwork that is used for pods. That means nobody can access K8s API at this point of time, until we specify allowed ranges.



**Step 3** Authenticate to the cluster.

```
gcloud container clusters get-credentials $ORG-$PRODUCT-$ENV-cluster --region us-central1 
``` 


**Step 4:** Connecting to a Private Cluster

Let's try to connect to the cluster:

```
kubectl get pods
```

```
Unable to connect to the server i/o timeout

```

!!! Fail
    This fails because private clusters firewall traffic to the master by default. In order to connect to the cluster you need to make use of the master authorized networks feature. 

**Step 4:** Enable Master Authorized networks on you cluster

Suppose you have a group of machines, outside of your VPC network, belongs to your organization. You could authorize those machines to access the public endpoint.


Here we will enable master authorized networks and whitelist the IP address for our Cloud Shell instance, to allow access to the master:

```
gcloud container clusters update $ORG-$PRODUCT-$ENV-cluster --enable-master-authorized-networks --master-authorized-networks $(curl ipinfo.io/ip)/32 --region us-central1
```

Now we can access the API server using `kubectl` from GCP console:


!!! note
    In real life it could be CIDR Range for you company, so only engineers or CI/CD systems from you company can connect to `kubectl` apis in secure manner.



Now we can access the API server using kubectl:

```
kubectl get pods
```


```
kubectl run hello-web --image=gcr.io/google-samples/hello-app:1.0 --port=8080
```


```
kubectl get pods
```

**Output:**
```
NAME        READY   STATUS    RESTARTS   AGE
hello-web   1/1     Running   0          7s
```

!!! result
    We can deploy Pods to our Private Cluster.

```
kubectl delete pod hello-web
```


**Step 2:** Testing Outbound Traffic

Outbound traffic is not routable in private clusters so access to the internet is limited. This isolates pods that are running sensitive workloads.

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: wget
spec:
  containers:
  - name: wget
    image: alpine
    command: ['wget', '-T', '5', 'http://www.example.com/']
  restartPolicy: Never
EOF
```


```
kubectl get pods
```

**Output:**
```
NAME   READY   STATUS   RESTARTS   AGE
wget   0/1     Error    0          4m41s
```

```
kubectl logs wget
```
**Output:**
```
Connecting to www.example.com (93.184.216.34:80)
wget: download timed out
```

!!! results
    Pods can't access internet, because it's a private cluster


!!! note
    Normally, if cluster is Private and doesn't have Cloud Nat configured, users
    can't even deploy images from Docker Hub. See troubleshooting details [here](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#docker_hub)


### 1.6 Create a Google Cloud Nat

Private clusters give you the ability to isolate nodes from having inbound and outbound connectivity to the public internet. This isolation is achieved as the nodes have internal IP addresses only.

If you want to provide outbound internet access for certain private nodes, you can use [Cloud NAT](https://cloud.google.com/nat/docs/overview#NATwithGKE)



Cloud NAT is a distributed, software-defined managed service. It's not based on proxy VMs or appliances.


You configure a NAT gateway on a Cloud Router, which provides the control plane for NAT, holding configuration parameters that you specify. Google Cloud runs and maintains processes on the physical machines that run your Google Cloud VMs.

Cloud NAT can be configured to automatically scale the number of NAT IP addresses that it uses, and it supports VMs that belong to managed instance groups, including those with autoscaling enabled.


**Step 1:** Create a Cloud NAT configuration using Cloud Router

Create Cloud Router in the same region as the instances that use Cloud NAT. Cloud NAT is only used to place NAT information onto the VMs. It is not used as part of the actual NAT gateway.


**Step 1a:** Create a `Cloud Router`:


```
gcloud compute routers create gke-nat-router \
    --network $ORG-$PRODUCT-$ENV-vpc \
    --region us-central1
```


Verify created Cloud Router:

```
Networking -> Hybrid Connectivity -> Cloud Routers
```


**Step 1b:** Create a `Cloud Nat` Gateway:

```
gcloud compute routers nats create nat-config \
    --router-region us-central1 \
    --router gke-nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

Verify created `Cloud Nat`:

```
Networking -> Network Services -> Cloud NAT
```


!!! note
     Cloud NAT uses Cloud Router only to group NAT configuration information (control plane). Cloud NAT does not direct a Cloud Router to use BGP or to add routes. NAT traffic does not pass through a Cloud Router (data plane).



Document Cloud Router and Cloud Nat creation  command in `docs/production_gke.md` doc under `step 5 and 6` respectively.


**Step 2:** Testing Outbound Traffic


Most outbound traffic is not routable in private clusters so access to the internet is limited. This isolates pods that are running sensitive workloads.

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: wget
spec:
  containers:
  - name: wget
    image: alpine
    command: ['wget', '-T', '5', 'http://www.example.com/']
  restartPolicy: Never
EOF
```


```
kubectl get pods
```

**Output:**
```
NAME                              READY   STATUS      RESTARTS   AGE
wget                              0/1     Completed   0          2m53s
```

```
kubectl logs wget
```

**Output:**
```
Connecting to www.example.com (93.184.216.34:80)
saving to 'index.html'
index.html           100% |********************************|  1256  0:00:00 ETA
'index.html' saved
```

!!! results
    Pods can access internet, thanks to our configured Cloud Nat.


## 2 Deploy NotePad Go webapp

### 2.1 Create a Namespace `dev`

**Step 1** Create 'dev' namespace that's going to be used to develop and deploy `Notestaker` application on Kubernetes using `kubetl` CLI.

```
kubectl create ns dev
```

**Step 3** Use `dev` context to create K8s resources inside this namespace.

```
kubectl config set-context --current --namespace=dev
```

**Step 4** Verify current context:

```
kubectl config view | grep namespace
```

!!! result
    `dev`

### 2.2 Create Mysql deployment

Create Mysql deployment:

**Step 1**  Deploy PVC, Deployment, Services and Network Policy:


```
cd ~/$MY_REPO/deploy_ycit020_a2
kubectl apply -f gowebapp-mysql-pvc.yaml        #Create PVC
kubectl apply -f secret-mysql.yaml              #Create Secret
kubectl apply -f gowebapp-mysql-service.yaml    #Create Service
kubectl apply -f gowebapp-mysql-deployment.yaml #Create Deployment
kubectl apply -f default-deny-dev.yaml   # deny-all Ingress Traffic to dev Namespace
kubectl apply -f gowebapp-mysql-netpol.yaml   # `mysql` pod can be accessed from the `gowebapp` pod
```


Verify status of mysql:

```
kubectl get deploy,secret,pvc
```

### 2.3 Create GoWebApp deployment

**Step 1:** Create ConfigMap for gowebapp's config.json file

```
cd ~/$MY_REPO/gowebapp/config/
kubectl create configmap gowebapp --from-file=webapp-config-json=config.json
kubectl describe configmap gowebapp
```

**Step 2** Deploy `gowebapp` app under `~/$MY_REPO/deploy_ycit020_a2/`

```
cd ~/$MY_REPO/deploy_ycit020_a2/
kubectl apply -f gowebapp-service.yaml    #Create Service
kubectl apply -f gowebapp-deployment.yaml #Create Deployment
kubectl apply -f gowebapp-netpol.yaml  #gowebapp pod from Internet on CIDR range
kubectl apply -f gowebapp-ingress.yaml   #Create Ingress
```

Verify status of gowebapp:

```
kubectl get pods,cm,ing
```


<!-- ### 2.5 TODO Scheduling a distributed `gowebapp` workload

In much the same way as multi-zone clusters. You can create workloads that are distributed among each zone for higher tolerance to failure.


```
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web-regional
  labels:
    app: hello-web
spec:
  replicas: 9
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
      - name: hello-web
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "50m"
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - hello-web
                topologyKey: failure-domain.beta.kubernetes.io/zone
EOF
```


```
kubectl get pods -o wide
```
 -->

## 3 Running GKE in Production
###  3.1 Deploy a New NodePool

A Kubernetes Engine cluster consists of a master and nodes. Kubernetes doesn't handle provisioning of nodes, so Google Kubernetes Engine handles this for you with a concept called *node pools*.

A node pool is a subset of node instances within a cluster that all have the same configuration. They map to instance templates in Google Compute Engine, which provides the VMs used by the cluster. By default a Kubernetes Engine cluster has a single node pool, but you can add or remove them as you wish to change the shape of your cluster.

In the previous example, you created a Kubernetes Engine cluster. This gave us three nodes (three e2-small (2 vCPUs, 2 GB memory), 100 GB of disk each) in a single node pool (called default-pool). Let's inspect the node pool:

```
gcloud config set compute/region us-central1
gcloud container node-pools list --cluster $ORG-$PRODUCT-$ENV-cluster
```

**Output:**

```
NAME          MACHINE_TYPE  DISK_SIZE_GB  NODE_VERSION
default-pool  e2-small     100           1.20.8-gke.700
```

If you want to add more nodes of this type, you can grow this node pool. If you want to add more nodes of a different type, you can add other node pools.

A common method of moving a cluster to larger nodes is to add a new node pool, move the work from the old nodes to the new, and delete the old node pool.

Let's add a second node pool, and migrate our workload over to it. This time we will use the larger `e2-medium`	(2 vCPUs, 4 GB memory), 100 GB of disk each machine type.

```
gcloud container node-pools create new-pool --cluster $ORG-$PRODUCT-$ENV-cluster \
    --machine-type e2-medium --num-nodes 1
```


**Output:**
```
NAME          MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
default-pool  e2-small      100           1.20.8-gke.700
new-pool      e2-medium  100           1.20.8-gke.700
```

```
gcloud container node-pools list --cluster $ORG-$PRODUCT-$ENV-cluster
```


**Output:**
```
NAME                                                  STATUS   ROLES    AGE    VERSION
gke-ayrat-notepad-dev-cl-default-pool-34b29738-b1m8   Ready    <none>   11m    v1.20.8-gke.700
gke-ayrat-notepad-dev-cl-default-pool-c22a1803-hh91   Ready    <none>   11m    v1.20.8-gke.700
gke-ayrat-notepad-dev-cl-default-pool-f2a61828-zjgk   Ready    <none>   11m    v1.20.8-gke.700
gke-ayrat-notepad-dev-cluste-new-pool-2df77edb-thgj   Ready    <none>   104s   v1.20.8-gke.700
gke-ayrat-notepad-dev-cluste-new-pool-40bedbb3-gsxr   Ready    <none>   101s   v1.20.8-gke.700
gke-ayrat-notepad-dev-cluste-new-pool-d8f28859-hgqn   Ready    <none>   60s    v1.20.8-gke.700
```

Kubernetes does not reschedule Pods as long as they are running and available, so your workload remains running on the nodes in the default pool.

Look at one of your nodes using kubectl describe. Just like you can attach labels to pods, nodes are automatically labelled with useful information which lets the scheduler make decisions and the administrator perform action on groups of nodes.

Replace "[NODE NAME]" with the name of one of your nodes from the previous step.

```
kubectl describe node [NODE NAME] | head -n 20
```

**Output:**
```
Name:               gke-ayrat-notepad-dev-cluste-new-pool-d8f28859-hgqn
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=e2-standard-4
                    beta.kubernetes.io/os=linux
                    cloud.google.com/gke-boot-disk=pd-standard
                    cloud.google.com/gke-container-runtime=containerd
                    cloud.google.com/gke-nodepool=new-pool
                    cloud.google.com/gke-os-distribution=cos
                    cloud.google.com/machine-family=e2
                    failure-domain.beta.kubernetes.io/region=us-central1
                    failure-domain.beta.kubernetes.io/zone=us-central1-b
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=gke-ayrat-notepad-dev-cluste-new-pool-d8f28859-hgqn
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=e2-standard-4
                    node.kubernetes.io/masq-agent-ds-ready=true
                    topology.gke.io/zone=us-central1-b
                    topology.kubernetes.io/region=us-central1
                    topology.kubernetes.io/zone=us-central1-b
<...>
```

You can also select nodes by node pool using the cloud.google.com/gke-nodepool label. We'll use this powerful construct shortly.


```
kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool
```

**Output:**

```
NAME                                          STATUS    ROLES     AGE       VERSION
gke-ayrat-notepad-dev-cl-default-pool-34b29738-b1m8   Ready    <none>   14m   v1.20.8-gke.700
gke-ayrat-notepad-dev-cl-default-pool-c22a1803-hh91   Ready    <none>   14m   v1.20.8-gke.700
gke-ayrat-notepad-dev-cl-default-pool-f2a61828-zjgk   Ready    <none>   14m   v1.20.8-gke.70
```


### 3.2 Migrating pods to the new Node Pool

To migrate your pods to the new node pool, we will perform the following steps:

`Cordon` the existing node pool: This operation marks the nodes in the existing node pool (default-pool) as unschedulable. Kubernetes stops scheduling new Pods to these nodes once you mark them as unschedulable.

Drain the existing node pool: This operation evicts the workloads running on the nodes of the existing node pool (default-pool) gracefully.

You could cordon an individual node using the kubectl cordon command, but running this command on each node individually would be tedious. To speed up the process, we can embed the command in a loop. Be sure you copy the whole line - it will have scrolled off the screen to the right!


```
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do kubectl cordon "$node"; done
```

**Output:**
```
node/gke-ayrat-notepad-dev-cl-default-pool-4a7a0d9d-d5q9 cordoned
node/gke-ayrat-notepad-dev-cl-default-pool-4d5f96ee-nnv9 cordoned
node/gke-ayrat-notepad-dev-cl-default-pool-eb93c399-zcs8 cordoned
```


!!! note
    This loop utilizes the command kubectl get nodes to select all nodes in the default pool (using the cloud.google.com/gke-nodepool=default-pool label), and then it iterates through and runs kubectl cordon on each one.

After running the loop, you should see that the default-pool nodes have SchedulingDisabled status in the node list:

```
kubectl get nodes
```

**Output:**
```
NAME                                            STATUS                     ROLES     AGE       VERSION
gke-ayrat-notepad-dev-cl-default-pool-4a7a0d9d-d5q9   Ready,SchedulingDisabled   <none>   38m     v1.20.8-gke.700
gke-ayrat-notepad-dev-cl-default-pool-4d5f96ee-nnv9   Ready,SchedulingDisabled   <none>   38m     v1.20.8-gke.700
gke-ayrat-notepad-dev-cl-default-pool-eb93c399-zcs8   Ready,SchedulingDisabled   <none>   38m     v1.20.8-gke.700
gke-ayrat-notepad-dev-cluste-new-pool-af1b690a-v6j9   Ready                      <none>   2m57s   v1.20.8-gke.700
gke-ayrat-notepad-dev-cluste-new-pool-f3195336-k11r   Ready                      <none>   2m44s   v1.20.8-gke.700
gke-ayrat-notepad-dev-cluste-new-pool-f47e9e43-6qzb   Ready                      <none>   2m50s   v1.20.8-gke.700
```


Next, we want to evict the Pods already scheduled on each node. To do this, we will construct another loop, this time using the kubectl drain command:


```
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do kubectl drain --force --ignore-daemonsets --delete-emptydir-data "$node"; done
```

**Output:**
```
<...>
pod/gowebapp-mysql-6ffb7f9586-prddc evicted
pod/hello-web evicted
node/gke-ayrat-notepad-dev-cl-default-pool-eb93c399-zcs8 evicted
<...>
```


As each node is drained, the pods running on it are evicted. Eviction makes sure to follow rules to provide the least disruption to the applications as possible. Users in production may want to look at more advanced features like `Pod Disruption Budgets`.

Because the default node pool is unschedulable, the pods are now running on the single machine in the new node pool:

```
kubectl get pods -o wide
```

**Output:**

```
NAME                         READY     STATUS    RESTARTS   AGE       IP          NODE
node/gke-ayrat-notepad-dev-cl-default-pool-4a7a0d9d-d5q9 cordoned
node/gke-ayrat-notepad-dev-cl-default-pool-4d5f96ee-nnv9 cordoned
node/gke-ayrat-notepad-dev-cl-default-pool-eb93c399-zcs8 cordoned
```


You can now delete the original node pool:

``` 
gcloud container node-pools delete default-pool --cluster $ORG-$PRODUCT-$ENV-cluster
```

Check that node-pool has been deleted:

```
gcloud container node-pools list --cluster $ORG-$PRODUCT-$ENV-cluster
```

!!! result
    GKE cluster running only on `new-pool` Node pool.


### 3.3 Configure Node auto-repair and Node auto-upgrades

Kubernetes Engine's node auto-repair feature helps you keep the nodes in your cluster in a healthy, running state. When enabled, Kubernetes Engine makes periodic checks on the health state of each node in your cluster. If a node fails consecutive health checks over an extended time period (approximately 10 minutes), Kubernetes Engine initiates a repair process for that node.


```
gcloud container node-pools update new-pool --cluster $ORG-$PRODUCT-$ENV-cluster --enable-autorepair
```

This will enable the autorepair feature for nodes. Please see
https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more
information on node autorepairs.




First, Enable and configure OS Login in GKE:

```
gcloud compute project-info add-metadata --metadata enable-oslogin=TRUE
```

Next, for some fun: let's break a VM!

This gcloud command will find the VM in your regional node pool which is in the default zone, and SSH into it. If you are asked to generate an SSH key just answer 'Y' at the prompt and hit enter to not set a passphrase.



```
gcloud compute ssh $(gcloud compute instances list | \
                      grep -m 1 notepad-dev | \
                      awk '{ print $1 }') \
                      --zone us-central1-a
```

You can simulate a node failure by removing the kubelet binary, which is responsible for running Pods on every Kubernetes node:

```
sudo rm /home/kubernetes/bin/kubelet && sudo systemctl restart kubelet
logout
```

Now when we check the node status we see the node is NotReady.

```
watch kubectl get nodes
```


```
NAME                                          STATUS     ROLES     AGE       VERSION
gke-gke-workshop-new-pool-42b33f8c-9grf   Ready      <none>    6m        v1.9.6-gke.1
gke-gke-workshop-new-pool-847e18a1-f1bp   Ready      <none>    6m        v1.9.6-gke.1
gke-gke-workshop-new-pool-8c4b26e9-fq8p   NotReady   <none>    6m        v1.9.6-gke.1
```


The Kubernetes Engine node repair agent will wait a few minutes in case the problem is intermittent. We'll come back to this in a minute.

Define a maintenance window
You can configure a maintenance window to have more control over when automatic upgrades are applied to Kubernetes on your cluster.

Creating a maintenance window instructs Kubernetes Engine to automatically trigger any automated tasks in your clusters, such as master upgrades, node pool upgrades, and maintenance of internal components, during a specific timeframe.

The times are specified in UTC, so select an appropriate time and set up a maintenance window for your cluster.

Open the cloud console and navigate to Kubernetes Engine. Click on the gke-workshop cluster and click the edit button at the top. Find the Maintenance Window option and select 3AM. Finally, click Save to update the cluster.

Enable node auto-upgrades
Whenever a new version of Kubernetes is released, Google upgrades your master to that version. You can then choose to upgrade your nodes to that version, bringing functionality and security updates to both the OS and the Kubernetes components.

Node Auto-Upgrades use the same update mechanism as manual node upgrades, but does the scheduled upgrades during your maintenance window.

Auto-upgrades are enabled per node pool.

```
gcloud container node-pools update new-pool --cluster $ORG-$PRODUCT-$ENV-cluster   --enable-autoupgrade
```



This will enable the autoupgrade feature for nodes. Please see
https://cloud.google.com/kubernetes-engine/docs/node-management for more
information on node autoupgrades.


```
ERROR: (gcloud.container.node-pools.update) ResponseError: code=400, message=Operation operation-1511896056410-dbda7f9f is currently operating on cluster gke-workshop. Please wait and try again once it's done.
```

That's good - that means our broken node is being repaired! Try again in a few minutes.

Check your node repair
How is that node repair coming?

After a few minutes, you will see that the master drained the node, and then removed it.

```
watch kubectl get nodes
```

```
NotReady,SchedulingDisabled
```

A few minutes after that, a new node was turned on in its place:


```
Ready
```

```
kubectl get pods
```


```
kubectl delete deployment hello-web
```


## 4 Commit Readme doc to repository and share it with Instructor/Teacher


**Step 1** Commit `docs` folder using the following Git commands:


```
cd ~/$MY_REPO
```

```
git add .
git commit -m "Readme doc for Production GKE Creation using gcloud"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```

## 5 Cleanup


```
gcloud container clusters delete $ORG-$PRODUCT-$ENV-cluster
```

!!! important
    We going to delete the `$ORG-$PRODUCT-$ENV-$PROJECT_PREFIX` only,
    please do not delete your project with Source Code Repo.

```
gcloud projects delete  $ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```



<!-- ### 3.3 Creating a secondary node pool with GPUs

TODO change to us-west1 region for this lab.

In addition to migrating from one node pool to another there may be situations where you would like to have a subset of your nodes to have a different configuration. For instance, perhaps some of your applications require the use of GPU hardware. However, it would be unnecessarily expensive if you were to attach GPUs to all nodes in your cluster.

In that case you can create a separate pool of nodes that have GPUs attached and schedule pods to use those GPU enabled nodes. Google Compute Engine allows you to attach up to 8 GPUs per node, assuming you have quota and the GPU device type is available in the zone you are using.

Let's first create a node pool where each node has one GPU attached:

```
gcloud beta container node-pools create nvidia-tesla-k80-pool --cluster $ORG-$PRODUCT-$ENV-cluster --machine-type n1-standard-1 --num-nodes 1 --accelerator type=nvidia-tesla-k80,count=1
```

Machines with GPUs have certain limitations which may affect your workflow.
Learn more at https://cloud.google.com/kubernetes-engine/docs/concepts/gpus

```
NAME                    MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
nvidia-tesla-k80-pool  n1-standard-1  100           1.9.6-gke.0
Next, we must install the NVIDIA drivers.
```

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```


Verify that the drivers are installed using the following command. Run the following commands to watch the status of the pod. When the pods are listed as 'Running", you will know that the drivers are finished installing. When you are done hit Ctrl-C to exit:

```
watch "kubectl get pods -n kube-system | grep nvidia-driver-installer"
```


You can verify that the new node has GPUs that are allocatable to pods with the following command:

```
kubectl get nodes -l cloud.google.com/gke-nodepool=nvidia-tesla-k80-pool -o yaml | grep allocatable -A4
```

```
    allocatable:
      cpu: 940m
      memory: 2708864Ki
      nvidia.com/gpu: "1"
      pods: "110"
```

Now let's create a pods that can consume GPUs:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
EOF
```

The pod should complete fairly quickly. You can verify that it was completed by watching the pod with the following command. Here we can see that the pod was run on one of the nodes with a GPU. When the pod enters 'Completed' status you can hit Ctrl-C to exit:


```
watch kubectl get pods --show-all -o wide
```
```
NAME              READY     STATUS      RESTARTS   AGE       IP          NODE
cuda-vector-add   0/1       Completed   0          10m       10.56.8.3   gke-gke-workshop-nvidia-tesla-k80-po-b310269d-gbs1
Verify the logs for the pod:
```

```
kubectl logs cuda-vector-add
```

```
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
Finally we will delete the GPU node pool:
```

```
gcloud container node-pools delete nvidia-tesla-k80-pool --cluster=$ORG-$PRODUCT-$ENV-cluster
```




### 3.4 TODO Create Preemptible Node Pools and deploy gowebapp

At this point your cluster configuration should be a single node pool named `new-pool`, with some number of 3 nodes.

`Preemptible VMs` are Google Compute Engine VM instances that last a maximum of 24 hours and provide no availability guarantees. Preemptible VMs are priced substantially lower than standard Compute Engine VMs and offer the same machine types and options.

If your workload can handle nodes disappearing, using Preemptible VMs with the Cluster Autoscaler lets you run work at a lower cost. To specify that you want to use Preemptible VMs you simply use the --preemptible flag when you create the node pool. But if you're using Preemptible VMs to cut costs, then you don't need them sitting around idle. So let's create a node pool of Preemptible VMs that starts with zero nodes, and autoscales as needed.

Hold on though: before we create it, how do we schedule work on the Preemptible VMs? These would be a special set of nodes for a special set of work - probably low priority or batch work. For that we'll use a combination of a NodeSelector and taints/tolerations. Preemptible nodes are currently in beta so we will use the beta version of the Kubernetes Engine API and the beta commands in gcloud. The full command we'll run is:


```
gcloud container node-pools create preemptible-pool \
    --cluster $ORG-$PRODUCT-$ENV-cluster --preemptible --num-nodes 0 \
    --enable-autoscaling --min-nodes 0 --max-nodes 5 \
    --node-taints=pod=preemptible:PreferNoSchedule
```


```
kubectl get nodes 
```


We now have two node pools, but the new "preemptible" pool is autoscaled and is sized to zero initially so we only see the three nodes from the autoscaled node pool that we created in the previous section.

Usually as far as Kubernetes is concerned, all nodes are valid places to schedule pods. We may prefer to reserve the preemptible pool for workloads that are explicitly marked as suiting preemption — workloads which can be replaced if they die, versus those that generally expect their nodes to be long-lived.

To direct the scheduler to schedule pods onto the nodes in the preemptible pool we must first mark the new nodes with a special label called a taint. This makes the scheduler avoid using it for certain Pods.

When our application autoscales, the Kubernetes scheduler will prefer nodes that do not have the taint. This is as opposed to requiring that new workloads run on nodes without a taint. This is an easy way to allow us to run pods from system DaemonSets like Calico or fluentd without extra work.


We can then mark pods that we want to run on the preemptible nodes with a matching toleration, which says they are OK to be assigned to nodes with that taint.

Let's update a `gowebapp` workload that's designed to run on preemptible nodes and nowhere else.

```
cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: hello-web
  name: hello-preempt
spec:
  replicas: 20
  selector:
    matchLabels:
      run: hello-web
  template:
    metadata:
      labels:
        run: hello-web
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        name: hello-web
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          requests:
            cpu: "50m"
      tolerations:
      - key: pod
        operator: Equal
        value: preemptible
        effect: PreferNoSchedule
      nodeSelector:
        cloud.google.com/gke-preemptible: "true"
EOF
```

```
watch kubectl get nodes,pods -o wide
```



Due to the NodeSelector, initially there were no nodes on which we could schedule the work. The scheduler works in tandem with the Cluster Autoscaler to provision new nodes in the pool with the node labels that match the NodeSelector. We haven't demonstrated it here, but the taint would mean prefer to prevent workloads with pods that don't tolerate the taint from being scheduled on these nodes.

As we do the cleanup for this section, let's delete the preemptible node pool and see what happens to the pods that we just created. This isn't something you would want to do in production!

```
gcloud container node-pools delete preemptible-pool --cluster \
    $ORG-$PRODUCT-$ENV-cluster
```

```
watch kubectl get pods 
```

As you can see, because of the NodeSelector, none of the pods are running. Now, delete the deployment.

```
kubectl delete deployment hello-preempt
```


!!! Congratulations
    You are now a master at node pools! So far we've learned:

    * Creating node pools 
    * Cordoning and draining nodes
    * Migrating the cluster from one machine type to another
    * Creating multiple node pools with different configurations
    * Creating nodes with GPUs
    * Scheduling pods to GPU nodes
    * Deleting node pools
    * Scale from Zero for Node Pools
    * NodeSelector and Node Affinity
    * Taints and Tolerations
    * Scheduling pods to Preemtible nodes -->


<!-- ### 3.5 Configure Node auto-repair and Node auto-upgrades

Kubernetes Engine's node auto-repair feature helps you keep the nodes in your cluster in a healthy, running state. When enabled, Kubernetes Engine makes periodic checks on the health state of each node in your cluster. If a node fails consecutive health checks over an extended time period (approximately 10 minutes), Kubernetes Engine initiates a repair process for that node.


```
gcloud container node-pools update new-pool --cluster $ORG-$PRODUCT-$ENV-cluster --enable-autorepair
```

This will enable the autorepair feature for nodes. Please see
https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more
information on node autorepairs.


Now, for some fun: let's break a VM!

This gcloud command will find the VM in your regional node pool which is in the default zone, and SSH into it. If you are asked to generate an SSH key just answer 'Y' at the prompt and hit enter to not set a passphrase.

```
gcloud compute ssh $(gcloud compute instances list | \
                      grep -m 1 gke-workshop-new | \
                      grep $(gcloud config get-value compute/zone) | \
                      awk '{ print $1 }')
```

You can simulate a node failure by removing the kubelet binary, which is responsible for running Pods on every Kubernetes node:

```
sudo rm /home/kubernetes/bin/kubelet && sudo systemctl restart kubelet
logout
```

Now when we check the node status we see the node is NotReady.

```
watch kubectl get nodes
```


```
NAME                                          STATUS     ROLES     AGE       VERSION
gke-gke-workshop-new-pool-42b33f8c-9grf   Ready      <none>    6m        v1.9.6-gke.1
gke-gke-workshop-new-pool-847e18a1-f1bp   Ready      <none>    6m        v1.9.6-gke.1
gke-gke-workshop-new-pool-8c4b26e9-fq8p   NotReady   <none>    6m        v1.9.6-gke.1
```


The Kubernetes Engine node repair agent will wait a few minutes in case the problem is intermittent. We'll come back to this in a minute.

Define a maintenance window
You can configure a maintenance window to have more control over when automatic upgrades are applied to Kubernetes on your cluster.

Creating a maintenance window instructs Kubernetes Engine to automatically trigger any automated tasks in your clusters, such as master upgrades, node pool upgrades, and maintenance of internal components, during a specific timeframe.

The times are specified in UTC, so select an appropriate time and set up a maintenance window for your cluster.

Open the cloud console and navigate to Kubernetes Engine. Click on the gke-workshop cluster and click the edit button at the top. Find the Maintenance Window option and select 3AM. Finally, click Save to update the cluster.

Enable node auto-upgrades
Whenever a new version of Kubernetes is released, Google upgrades your master to that version. You can then choose to upgrade your nodes to that version, bringing functionality and security updates to both the OS and the Kubernetes components.

Node Auto-Upgrades use the same update mechanism as manual node upgrades, but does the scheduled upgrades during your maintenance window.

Auto-upgrades are enabled per node pool.

```
gcloud container node-pools update new-pool --cluster $ORG-$PRODUCT-$ENV-cluster   --enable-autoupgrade
```



This will enable the autoupgrade feature for nodes. Please see
https://cloud.google.com/kubernetes-engine/docs/node-management for more
information on node autoupgrades.


```
ERROR: (gcloud.container.node-pools.update) ResponseError: code=400, message=Operation operation-1511896056410-dbda7f9f is currently operating on cluster gke-workshop. Please wait and try again once it's done.
```

That's good - that means our broken node is being repaired! Try again in a few minutes.

Check your node repair
How is that node repair coming?

After a few minutes, you will see that the master drained the node, and then removed it.

```
watch kubectl get nodes
```

```
NotReady,SchedulingDisabled
```

A few minutes after that, a new node was turned on in its place:


```
Ready
```

```
kubectl get pods
```


```
kubectl delete deployment hello-web
``` -->

