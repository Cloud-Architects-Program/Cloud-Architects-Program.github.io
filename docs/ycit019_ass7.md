# Deploy Applications on Kubernetes

**Objective:**

  * Review process of creating K8s:
    * Network Policy
    * Ingress
    * PVC, PV


##  Prepare the Cloud Source Repository Environment with Module 10 and 11 Assignments

This lab can be executed in you GCP Cloud Environment using Google Cloud Shell.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")


Once opened, you can use it to run the instructions for this lab.

Cloud Source Repositories: Qwick Start


**Step 1** Locate directory where kubernetes `YAML` manifest going to be stored.

```
cd ~/ycit019_2022/
git pull       # Pull latest Mod10_assignment
```

In case you don't have this folder clone it as following:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit019_2022
cd ~/ycit019_2022/Mod10_assignment/
ls
```


**Step 2** Go into the local repository you've created:

```
export student_name=<write_your_name_here_and_remove_brakets>
```

!!! important
    Replace above with your project_id student_name

```
cd ~/$student_name-notepad
```


**Step 3** Copy `Mod10_assignment` folder to your repo:

```
git pull                              # Pull latest code from you repo
cp -r ~/ycit019_2022/Mod10_assignment/ .
```

**Step 4** Commit `Mod10_assignment` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding `Mod10_assignment` with kubernetes YAML manifest"
```

**Step 5** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

**Step 6** Review Cloud Source Repositories


Use the `Google Cloud Source Repositories` code browser to view repository files. 
You can filter your view to focus on a specific branch, tag, or comment.

Browse the Mod10_assignment files you pushed to the repository by opening the Navigation menu and selecting Source Repositories:

```
Click Menu -> Source Repositories > Source Code.
```

!!! result
    The console shows the files in the master branch at the most recent commit.


### 1.1 Create GKE Cluster with Cluster Network Policy Support
**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:
```
gcloud container clusters create k8s-networking \
--zone us-central1-c \
--enable-ip-alias \
--create-subnetwork="" \
--network=default \
--enable-dataplane-v2 \
--num-nodes 2
```

!!! note
    Dataplane v2 CNI on GKE is Google's integration of Cilium Networking based on EBPF.
    If you prefer using Calico CNI you need to replace 

**Output:**
```
NAME: k8s-networking
LOCATION: us-central1-c
MASTER_VERSION: 1.22.8-gke.202
MASTER_IP: 34.66.47.138
MACHINE_TYPE: e2-medium
NODE_VERSION: 1.22.8-gke.202
NUM_NODES: 2
STATUS: RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-networking --zone us-central1-c
``` 


## 2 Configure Volumes

### 2.1 Create Persistent Volume for `gowebapp-mysql`


**Step 1** Edit a Persistent Volume Claim (PVC) `gowebapp-mysql-pvc.yaml` manifest so that it will Dynamically creates 5G GCP PD Persistent Volume (PV), using  `balanced` persistent disk Provisioner with Access Mode `ReadWriteOnce`.


```
cd ~/$student_name-notepad/Mod10_assignment/deploy
edit gowebapp-mysql-pvc.yaml
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gowebapp-mysql-pvc
  labels:
    run: gowebapp-mysql
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Please see reference for PVC creation here:
https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims


**Step 2** Apply the manifest for `gowebapp-mysql-pvc` to Dynamically provision Persistent Volume:

```
kubectl apply -f gowebapp-mysql-pvc.yaml
```

**Step 3** Verify  `STATUS` of PVC

```
kubectl get pvc
```

**Output:**
```
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS 
gowebapp-mysql-pvc   Bound     pvc-xxx  5G         RWO            standard   
```

List PVs:
```
kubectl get pv
```

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS
pvc-xxxx  5Gi        RWO            Delete           Bound    default/gowebapp-mysql-pvc   standard 
```

List GCP Disks:

```
gcloud compute disks list
```


### 2.2 Create Mysql deployment

