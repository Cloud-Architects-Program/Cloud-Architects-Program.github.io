# 1 Deploy Applications on Kubernetes

**Objective:**

  * Review process of creating NameSpaces
  * Review process of changing Context
  * Review process of creating K8s:
    * Services
    * Labels, Selectors
    * Deployments
    * Rolling Updates

## 1.1 Development Tools

!!! note
    This steps is optional

**Step 1** Choose one of the following option to develop the YAML manifests:

*Option 1:* You can develop in [Google Cloud Shell Editor](https://shell.cloud.google.com/)

*Option 2:* You can also develop locally on your laptop using [VCScode](https://code.visualstudio.com/). We recommend to use it in conjunction with VSC YAML extension from [Redhat](https://t.co/D5r4HZZdUC?amp=1)

*Option 3:* Use your preferred text editor on Linux VM (vim, nano).

## 2.1 Create GKE Cluster

**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with two nodes:
```
gcloud container clusters create k8s-concepts \
--zone us-central1-c \
--num-nodes 2
```

**Output:**
```
NAME          LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
k8s-concepts  us-central1-c  1.19.9-gke.1400  34.121.222.83  e2-medium     1.19.9-gke.1400  2          RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-concepts --zone us-central1-c
``` 


## 2.2 Setup KUBECTL AUTOCOMPLETE

Since we going to use a lot of kubectl cli let's setup autocomplete.

```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```



## 2.3 Create 'dev' namespace and make it default.

**Step 1** Create 'dev' namespace that's going to be used to develop and deploy `Notestaker` application on Kubernetes using `kubetl` CLI.

```
kubectl create ns dev
```

**Step 2** Use `dev` context to create K8s resources inside this namespace.

```
kubectl config set-context --current --namespace=dev
```

**Step 3** Verify current context:

```
kubectl config view | grep namespace
```

!!! result
    `dev`


## 2.4 Create Service Object for MySQL

**Step 1** Locate directory where kubernetes manifests going to be stored.

```
cd ~/ycit019/
git pull       # Pull latest assignement3
```

In case you don't have this folder clone it as following:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit019
cd ~/ycit019/Assignment3/
ls
```


**Step 2** Go into the local repository you've created:

```
cd ~/$MY_REPO
```

**Step 3** Copy Assignment 3 `deploy` folder to your repo:

```
git pull                              # Pull latest code from you repo
cp -r ~/ycit019/Assignment3/deploy .
```

**Step 4** Commit `deploy` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding K8s manifests in deploy folder"
```

**Step 5** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

**Step 6** Define a Kubernetes Service object for the backend MySQL database.

```
cd ~/$MY_REPO/deploy
```

Follow instructions below to populate  `gowebapp-mysql-service.yaml`

For reference, please see `Service` [docs](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service
):

```
https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service
```

Additionally, you can use `kubectl` built-in docs for any type of resources:

```
kubectl explain service
```

```
vim gowebapp-mysql-service.yaml
```

!!! note 
    You can also use VCS or Cloud Code to work with `yaml` manifest.

```
apiVersion: v1
kind: Service
metadata:
  name: gowebapp-mysql
  labels:
    run: gowebapp-mysql
spec:
  clusterIP: None
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    run: gowebapp-mysql
```

**Step 3** Create a Service object for MySQL

```
kubectl apply -f gowebapp-mysql-service.yaml --record
```

**Step 4** Check to make sure it worked
```
kubectl get service -l "run=gowebapp-mysql"
```

## 2.5 Create Deployment object for the backend MySQL database


**Step 1** Follow instructions below to populate `gowebapp-mysql-deployment.yaml`

For reference, please see `Deployment` doc:

```
https://kubernetes.io/docs/concepts/workloads/controllers/deployment
```

```
vim gowebapp-mysql-deployment.yaml
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
          value: rootpasswd
        image: gcr.io/${PROJECT_ID}/gowebapp-mysql:v1
        name: gowebapp-mysql
        ports:
        - containerPort: 3306
```

**Step 2** Create a Deployment object for MySQL

```
kubectl apply -f gowebapp-mysql-deployment.yaml --record
```

**Step 3** Check to make sure it worked

```
kubectl get deployment -l "run=gowebapp-mysql"
```

**Step 3** Check `mysql` pod logs:

List mysql Pods and note the name the `pod`:
```
kubectl get pods -l "run=gowebapp-mysql"
```

Ensure `Mysql` is up by looking at `pod` logs:

```
kubectl logs <Pod_name>
```

!!! result
    We have created `Service` and `Deployment` for backend application.

## 2.5 Create a K8s Service for the frontend gowebapp.

**Step 1** Follow instructions below to populate `gowebapp-service.yaml`

```
vim gowebapp-service.yaml
```

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
  type: LoadBalancer
```

**Step 2** Create a Service object for gowebapp

```
kubectl apply -f gowebapp-service.yaml --record
```

**Step 3** Check to make sure it worked

```
kubectl get service -l "run=gowebapp"
```

## 2.6 Create a K8s Deployment object for the frontend gowebapp

**Step 1** Follow instructions below to populate `gowebapp-deployment.yaml`

```
vim gowebapp-deployment.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gowebapp
  labels:
    run: gowebapp
spec:
  replicas: 2
  selector:
    matchLabels:
      run: gowebapp
  template:
    metadata:
      labels:
        run: gowebapp
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpasswd
        image: gcr.io/${PROJECT_ID}/gowebapp:v1
        name: gowebapp
        ports:
        - containerPort: 80
```


**Step 2** Create a Deployment object for gowebapp

```
kubectl apply -f gowebapp-deployment.yaml --record
```

**Step 3** Check to make sure it worked
```
kubectl get deployment -l "run=gowebapp"
```

**Step 4** Access your application on Public IP via automatically created Loadbalancer 
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

!!! result
    Congrats!!! You've deployed you code to Kubernetes


## 2.7 Fix gowebapp code bugs and build a new image.

Task:
As you've noticed `gowebapp` frontend app has `YCIT019` logo in it.
Since you may want to use application for you personal needs, let's change
`YCIT019` logo to you `Name`.

**Step 1** Modify gowebapp frontend so that it has name of you company and link
to company web page e.g.

```

vim ~/$MY_REPO/gowebapp/code/template/partial/footer.tmpl
```

**Step 2** Build a new version of Image


```
cd ~/$MY_REPO/gowebapp
docker build -t gcr.io/${PROJECT_ID}/gowebapp:v2 .
docker push gcr.io/${PROJECT_ID}/gowebapp:v2 .
```

## 2.7 Rolling Upgrade

For `gowebapp` frontend deployment manifest we've not specified
any upgrade strategy type. It means application will use default
Upgrade strategy called `RollingUpdate`.

`RollingUpdate` strategy - updates Pods in a rolling update fashion.


`maxUnavailable` - is an optional field that specifies the maximum number of Pods that can be unavailable during the update process. By default, it ensures that at least 25% less than the desired number of Pods are up (25% max unavailable).

`Max Surge` - is an optional field that specifies the maximum number of Pods that can be created over the desired number of Pods. By default, it ensures that at most 25% more than the desired number of Pods are up (25% max surge).

**Step 1** Locate directory with manifest

```
cd  ~/$MY_REPO/deploy
```

**Step 2** Trigger rolling upgrade using `kubectl set` command

```
kubectl set image deployments/gowebapp gowebapp=gcr.io/${PROJECT_ID}/gowebapp:v2
```

**Step 3** Verify rollout history

```
kubectl rollout history deployment/gowebapp
```

**Step 4** Perform Rollback to v1

```
kubectl rollout undo deployment/gowebapp --to-revision=1
```

## 2.8 Commit K8s manifests to repository and share it with Instructor/Teacher


**Step 1** Commit `deploy` folder using the following Git commands:

```
git add .
git commit -m "k8s manifests"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```

## 2.9 Cleaning Up
**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-concepts
```
