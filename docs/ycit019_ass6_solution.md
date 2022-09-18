# Deploy Applications on Kubernetes

**Objective:**

  * Review process of creating K8s:
    * Network Policy
    * Ingress
    * PVC, PV


## 1. Prepare Environment


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
--enable-network-policy \
--num-nodes 2
```

**Output:**
```
NAME          LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
k8s-scaling  us-central1-c  1.19.9-gke.1400  34.121.222.83  e2-medium     1.19.9-gke.1400  2          RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-networking --zone us-central1-c
``` 


### 1.2 Locate Assignment 6

**Step 1** Locate directory where Kubernetes manifests going to be stored.

```
cd ~/ycit019/
git pull       # Pull latest assignement6
```

In case you don't have this folder clone it as following:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit019
cd ~/ycit019/Assignment6/
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

**Step 3** Copy Assignment 6 `deploy_a6` folder to your repo:

```
cp -r ~/ycit019/Assignment6/deploy_a6 .
```

**Step 4** Commit `deploy` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding K8s manifests for assignment 6"
```

**Step 5** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

### 1.3 Create a Namespace `dev`

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



## 2 Configure Volumes

### 2.1 Create Persistent Volume for `gowebapp-mysql`


**Step 1** Edit a Persistent Volume Claim (PVC) `gowebapp-mysql-pvc.yaml` manifest so that it will Dynamically creates 5G GCP PD Persistent Volume (PV), using  `balanced` persistent disk Provisioner with Access Mode `ReadWriteOnce`.



```
cd ~/$MY_REPO/deploy_a6
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
#TODO Access Mode `ReadWriteOnce`
#TODO Request 5G GCP PD Persistent Storage
#TODO Balanced PD CSI Storage Class 
#TODO GKE CSI Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/gce-pd-csi-driver
  storageClassName: standard-rwo
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Please see reference for PVC creation here:
https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims


Save the file!


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


### 2.2 Create Mysql deployment

Create Mysql deployment using created Persistent Volume (PV)

**Step 1**  Update `gowebapp-mysql-deployment.yaml` and add Volume


```
cd ~/$MY_REPO/deploy_a6
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


Save the file!

**Step 2** Deploy `gowebapp-mysql` app under `~/$MY_REPO/deploy_a6/`

```
cd ~/$MY_REPO/deploy_a6/
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
kubectl get describe $POD
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
cd ~/$MY_REPO/gowebapp/config/
kubectl create configmap gowebapp --from-file=webapp-config-json=config.json
kubectl describe configmap gowebapp
```

**Step 2** Deploy `gowebapp` app under `~/$MY_REPO/deploy_a6/`

```
cd ~/$MY_REPO/deploy_a6/
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


**Step 5** Access `Loadbalancer` IP via browser:

```
EXTERNAL-IP:9000
```

!!! result
    Congrats!!! You've deployed you code to Kubernetes


## 3 Ingress

### 3.1 Expose `gowebapp` with Ingress resource
**Step 1** Delete the LoadBalancer gowebapp service.

```
kubectl delete svc gowebapp
```

**Step 2** Modify gowebapp service manifest `gowebapp-service.yaml` to change the service to a ClusterIP service instead of LoadBalancer service.


```
cd ~/$MY_REPO/deploy_a6/
edit gowebapp-service.yaml
```

!!! result 
    Cloud Shell Editor opens. Make sure update service and save it!

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
  type: NodePort
```

Go back to  Cloud Shell Terminal.


```
kubectl apply -f gowebapp-service.yaml  #Re-Create Service with ClusterIP
```


**Step 3** Create an Ingress resource for the gowebapp service to expose it externally at the path `/*`

```
cat > gowebapp-ingress.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gowebapp-ingress
spec:
  rules:
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: gowebapp
            port:
              number: 9000
EOF
```


**Step 4** Deploy `gowebapp-ingress` app under ~/$MY_REPO/deploy_a6/

```
kubectl apply -f gowebapp-ingress.yaml   #Create Ingress
```

!!! note
    When you create Ingress first time it takes several minutes, as Ingress Resource will create backing Loadbalancer for it.

**Step 5** Verify the Ingress

After you have deployed a workload, grouped its Pods into a Service, and created an Ingress for the Service, you should verify that the Ingress has provisioned the container-native load balancer successfully.


```
kubectl describe ingress gowebapp-ingress
```


**Step 6**  Test load balancer functionality

!!! note
    Wait several minutes for the HTTP(S) load balancer to be configured.

Get the Ingress IP address, run the following command:

```
kubectl get ing gowebapp-ingress
```

In the command output, the Ingress' IP address is displayed in the ADDRESS column. Visit the IP address in a web browser

```
$ADDRESS/gowebapp
```



## 4 Kubernetes Network Policy


Let's secure our 2 tier application using Kubernetes Network Policy!

