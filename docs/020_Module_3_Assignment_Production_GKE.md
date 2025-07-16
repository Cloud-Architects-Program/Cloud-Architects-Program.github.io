Assignment: Creating Production GKE Cluster

**Objective:**

* GKE Regional, Private Standard Cluster for Production Deployment
* GKE Autopilot Cluster 

  
## 1 Creating Production GKE Cluster

In Part 1 of the Assignment we going to deploy `GKE` Standard cluster for production usage. This cluster will have `Regional` control plain for high availability.
Also following production security Best Practices we going to setup `GKE` with `private` nodes (No Public IP) and learn how to setup `Cloud Nat` which allows Private cluster's Pods access securely Internet.

### 1.1 Locate Assignment 2

**Step 1**  Clone `ycit020` repo with Kubernetes manifests, which going to use for our work:


```
cd ~/ycit020_2022/
git pull       # Pull latest Mod3_assignment
```

In case you don't have this folder clone it as following:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit020_2022
cd ~/ycit020_2022/module3_assignment/
ls
```


!!! result 
    You can see Kubernetes manifests 


**Step 2** Go into your personal Google Cloud Source Repository:


```
export student_name=<write_your_name_here_and_remove_brakets>
```

!!! note
    Replace $student_id with your ID

```
cd ~/$student_name-notepad
```

```
git pull                              # Pull latest code from you repo
```

**Step 3** Copy `module3_assignment` folder to your repo:

```
cp -r ~/ycit020_2022/module3_assignment ycit020_module3
```


**Step 4** Create `production_gke.txt` file that will have instructions how to deploy Production GKE Clusters.

```
cd ~/$student_name-notepad/ycit020_module3
cat > production_gke.txt << EOF
# Creating Production GKE Cluster

**Step 1** Create a vpc network:

**Step 2** Create a subnet VPC design:

**Step 3** Create a subnet with primary and secondary ranges:

**Step 4** Create a Private GKE Cluster:

**Step 5** Create a Cloud Router:

**Step 6** Create a Cloud Nat Gateway:

**Step 7** Create an Autopilot GKE Cluster:

EOF
```

!!! important
    This document will be graded.

**Step 5** Commit `ycit020_module3` folder using the following Git commands:

```
cd ~/$student_name-notepad
git status 
git add .
git commit -m "adding documentation for ycit020 module 3 assignment"
```

**Step 6** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

### 1.2 Creating a GCP project


!!! note
    We are going to create a temporary project for this assignment.
    While you going to store code in existing project that we've used so far in class

Set variables:

```
export ORG=$student_name
```

```
export PRODUCT=notepad
export ENV=gcloud
export PROJECT_PREFIX=1   # Project has to be unique
export PROJECT_ID=$ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```

Create a project:

```
gcloud projects create $ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```


List billings and take note of `ACCOUNT_ID`:

```
ACCOUNT_ID=$(gcloud beta billing accounts list | grep -B2 True  | head -1 | grep ACCOUNT_ID  |awk '{ print $2}') 
```

Check account ID was set properly as variable:

```
echo $ACCOUNT_ID
```

Attach your project to the correct billing account:

```
gcloud beta billing projects link $PROJECT_ID --billing-account=$ACCOUNT_ID
```


Set the newly created project as default for gcloud:

```
gcloud config set project $PROJECT_ID
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


```
gcloud config set compute/region us-central1
```

### 1.2 Deleting Default VPC and associated Firewall Rules to it

Observe Asset Inventory in GCP UI:

```
Products -> IAM & Admin -> Asset Inventory -> Overview
```

Select resource type `compute.Subnetwork`


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
VPC Network->Firewall
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

**Step 1:** Create a new custom mode VPC network:


with name `$ORG-$PRODUCT-$ENV-vpc`, with subnet mode `custom`,
and `regional`.

**Step 1:** Create a new custom mode VPC network:

Set variables:


```
export ORG=$student_name
```

```
export PRODUCT=notepad
export ENV=gcloud
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

Document a command to create `vpc` network in `production_gke.txt` doc under `step 1`.

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```

