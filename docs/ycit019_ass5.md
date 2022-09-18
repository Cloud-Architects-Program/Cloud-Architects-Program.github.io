# Deploy Applications on Kubernetes

**Objective:**

  * Review process of creating K8s:
    * Resource
    * Limits
    * HPA
    * VPA
    * Limit Range

## 0 Create GKE Cluster with Cluster and Vertical Autoscaling Support
**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:
```
gcloud container clusters create k8s-scaling \
--zone us-central1-c \
--enable-vertical-pod-autoscaling \
--num-nodes 2 \
--enable-autoscaling --min-nodes 1 --max-nodes 3
```


**Output:**
```
NAME          LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
k8s-scaling  us-central1-c  1.19.9-gke.1400  34.121.222.83  e2-medium     1.19.9-gke.1400  2          RUNNING
```

**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-scaling --zone us-central1-c
``` 


## 1.2 Locate Assignment 5

**Step 1** Locate directory where Kubernetes manifests going to be stored.

```
cd ~/ycit019/
git pull       # Pull latest assignement5
```

In case you don't have this folder clone it as following:

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit019
cd ~/ycit019/Assignment5/
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

**Step 3** Copy Assignment 5 `deploy_a5` folder to your repo:

```
cp -r ~/ycit019/Assignment5/deploy_a5 .
```

**Step 4** Commit `deploy` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding K8s manifests for assignment 5"
```

**Step 5** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

## 1.3 Create a Namespace `dev`

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


## 2.1 Configure VPA for `gowebapp-mysql` to find optimal Resource Request and Limits values

`requests` and `limits` is the way Kubernetes set's QoS for Pods, as well as enable's features like HPA, CA, Resource Quota's and more.

However setting best values for resource `requests` and `limits` is hard, VPA is here to help. Set VPA for `gowebapp` and observe usage recommendation for `requests` and `limits`

**Step 1** Deploy `gowebapp-mysql` app under `~/$MY_REPO/deploy_a5/`

```
cd ~/$MY_REPO/deploy_a5/
kubectl apply -f secret-mysql.yaml              #Create Secret
kubectl apply -f gowebapp-mysql-service.yaml    #Create Service
kubectl apply -f gowebapp-mysql-deployment.yaml #Create Deployment
```

```
kubectl get deploy
```

!!! result
    Our Deployment is up, however without `request` and `limits` it will be treated as Best Effort QoS resource on the Cluster.

**Step 2** Edit a  manifest for `gowebapp-mysql` Vertical Pod Autoscaler resource:

```
cd ~/$MY_REPO/deploy_a5
vim gowebapp-mysql-vpa.yaml
```

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: gowebapp-mysql
#TODO: Configure VPA with updateMode:OFF
```

**Step 3** Apply the manifest for `gowebapp-mysql-vpa`
```
kubectl apply -f gowebapp-mysql-vpa.yaml
```

**Step 4** Wait a minute, and then view the VerticalPodAutoscaler

```
kubectl describe vpa gowebapp-mysql
```

!!! note
    If you don't see it, wait a little longer and try the previous command again. 


**Step 5** Locate the "Container Recommendations" at the end of the output from the `describe` command.

!!! result
    We will be using `Lower Bound` values to set our `request` value and `Upper Bound` as our `limits` value.



## 2.2 Set Recommended Request and Limits values to `gowebapp-mysql`

**Step 1** Edit a manifest for `gowebapp` deployment resource:

```
cd ~/$MY_REPO/deploy_a5
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
#TODO define a resource request and limits based on VPA Recommender
```


**Step 2** Redeploy our application with defined resource `request` and `limits`

```
cd ~/$MY_REPO/deploy_a5/
kubectl delete -f gowebapp-mysql-deployment.yaml
kubectl apply -f gowebapp-mysql-deployment.yaml
```


## 2.3 Configure VPA for `gowebapp` to find optimal Resource Request and Limits values


**Step 1:** Create ConfigMap for gowebapp's config.json file

```
cd ~/$MY_REPO/gowebapp/config/
kubectl create configmap gowebapp --from-file=webapp-config-json=config.json
kubectl describe configmap gowebapp
```

**Step 2** Deploy `gowebapp` app under `~/$MY_REPO/deploy_a5/`

```
cd ~/$MY_REPO/deploy_a5/
kubectl apply -f gowebapp-service.yaml    #Create Service
kubectl apply -f gowebapp-deployment.yaml #Create Deployment
```

```
kubectl get deploy
```

!!! result
    Our Deployment is up, however without `request` and `limits` it will be treated as Best Effort QoS resource on the Cluster.

**Step 3** Edit a  manifest for `gowebapp` Vertical Pod Autoscaler resource:

```
cd ~/$MY_REPO/deploy_a5
vim gowebapp-vpa.yaml
```

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: gowebapp
#TODO: Ref: https://cloud.google.com/kubernetes-engine/docs/how-to/vertical-pod-autoscaling
#TODO: Configure VPA with updateMode:OFF
```

