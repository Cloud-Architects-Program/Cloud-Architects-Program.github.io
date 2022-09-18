# 1 Deploy Applications on Kubernetes

**Objective:**

  * Review process of creating NameSpaces
  * Review process of changing Context
  * Review process of creating K8s:
    * Services
    * Labels, Selectors
    * Deployments
    * Rolling Updates



##  Prepare the Cloud Source Repository Environment with Module 6 Assignment

This lab can be executed in you GCP Cloud Environment using Google Cloud Shell.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")


Once opened, you can use it to run the instructions for this lab.

Cloud Source Repositories: Qwik Start


**Step 1** Locate directory where kubernetes `YAML` manifest going to be stored.

```
cd ~/ycit019_2022/
git pull       # Pull latest Mod8_assignment
```

In case you don't have this folder clone it as following:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit019_2022
cd ~/ycit019_2022/Mod8_assignment/
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


**Step 3** Copy `Mod8_assignment` folder to your repo:

```
git pull                              # Pull latest code from you repo
cp -r ~/ycit019_2022/Mod8_assignment/ .
```

**Step 4** Commit `Mod8_assignment` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding `Mod8_assignment` with kubernetes YAML manifest"
```

**Step 5** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

**Step 6** Review Cloud Source Repositories


Use the `Google Cloud Source Repositories` code browser to view repository files. 
You can filter your view to focus on a specific branch, tag, or comment.

Browse the Mod8_assignment files you pushed to the repository by opening the Navigation menu and selecting Source Repositories:

```
Click Menu -> Source Repositories > Source Code.
```

!!! result
    The console shows the files in the master branch at the most recent commit.



## 1.1 Development Tools

!!! note
    This steps is optional

**Step 1** Choose one of the following option to develop the YAML manifests:

*Option 1:* You can develop in [Google Cloud Shell Editor](https://shell.cloud.google.com/)

*Option 2:* You can also develop locally on your laptop using [VCScode](https://code.visualstudio.com/). We recommend to use it in conjunction with VSC YAML extension from [Redhat](https://t.co/D5r4HZZdUC?amp=1)

*Option 4:* Use your preferred text editor on Linux VM (vim, nano).

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


**Step 1** Locate folder with Kubernetes Manifests:

```
cd ~/$student_name-notepad/Mod8_assignment/deploy
ls
```

**Output:** `gowebapp-deployment.yaml  gowebapp-mysql-deployment.yaml  gowebapp-mysql-service.yaml  gowebapp-service.yaml`

!!! result
    You can see 4 manifest with `Deployment` and `Service` manifests for `gowebapp` and  `gowebapp-mysql`

**Step 6** Define a Kubernetes Service object for the backend MySQL database.


Follow instructions below to populate  `gowebapp-mysql-service.yaml`


**Reference K8s Docs:** 

1. [Service](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service)


Additionally, you can use `kubectl` built-in docs for any type of resources:

```
kubectl explain service
```

```
edit gowebapp-mysql-service.yaml
```

!!! note 
    You can also use VCS or Cloud Code to work with `yaml` manifest.

```
#TODO: Specify Kubernetes API apiVersion
#TODO: Identify the kind of Object
metadata:
 #TODO: Give the service a name: "gowebapp-mysql"
  labels:
    #TODO: Add a label KV "run: gowebapp-mysql"
spec:
  #TODO: leave the clusterIP to None. We allow k8s to assign clusterIP.
  ports:
    #TODO: Define a "port" as 3306
    #TODO: Define a "targetPort" as 3306
  #TODO: Add a selector for our pods as label "run" with value "gowebapp-mysql"
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

**Reference K8s Docs:** 

1. [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment)
2. [Updating Env variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)

```
edit gowebapp-mysql-deployment.yaml
```
```
apiVersion: apps/v1
#TODO: Identify the type of Object
metadata:
  #TODO: Give the Deployment a name "gowebapp-mysql"
  labels:
  #TODO: Add a label KV "run: gowebapp-mysql"
  #TODO: give the Deployment a label: tier: backend
spec:
  #TODO: Define number of replicas, set it to 1
  #TODO: Starting from Deplloyment v1 selectors are mandatory
    #add selector KV "run: gowebapp-mysql"
  strategy:
    type: # Set strategy type as `Recreate`
  template:
    metadata:
      #TODO: Add a label called "run" with the name of the service: "gowebapp-mysql"
    spec:
      containers:
      - env:
         - #TODO: define name as MYSQL_ROOT_PASSWORD 
           #TODO: define value as rootpasswd
         image: #TODO define mysql image created in previous assignment, located in gcr registry
         name: gowebapp-mysql
         ports:
         #TODO: define containerPort: 3306
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
edit gowebapp-service.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: gowebapp
  labels:
    #TODO: give the Service a label: run: gowebapp
    #TODO: give the Service a label: tier: frontend
spec:
  #TODO: Define a "ports" array with the "port" attribute: 9000 and "targetPort" attributes: 80
  #TODO: Add a selector for our pods as label "run" with value "gowebapp"
  #TODO: Add a Service Type of LoadBalancer
  #If you need help, see reference: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
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

**Reference K8s Docs:** 

1. [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment)
2. [Updating Env variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)


```
edit gowebapp-deployment.yaml
```

```
apiVersion: apps/v1
#TODO: define the kind of object as Deployment
metadata:
  #TODO: Add a name attribute for the service as "gowebapp"
  labels:
    #TODO: give the Deployment a label: run: gowebapp
    #TODO: give the Deployment a label: tier: frontend
spec:
  #TODO: Define number of replicas, set it to 2
  #TODO: add selector KV "run: gowebapp"
  template:
    metadata:
      labels:
        run: gowebapp
        tier: frontend
    spec:
      containers:
      - env:
        - #TODO: define name as MYSQL_ROOT_PASSWORD
          #TODO: define value as rootpasswd
        image: #TODO define gowebapp image created in previous assignment, located in gcr registry
        name: gowebapp
        ports:
        - #TODO: define the container port as 80
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

edit ~/$student_name-notepad/Mod8_assignment/gowebapp/code/template/partial/footer.tmpl
```

**Step 2** Build a new version of Image

```
cd ~/$student_name-notepad/Mod8_assignment/gowebapp
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
cd  ~/$student_name-notepad/Mod8_assignment/deploy
```

**Step 2** Trigger rolling upgrade using `kubectl set` command

```
#TO DO
```

**Step 3** Verify rollout history

```
#TO DO
```

**Step 4** Perform Rollback to v1

```
#TO DO
```

## 2.8 Commit K8s manifests to repository and share it with Instructor/Teacher


**Step 1** Commit `deploy` folder using the following Git commands:

```
cd ~/$student_name-notepad/
```

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