Save and go to next step.

!!! important
    If `production_gke.txt` is not updated with commands this will reduce score for you assignment

**Step 2:** Create firewall rules  `default-allow-ssh`:

```
gcloud compute firewall-rules create $ORG-$PRODUCT-$ENV-allow-tcp-ssh-icmp --network $ORG-$PRODUCT-$ENV-vpc --allow tcp:22,tcp:3389,icmp
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
  
  * 2 GKE Clusters per Project
  
  * Max 200 `Nodes` per cluster and *primary range* belongs to Class A and starting from (10.130.0.0/x) 
  
  * Max 110 `Pods` per node and *pod secondary ranges*  belongs to Class A and starting from (10.0.0.0/x) 
  
  * Max 500 `Services` per cluster and *service secondary ranges*  belongs to Class A and starting from (10.100.0.0/x)
  
  * CIDR range for master-ipv4-cidr (k8s api range) belongs to Class B (172.16.0.0/x)


Use following reference docs:

  1. [VPC-native clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)
  2. [GKE address management](https://cloud.google.com/architecture/gke-address-management-options) 


Using tables from [VPC-native clusters document](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips) and online subnet [calculator](https://www.subnet-calculator.com/), create a table for `dev`, `stg`, `prd` in the following format and store result under `production_gke.txt`:

```
Project   | Subnet Name |     subnet          |    pod range     |    srv range   | kubectl api range
app 1 Dev | dev-1       |     (10.130.0.0/x)  |     IP range     |     IP range   |     IP range    
          | dev-2       |     (10.130.0.0/x)  |     IP range     |     IP range   |     IP range      
```

Document a subnet VPC design in `production_gke.txt` doc under `step 2`.

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```

Save and go to next step.

**Task N3:** Create a `user-managed` subnet.

Create a subnet for `dev` cluster, taking in consideration VPC subnet ranges created in above table, where:

Subnet N1 for `GKE Standard Cluster`:

  * Subnet name: `gke-standard-$ORG-$PRODUCT-$ENV-subnet`

  * Region: `us-central1`
  
  * Node Range: See column `subnet` in above table for `dev-1` cluster
  
  * Secondary Ranges:
  
    * Service range name: `gke-standard-services`
  
    * Service range CIDR: See column `srv range` in above table for `dev-1` cluster
  
    * Pods range name: `gke-standard-pods`
  
    * Pods range CIDR: See column `pod range` in above table for `dev-1` cluster
  
  * Features:
    * Flow Logs
    * Private IP Google Access


Subnet N2 for `GKE Autopilot`:
  
  * Subnet name: `gke-auto-$ORG-$PRODUCT-$ENV-subnet`
  * Region: `us-central1`
  * Node Range: See column `subnet` in above table for `dev-2` cluster
  * Secondary Ranges:
    * Service range name: `gke-auto-services`
    * Service range CIDR: See column `srv range` in above table for `dev-2` cluster
    * Pods range name: `gke-auto-pods`
    * Pods range CIDR: See column `pod range` in above table for `dev-2` cluster
  * Features:
    * Flow Logs
    * Private IP Google Access




```
export ORG=$student_name
```
```
export PRODUCT=notepad
export ENV=gcloud
```

```
TODO: gcloud compute networks subnets create gke-standard-$ORG-$PRODUCT-$ENV-subnet

TODO: gcloud compute networks subnets create gke-auto-$ORG-$PRODUCT-$ENV-subnet
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


Document a subnet VPC creation command in `production_gke.txt` doc under `step 3`.

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```

Save and go to next step.


### 1.5 Creating a Private, Regional and VPC Native GKE Cluster