**Step 4** Apply the manifest for `gowebapp-vpa`
```
kubectl apply -f gowebapp-vpa.yaml
```

**Step 5** Wait a minute, and then view the VerticalPodAutoscaler

```
kubectl describe vpa gowebapp
```

!!! note
    If you don't see it, wait a little longer and try the previous command again. 


**Step 6** Locate the "Container Recommendations" at the end of the output from the `describe` command.

!!! result
    We will be using `Lower Bound` values to set our `request` value and `Upper Bound` as our `limits` value.


## 2.4 Set Recommended Request and Limits values to `gowebapp`

**Step 1** Edit a manifest for `gowebapp` deployment resource:

```
cd ~/$MY_REPO/deploy_a5
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
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql
              key: password
        image: gcr.io/${PROJECT_ID}/gowebapp:v3
        name: gowebapp
        ports:
          - containerPort: 80
        livenessProbe:
          httpGet:
            path: /register
            port: 80
          initialDelaySeconds: 15
          timeoutSeconds: 5
        readinessProbe:
          httpGet: 
            path: /register
            port: 80
          initialDelaySeconds: 25
          timeoutSeconds: 5
        resources:
        #TODO define a resource request and limits based on VPA Recommender
        volumeMounts:
          - name: config-volume
            mountPath: /go/src/gowebapp/config
      volumes: 
      - name: config-volume
        configMap:
          name: gowebapp
          items:
          - key: webapp-config-json
            path: config.json
```


**Step 2** Redeploy our application with defined resource `request` and `limits`

```
cd ~/$MY_REPO/deploy_a5/
kubectl delete -f gowebapp-deployment.yaml
kubectl apply -f gowebapp-deployment.yaml
```


## 2.5 Configure HPA for `gowebapp`

Our NotePad Application is going to Production soon. To make sure our application can scale based on requests we will set HPA for our deployment resource using Horizontal Pod Autoscaler.


**Step 1** Create HPA for `gowebapp` based on CPU with minReplicas `1` and maxReplicas `5` with target 50.

```
cd ~/$MY_REPO/deploy_a5
vim gowebapp-hpa.yaml
```

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: gowebapp-hpa
spec:
  scaleTargetRef:
  #TODO: Ref: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
  #TODO Create HPA based on CPU with minReplicas `1` and maxReplicas `5` with target 50.
```


**Step 2** Apply the manifest for `gowebapp-hpa`

```
kubectl apply -f gowebapp-hpa.yaml
```

**Step 3** Take a closer look at the HPA and observe autoscaling or downscaling if any.

```
kubectl describe hpa kubia
```

!!! note
    It will take some time to collect metrics information about current cpu usage and since our does't have real load it might not trigger any scaling


## 2.6 Configure LimitRanges for Namespace `dev`

In order to prevent developers accidentally forget to set values for  `request` and `limits`.
Ops team decide to create a Configuration `LimitRange`, that will enforce some default values for  `request` and `limits` if they have not been set, as well as `Minimum` and `maximum` requests/limits a container can have to prevent resources abuse.


**Step 1** Edit a  manifest for `limit-range` that can be used for `dev` namespace with following requirements:


```
cd ~/$MY_REPO/deploy_a5/
vim limit-range.yaml
```

```
apiVersion: v1
kind: LimitRange
metadata:
  name: gowebapp-system
  #TODO Specify Namespace
spec:
#TODO Ref: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/
#TODO If not specified the Container's `memory` limit is set to 256Mi, which is the default memory limit for the namespace.
#TODO If not specified default limit `cpu` is 250m per container
#TODO If not specified set the Container's Default memory requests to 256Mi
#TODO If not specified set the Container's Default cpu requests to 250m
#TODO Configure Minimum `20m` and Maximum `800m` CPU Constraints for a Namespace
```

## 2.7 Viewing cluster autoscaler events

Our cluster was configured to use Cluster Autoscaler, verify if during you assignment, cluster did went through autoscaling process?

To view the logs, perform the following:

**Step 1:** In the Cloud Console, go to the Logs Viewer page.

**Step 2:** Search for the logs using the basic or advanced query interface.

To search for logs using the basic query interface, perform the following:

  * a. From the resources drop-down list, select Kubernetes Cluster, then select the location of your cluster, and the name of your cluster.
  * b. From the logs type drop-down list, select container.googleapis.com/cluster-autoscaler-visibility.
  * c. From the time-range drop-down list, select the desired time range.

OR search for logs using the advanced query interface, apply the following advanced filter:

```
resource.type="k8s_cluster"
resource.labels.location="cluster-location"
resource.labels.cluster_name="cluster-name"
logName="projects/project-id/logs/container.googleapis.com%2Fcluster-autoscaler-visibility"
```

Reference [link](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-autoscaler-visibility#viewing_events)


## 2.7 Commit K8s manifests to repository and share it with Instructor/Teacher


**Step 1** Commit `deploy` folder using the following Git commands:

```
git add .
git commit -m "k8s manifests for Hands-on Assignment 5"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```

## 3.6 Cleaning Up

**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-scaling
```