Create Mysql deployment using created Persistent Volume (PV)

**Step 1**  Update `gowebapp-mysql-deployment.yaml` and add Volume


```
cd ~/$student_name-notepad/Mod10_assignment/deploy
edit gowebapp-mysql-deployment.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gowebapp-mysql
  labels:
    run: gowebapp-mysql
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      run: gowebapp-mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        run: gowebapp-mysql
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql
              key: password
        image: gcr.io/${PROJECT_ID}/gowebapp-mysql:v1
        name: gowebapp-mysql
        ports:
        - containerPort: 3306
        livenessProbe:
          tcpSocket: 
            port: 3306
          initialDelaySeconds: 30
          timeoutSeconds: 2
        readinessProbe: 
          tcpSocket:
            port: 3306
          initialDelaySeconds: 25
          timeoutSeconds: 2
        resources:
          requests:
            cpu: 20m
            memory: 252M
          limits:
            cpu: 1000m
            memory: 2G
        #TODO add the definition for volumeMounts:
        volumeMounts:
        - #TODO: add mountPath as /var/lib/mysql
        - mountPath: /var/lib/mysql
          name: mysql
      #TODO Configure Pods access to storage by using the claim as a volume
      #TODO: define persistentVolumeClaim
      #TODO: claimName is the name defined in gowebapp-mysql-pvc.yaml
      #TODO Ref: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes
      volumes:
        - name: mysql
          persistentVolumeClaim:
            claimName: gowebapp-mysql-pvc
```

**Step 2** Deploy `gowebapp-mysql` app under `~/$MY_REPO/deploy/`

```
cd ~/$student_name-notepad/Mod10_assignment/deploy/
kubectl apply -f secret-mysql.yaml              #Create Secret
kubectl apply -f gowebapp-mysql-service.yaml    #Create Service
kubectl apply -f gowebapp-mysql-deployment.yaml #Create Deployment
```

Check Status of the `mysql` Pod:

```
kubectl get pods
```

Check Events from the `mysql` Pod:

```
kubectl describe $POD_NAME_FROM_PREVIOUS_COMMAND
```

!!! note
    Notice the sequence of events:

    * Scheduler selects node
    * Volume mounted to Pod
    * `kubelet` pulls image
    * `kubelet` creates and starts container


!!! result
    Our Mysql Deployment is up and running with Volume mounted


### 2.4 Create GoWebApp deployment

**Step 1:** Create ConfigMap for gowebapp's config.json file

```
cd ~/$student_name-notepad/Mod10_assignment/gowebapp/config/
kubectl create configmap gowebapp --from-file=webapp-config-json=config.json
kubectl describe configmap gowebapp
```

**Step 2** Deploy `gowebapp` app under `~/$MY_REPO/deploy/`

```
cd ~/$student_name-notepad/Mod10_assignment/deploy/
kubectl apply -f gowebapp-service.yaml    #Create Service
kubectl apply -f gowebapp-deployment.yaml #Create Deployment
```

```
kubectl get pods
```


**Step 3** Access your application on Public IP via automatically created Loadbalancer 
created for `gowebapp` service.

To get the value of `Loadbalancer` run following command:

```
kubectl get svc gowebapp -o wide
```

Expected output:
```
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
gowebapp         Loadbalancer 10.107.15.39  XXXXXX        9000:32634/TCP   30m
gowebapp-mysql   ClusterIP   None           <none>        3306/TCP         1h
```


**Step 4** Access `Loadbalancer` IP via browser:

```
EXTERNAL-IP:9000
```

!!! result
    Congrats!!! You've deployed you code to Kubernetes


## 3.1 Ingress

### 3.1 Expose `gowebapp` with Ingress resource
**Step 1** Delete the LoadBalancer gowebapp service.

```
#TODO delete the gowebapp service
```

**Step 2** Modify gowebapp service manifest `gowebapp-service.yaml` to change the service to a NodePort service instead of LoadBalancer service.