**Task:** We need to ensure following is configured

  *  `dev` namespace Denys by default All `Ingress` traffic including from Outside, With In Namespace, and in Cluster
  * `mysql` pod can be accessed from the `gowebapp` pod
  * `gowebapp` pod can be accessed from the Internet, in our case GKE Ingress (HTTPs Loadbalancer) and it must be configure to allow the appropriate HTTP(S) load balancer health check IP ranges FROM CIDR: `35.191.0.0/16` and `130.211.0.0/22` or
  to make it less secure From any Endpoint CIDR: `0.0.0.0/0`
  

### Prerequisite
In order to ensure that Kubernetes can configure Network Policy we need to make sure our CNI
supports this feature.  As we've learned from class Calico, WeaveNet and Cilium are CNIs that support
network Policy.

Our GKE cluster already deployed using Calico CNI. To check that you calico installed you cluster run:

```
kubectl get pods -n kube-system | grep calico
```

**Output** 

```
calico-node-2n65c                                           1/1     Running   0          20h
calico-node-prhgk                                           1/1     Running   0          20h
calico-node-vertical-autoscaler-57bfddcfd8-85ltc            1/1     Running   0          20h
calico-typha-67c885c7b7-7vf6d                               1/1     Running   0          20h
calico-typha-67c885c7b7-zwkll                               1/1     Running   0          20h
calico-typha-horizontal-autoscaler-58b8486cd4-pnt7c         1/1     Running   0          20h
calico-typha-vertical-autoscaler-7595df8859-zxzmp           1/1     Running   0          20h
```



### 4.1 Configure `deny-all` Network Policy for dev Namespace

**Step 1** Locate Directory
```
cd ~/$MY_REPO/deploy_a6/
```

**Step 1** Create Policy so that  `dev` namespace deny by default all `Ingress` traffic including from Outside Internet, With In Namespace, and inside of Cluster

```
cat > default-deny-dev.yaml << EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
    - Ingress
  ingress: []
EOF
```


Edit the required fields and save:

```
edit default-deny-dev.yaml 
```

**Step 2** Deploy `default-deny-dev` Policy app under ~/$MY_REPO/deploy_a6/

```
kubectl apply -f default-deny-dev.yaml   # deny-all Ingress Traffic to dev Namespace
```

Verify Policy:

```
kubectl describe netpol default-deny
```

**Step 3** Verify that you can't access application from Ingress anymore:

Visit the IP address in a web browser

```
$Ingress IP
```

### 4.2 Configure Network Policy for `gowebapp-mysql`


**Step 1** Create a `backend-policy` network policy to allow access to `gowebapp-mysql` pod from the `gowebapp` pod


```
cd ~/$MY_REPO/deploy_a6/
```


```
cat > gowebapp-mysql-netpol.yaml << EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      run: gowebapp-mysql
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: gowebapp
EOF
```

Edit the required fields and save:

```
edit gowebapp-mysql-netpol.yaml
```

**Step 2** Using [Cilium Editor](https://editor.cilium.io/) test you Policy for Ingress traffic only

Open Browser, Upload created Policy YAML. 

!!! result
    `mysql` pod can only communicate with `gowebapp` pod



**Step 3** Deploy `gowebapp-mysql-netpol` Policy app under ~/$MY_REPO/deploy_a6/

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


**Step 1**  Configure a `frontend-policy` Network Policy to allow access for the Healthcheck IP ranges needed for the Ingress Loadbalancer, and hence allow access through the Ingress Loadbalancer. The IP rangers you need to enable access from are `35.191.0.0/16` and `130.211.0.0/22`



```
cd ~/$MY_REPO/deploy_a6/
```


```
cat > gowebapp-netpol.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector: 
    matchLabels:
      run: gowebapp
  ingress:
  policyTypes:
    - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 35.191.0.0/16
    - ipBlock:
        cidr: 130.211.0.0/22
  policyTypes:
  - Ingress
EOF
```
Edit the required fields and save:

```
edit gowebapp-netpol.yaml
```


**Step 2** Using [Cilium Editor](https://editor.cilium.io/) test you Policy for Ingress traffic only

Open Browser, Upload created Policy YAML. 

!!! result
    `gowebapp` pod can only get ingress traffic from CIDRs: `35.191.0.0/16` and `130.211.0.0/22`



**Step 3** Deploy `gowebapp-netpol` Policy app under ~/$MY_REPO/deploy_a6/

```
kubectl apply -f gowebapp-netpol.yaml  # `gowebapp` pod from Internet on CIDR: `35.191.0.0/16` and `130.211.0.0/22`
```

Verify Policy:

```
kubectl describe netpol frontend-policy
```

!!! result
    `run=gowebapp`   Allowing ingress traffic from CIDR: 35.191.0.0/16 and 130.211.0.0/22



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
gcloud container clusters delete k8s-scaling
```