**Task N4:** Create GKE Standard Cluster with  *Private* Nodes and following values:

  * Cluster name: `$ORG-$PRODUCT-$ENV-standard-cluster`
  * VPC `network` name: `$ORG-$PRODUCT-$ENV-vpc`
  * VPC `subnet` name: `gke-standard-$ORG-$PRODUCT-$ENV-subnet`
  * Secondary `pod` range with name: `gke-standard-pods`
  * Secondary `service` range with name: `gke-standard-services`
  * VM size: `e2-micro`
  * Node count 1 per zone: `num-nodes`
  * GKE Control plane is replicated across three zones of a region: `us-central1`
  * GKE Release channel: `regular`
  * Enable Cilium based Networking DataplaneV2: `enable-dataplane-v2`
  * GKE version: "1.22.12-gke.300"
  * Cluster Node Communication `VPC Native`: `enable-ip-alias`
  * Cluster Nodes access: *Private* Node GKE Cluster with *Public API* endpoint
  * For `master-ipv4-cidr` ranges: See column `kubectl api range` in above table for `dev-1` cluster



Use following reference docs:

  1. [Creating a private cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters)
  2. [GKE Release Channels](https://cloud.google.com/kubernetes-engine/docs/concepts/release-channels#channels)
  3. Use `gcloud container clusters create` command [reference](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create) for all available options.



```
export ORG=$student_name
```
```
export PRODUCT=notepad
export ENV=gcloud
```

```
#TODO gcloud clusters create $ORG-$PRODUCT-$ENV-cluster
```

Document a GKE cluster creation command in `production_gke.txt` doc under `step 4`.

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```

Save and go to next step.



!!! result 
    The private cluster is now created. `gcloud` has set up `kubectl` to authenticate with the private cluster but the cluster's K8s API server will only accept connections from the primary range of your subnetwork, and the secondary range of your subnetwork that is used for pods. That means nobody can access K8s API at this point of time, until we specify allowed ranges.



**Step 3** Authenticate to the cluster.

```
gcloud container clusters get-credentials $ORG-$PRODUCT-$ENV-standard-cluster --region us-central1 
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

Suppose you have a group of machines, outside of your VPC network, belongs to your organization. You could authorize ONLY those machines to access the public endpoint (Kubernetes API).


Here we will enable master authorized networks and whitelist the IP address for our Cloud Shell instance, to allow access to the master:

```
gcloud container clusters update $ORG-$PRODUCT-$ENV-standard-cluster --enable-master-authorized-networks --master-authorized-networks $(curl ipinfo.io/ip)/32 --region us-central1
```

Now we can access the API server using `kubectl` from GCP console:


!!! note
    In real life it could be CIDR Range for you company, so only engineers or CI/CD systems from you company can connect to `kubectl` apis in secure manner.


Now we can access the API server using kubectl:

```
kubectl get pods
```

**Output:**

```
No resources found in default namespace.
```

Let's deploy basic application on the GKE Standard Cluster:

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
wget: bad address 'www.example.com'
```

!!! results
    Pods can't access internet, because it's a private cluster


!!! note
    Normally, if cluster is Private and doesn't have Cloud Nat configured, users
    can't even deploy images from Docker Hub. See troubleshooting details [here](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#docker_hub)


Delete test pod:

```
kubectl delete pods wget
```

### 1.6 Create a Google Cloud Nat

Private clusters give you the ability to isolate nodes from having inbound and outbound connectivity to the public internet. This isolation is achieved as the nodes have internal IP addresses only.

If you want to provide outbound internet access for certain private nodes, you can use [Cloud NAT](https://cloud.google.com/nat/docs/overview#NATwithGKE)



Cloud NAT is a distributed, software-defined managed service. It's not based on proxy VMs or appliances.


You configure a NAT gateway on a Cloud Router, which provides the control plane for NAT, holding configuration parameters that you specify. Google Cloud runs and maintains processes on the physical machines that run your Google Cloud VMs.

Cloud NAT can be configured to automatically scale the number of NAT IP addresses that it uses, and it supports VMs that belong to managed instance groups, including those with autoscaling enabled.


**Step 1:** Create a Cloud NAT configuration using Cloud Router

Create Cloud Router in the same region as the instances that use Cloud NAT. Cloud NAT is only used to place NAT information onto the VMs. It is not used as part of the actual NAT gateway.


**Task N5:** Create `Cloud Router`


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

Document `Cloud Router` creation  command in `production_gke.txt` doc under `step 5`

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```


**Task N6:** Create `Cloud Nat` Gateway

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



Document Cloud Nat creation  command in `production_gke.txt` doc under `step 6`:

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```

Save and go to next step.



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


###  1.7 Delete Default Node Pool and Create Custom Node Pool

A Kubernetes Engine cluster consists of a master and nodes. Kubernetes doesn't handle provisioning of nodes, so Google Kubernetes Engine handles this for you with a concept called *node pools*.

A node pool is a subset of node instances within a cluster that all have the same configuration. They map to instance templates in Google Compute Engine, which provides the VMs used by the cluster. By default a Kubernetes Engine cluster has a single node pool, but you can add or remove them as you wish to change the shape of your cluster.


In the previous example, you've created a Kubernetes Engine cluster. This gave us three nodes (three e2-micro (2 vCPUs, 1 GB memory), 100 GB of disk each) in a single node pool (called default-pool). Let's inspect the node pool:

```
gcloud config set compute/region us-central1
gcloud container node-pools list --cluster $ORG-$PRODUCT-$ENV-standard-cluster
```

**Output:**

```
NAME: default-pool
MACHINE_TYPE: e2-micro
DISK_SIZE_GB: 100
NODE_VERSION: 1.22.12-gke.300
```

If you want to add more nodes of this type, you can grow this node pool. If you want to add more nodes of a different type, you can add other node pools.

A common method of moving a cluster to larger nodes is to add a new node pool, move the work from the old nodes to the new, and delete the old node pool.

Let's add a second node pool. This time we will use the larger `e2-medium`	(2 vCPUs, 4 GB memory), 100 GB of disk each machine type.


**Task N7:** Create Create Custom Node Pool and Delete Default Node Pool 

```
gcloud container node-pools create custom-pool --cluster $ORG-$PRODUCT-$ENV-standard-cluster \
    --machine-type e2-medium --num-nodes 1
```


**Output:**
```
NAME: custom-pool
MACHINE_TYPE: e2-medium
DISK_SIZE_GB: 100
NODE_VERSION: 1.22.12-gke.300
```

```
gcloud container node-pools list --cluster $ORG-$PRODUCT-$ENV-standard-cluster
```
**Output:**

```
NAME: default-pool
MACHINE_TYPE: e2-micro
DISK_SIZE_GB: 100
NODE_VERSION: 1.22.12-gke.300

NAME: custom-pool
MACHINE_TYPE: e2-medium
DISK_SIZE_GB: 100
NODE_VERSION: 1.22.12-gke.300
```

You can now delete the original `default-pool` node pool:

```
gcloud container node-pools delete default-pool --cluster $ORG-$PRODUCT-$ENV-standard-cluster
```

Check that node-pool has been deleted:

```
gcloud container node-pools list --cluster $ORG-$PRODUCT-$ENV-standard-cluster
```

!!! result
    GKE cluster running only on `custom-pool` Node pool.


## 2 Creating GKE Autopilot Clusters

In Part 2 of the Assignment we going to deploy GKE Autopilot Clusters for Production Usage.
Autopilot Clusters provides easy way to configure and manage GKE Clusters.
They configured with best practices and  high availability out of the box. 
And can additionally configured with Private Clusters. 


**Task N8:** Create GKE Autopilot Cluster with following subnet ranges:

  * Cluster name: `$ORG-$PRODUCT-$ENV-auto-cluster`
  * In `region`: `us-central1`
  * VPC `network` name: `$ORG-$PRODUCT-$ENV-vpc`
  * VPC `subnet` name: `gke-auto-$ORG-$PRODUCT-$ENV-subnet`
  * Secondary `pod` range with name: `gke-auto-pods`
  * Secondary `service` range with name: `gke-auto-services`

Use following reference docs:

  1. [Creating a GKE Autopilot cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-an-autopilot-cluster#gcloud)



```
export ORG=$student_name
```
```
export PRODUCT=notepad
export ENV=gcloud
```


```
#TODO gcloud container clusters create-auto $ORG-$PRODUCT-$ENV-auto-cluster
```


!!! result 
    Production ready GKE Autopilot Cluster has been created

Document a GKE Autopilot cluster creation command in `production_gke.txt` doc under `step 8`.

```
cd ~/$student_name-notepad/ycit020_module3
edit production_gke.txt
```

Save and go to next step.

Authenticate to GKE Autopilot cluster:

```
gcloud container clusters get-credentials $ORG-$PRODUCT-$ENV-auto-cluster --region us-central1 
``` 

GKE Autopilot starts with 2 small Nodes, that is not charged to Customers:

```
kubectl get nodes
```

```
gk3-archy-notepad-gcloud-default-pool-5c17f5f3-lp4p   Ready    <none>   11m   v1.22.12-gke.300
gk3-archy-notepad-gcloud-default-pool-fae5625e-cwxx   Ready    <none>   11m   v1.22.12-gke.300

```

This Nodes are used to run GKE system containers such as various agents and networking pods:

```
kubectl get pods -o wide --all-namespaces
```

!!! note
    With GKE Autopilot, you don't need to worry about nodes or nodepools at all. Autopilot will spin nodes up and down based on the resources needed by your deployments, but you will only be charged for the resources requested by your actual deployments.


Let's deploy basic application on the GKE Standard Cluster:

```
kubectl run hello-web --image=gcr.io/google-samples/hello-app:1.0 --port=8080
```


Let's verify that Pod deployed:
```
kubectl get pods
```

**Output:**
```
NAME        READY   STATUS    RESTARTS   AGE
hello-web   0/1     Pending   0          9s
```

After about 1 munute you should see:
```
NAME        READY   STATUS              RESTARTS   AGE
hello-web   0/1     ContainerCreating   0          101s
```

!!! note
    That the application takes longer than usual to start. This is because Autopilot NAP system starting off new Nodes for application.



Let's verify that Pod deployment after few minutes:

```
kubectl get pods
```

**Output:**
```
NAME        READY   STATUS    RESTARTS   AGE
hello-web   1/1     Running   0          7s
```

Let's verify that Nodes deployed:

```
kubectl get nodes
```

**Output:**
```
NAME                                                  STATUS   ROLES    AGE     VERSION
gk3-archy-notepad-gcloud-default-pool-5c17f5f3-lp4p   Ready    <none>   24m     v1.22.12-gke.300
gk3-archy-notepad-gcloud-default-pool-5c17f5f3-t89j   Ready    <none>   5m35s   v1.22.12-gke.300
gk3-archy-notepad-gcloud-default-pool-fae5625e-cwxx   Ready    <none>   24m     v1.22.12-gke.300
```

!!! result
    You will see 3d node has been created. GKE Autopilot created a Node based on workload requirement


!!! note
    If you don't specify resource requests in your deployment spec, then Autopilot will set `CPU` to 2000m and `Memory` to 4gb. If your app requires less resources than that, make sure you set the resource request for your deployment. The minimum CPU is 250m and the minimum memory is 512mb.




## 3 Commit Terraform configurations to repository and share it with Instructor/Teacher

**Step 1** Commit `ycit020_module3` folder using the following Git commands:


```
cd ~/$student_name-notepad
```

```
git add .
git commit -m "Gcloud Documentation for Module 3 Assignment"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```


!!! result
    Your instructor will be able to review your code and grade it.


## 4 Cleanup

```
export ORG=$student_name
```

```
export PRODUCT=notepad
export ENV=gcloud
export PROJECT_PREFIX=1   # Project has to be unique
export PROJECT_ID=$ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```

Delete GKE standard Cluster:

```
gcloud container clusters delete $ORG-$PRODUCT-$ENV-standard-cluster
```

Delete GKE Autopilot Cluster:

```
gcloud container clusters delete $ORG-$PRODUCT-$ENV-auto-cluster
```


!!! important
    We are going to delete the `$ORG-$PRODUCT-$ENV-$PROJECT_PREFIX` only,
    please do not delete your project with Source Code Repo.

```
gcloud projects delete $ORG-$PRODUCT-$ENV-$PROJECT_PREFIX
```