```
cd ~/$student_name-notepad/Mod10_assignment/deploy/
edit gowebapp-service.yaml
```

!!! result 
    Cloud Shell Editor opens. Make sure to update the service and save it!


```
apiVersion: v1
kind: Service
metadata:
  name: gowebapp
  labels:
    run: gowebapp
spec:
  ports:
  - port: 9000
    targetPort: 80
  selector:
    run: gowebapp
  #TODO add the appropriate type
```


Go back to  Cloud Shell Terminal.

```
kubectl apply -f gowebapp-service.yaml   #Re-Create the service
```

**Step 3** Update an Ingress resource for the `gowebapp` service to expose it externally at the path /*

```
edit gowebapp-ingress.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gowebapp-ingress
spec:
  rules:
  - http:
      paths:
      - path: #TODO add the correct path
        pathType: #TODO add the pathType appropriate for a GKE Ingress
        backend:
          service:
            name: #TODO add the appropriate service name
            port:
              number: #TODO add the appropriate service port
```


**Step 4** Deploy `gowebapp-ingress` app under ~/$MY_REPO/deploy/

```
kubectl apply -f gowebapp-ingress.yaml   #Create Ingress
```


**Step 5** Verify the Ingress

After you have deployed a workload, grouped its Pods into a Service, and created an Ingress for the Service, you should verify that the Ingress has provisioned the container-native load balancer successfully.


```
kubectl describe ingress gowebapp-ingress
```


**Step 6**  Test that application is reachable via `Ingress`

!!! note
    Wait several minutes for the HTTP(S) load balancer to be configured.

Get the Ingress IP address, run the following command:

```
kubectl get ing gowebapp-ingress
```

In the command output, the Ingress' IP address is displayed in the ADDRESS column. Visit the IP address in a web browser

```
$ADDRESS/*
```

## 4 Kubernetes Network Policy

Let's secure our 2 tier application using Kubernetes Network Policy!

**Task:** We need to ensure the following is configured

  *  Your namespace deny by default all `Ingress` traffic including from outside the cluster (Internet), With-in the namespace, and from inside the Cluster
  * `mysql` pod can be accessed from the `gowebapp` pod
  * `gowebapp` pod can be accessed from the Internet, in our case GKE Ingress (HTTPs Loadbalancer) and it must be configure to allow the appropriate HTTP(S) load balancer health check IP ranges FROM CIDR: `35.191.0.0/16` and `130.211.0.0/22` or
  to make it less secure From any Endpoint CIDR: `0.0.0.0/0`
  

### Prerequisite
In order to ensure that Kubernetes can configure Network Policy we need to make sure our CNI supports this feature.  As we've learned from class Calico, WeaveNet and Cilium are CNIs that support network Policy.

Our GKE cluster is already deployed using Cilium CNI. To check that calico is installed in your cluster, run:

```
 kubectl -n kube-system get ds -l k8s-app=cilium -o wide
```

**Output** 

```
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE   CONTAINERS     IMAGES                                  
anetd   2         2         2       2            2           kubernetes.io/os=linux   45m   cilium-agent   gke.gcr.io/cilium/cilium:v1.11.1-gke3.3
```

!!! note
    `anetd` is a Cilium agent DaemonSet that runs on each node.

!!! result
    Our GKE cluster running `cilium:v1.11.1` which Cilium version 1.11.1


### 4.1 Configure `deny-all` Network Policy inside the Namespace


```
cd ~/$student_name-notepad/Mod10_assignment/deploy/
```

**Step 1** Create a policy so that you current namespace deny by default all `Ingress` traffic including from the Internet, with-in the namespace, and from inside the Cluster


```
edit default-deny.yaml
```

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: #TODO add the appropriate match labels to block all traffic
  policyTypes:
    #TODO Default deny all ingress traffic https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-ingress-traffic
```

Edit the required fields and save


**Step 2** Deploy `default-deny` Policy app under ~/$MY_REPO/deploy/

```
kubectl apply -f default-deny.yaml   # deny-all Ingress Traffic inside Namespace
```

Verify Policy:

```
kubectl describe netpol default-deny
```


!!! result
    <none> - blocking the specific traffic to all pods in this namespace

**Step 3** Verify that you can't access the application through Ingress anymore:

Visit the IP address in a web browser

```
$Ingress IP
```

!!! result
    can't access the app through ingress loadbalancer anymore.

### 4.2 Configure Network Policy for `gowebapp-mysql`

**Step 1** Create a `backend-policy` network policy to allow access to `gowebapp-mysql` pod from the `gowebapp` pods

```
cd ~/$student_name-notepad/Mod10_assignment/deploy/
```

```
edit gowebapp-mysql-netpol.yaml
```

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      #TODO add the appropriate match labels
  ingress:
    - from:
      - podSelector:
          matchLabels:
            #TODO add the appropriate match labels
```

Edit the required fields and save:

```
edit gowebapp-mysql-netpol.yaml
```


**Step 2** Using [Cilium Editor](https://editor.cilium.io/) test you Policy for Ingress traffic only

Open Browser, Upload created Policy YAML. 

!!! result
    `mysql` pod can only communicate with `gowebapp` pod

**Step 3** Deploy `gowebapp-mysql-netpol` Policy app under ~/$MY_REPO/deploy/

```
kubectl apply -f gowebapp-mysql-netpol.yaml   # `mysql` pod can be accessed from the `gowebapp` pod
```

Verify Policy:

```
kubectl describe netpol backend-policy
```

!!! result
    `run=gowebapp-mysql` Allowing ingress traffic only From PodSelector: `run=gowebapp` pod

### 4.3 Configure Network Policy for `gowebapp`

**Step 1** Configure a Network Policy to allow access for the Healthcheck IP ranges needed for the Ingress Loadbalancer, and hence allow access through the Ingress Loadbalancer. The IP rangers you need to enable access from are `35.191.0.0/16` and `130.211.0.0/22`


```
cd ~/$student_name-notepad/Mod10_assignment/deploy/
```

```
edit gowebapp-netpol.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector: 
    matchLabels:
      run: gowebapp
  ingress:
  #TODO add the appropriate rules to enable access from IP ranges
  policyTypes:
  - Ingress
```


Edit the required fields and save:

```
edit gowebapp-netpol.yaml
```


**Step 2** Using [Cilium Editor](https://editor.cilium.io/) test you Policy for Ingress traffic only

Open Browser, Upload created Policy YAML. 

!!! result
    `gowebapp` pod can only get ingress traffic from CIDRs: `35.191.0.0/16` and `130.211.0.0/22`


**Step 3** Deploy `gowebapp-netpol` Policy app under ~/$MY_REPO/deploy/

```
kubectl apply -f gowebapp-netpol.yaml  # `gowebapp` pod from Internet on CIDR: `35.191.0.0/16` and `130.211.0.0/22`
```

Verify Policy:

```
kubectl describe netpol frontend-policy
```

!!! result
    `run=gowebapp` Allowing ingress traffic from CIDR: 35.191.0.0/16 and 130.211.0.0/22



**Step 4**  Test that application is reachable via `Ingress` after all Network Policy has been applied.


Get the Ingress IP address, run the following command:

```
kubectl get ing gowebapp-ingress
```

In the command output, the Ingress' IP address is displayed in the ADDRESS column. Visit the IP address in a web browser

```
$ADDRESS/*
```


**Step 5** Verify that `NotePad` application is functional (e.g can login and create new entries)

## 5 Commit K8s manifests to repository and share it with Instructor/Teacher

**Step 1** Commit `deploy` folder using the following Git commands:

```
git add .
git commit -m "k8s manifests for Hands-on Assignment 6"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```

## 6 Cleaning Up

**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-networking
```
