# 1 Kubernetes Autoscaling


**Objective**

* Resource & Limits
* Scheduling
* HPA
* VPA
* Cluster Autoscaling
* Node Auto provisioning (NAP)


## 0 Create GKE Cluster
**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:
```
gcloud container clusters create k8s-scaling \
--zone us-central1-c \
--enable-vertical-pod-autoscaling \
--num-nodes 2
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


## 1.1 Resource and Limits

**Step 1:** Inspecting a node’s capacity

```
kubectl describe nodes | grep -A15  Capacity:
```

The output shows two sets of amounts related to the available resources on the node: the node’s capacity and allocatable resources. The capacity represents the total resources of a node, which may not all be available to pods. Certain resources may be reserved for Kubernetes and/or system components. The Scheduler bases its decisions only on the allocatable resource amounts.

**Step 2:** Show metrics for a given node

```
kubectl top nodes
kubectl top pods -n kube-system
```

!!! result
    CPU and Memory information is available for pods and node through the metrics API.


**Step 3** Create a `deployment` `best_effort.yaml` as showed below.
This is regular deployment with  `resources` configured
```
cat <<EOF > best_effort.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  selector:
    matchLabels:
      app: kubia
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
EOF
```

**Step 4** Deploy application
```
kubectl create -f best_effort.yaml
```

**Step 5** Verify what is the QOS for this pod:

```
kubectl describe pods  | grep QoS
```

!!! result
    If you don't specify request/limits K8s provides `Best Effort` QOS

**Step 6** Cleanup

```
kubectl delete -f best_effort.yaml
```


**Step 7** Create a `deployment` `guaranteed.yaml` as showed below.
This is regular deployment with  `resources` configured
```
cat <<EOF > guaranteed.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  selector:
    matchLabels:
      app: kubia
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 100m
            memory: 200Mi
EOF
```

**Step 8** Deploy application
```
kubectl create -f guaranteed.yaml
```

**Step 9** Verify what is the QOS for this pod:

```
kubectl describe pods  | grep QoS
```

!!! result
    If you request = limits K8s provides `guaranteed` QOS

**Step 10** Cleanup

```
kubectl delete -f guaranteed.yaml
```


**Step 11** Create a `deployment` `burstable.yaml` as showed below.
This is regular deployment with  `resources` configured
```
cat <<EOF > burstable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  selector:
    matchLabels:
      app: kubia
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 3000
EOF
```


**Step 12** Deploy application

```
kubectl create -f burstable.yaml
```



**Step 13** Verify what is the QOS for this pod:

```
kubectl describe pods  | grep QoS
```

!!! result
    If you specify `request > or < limits`  K8s provides `Burstable` QOS


**Step 14**  Check status of the Pods 

```
kubectl get pods
```

!!! Pending
    Why the deployment failed ???


**Step 15** Cleanup

```
kubectl delete -f burstable.yaml
```


## 1.2 Creating a Horizontal Pod Autoscaler based on CPU usage

Prerequisites: Ensure metrics api is running in your cluster.

```
kubectl get pod -n kube-system
```

Check the status of `metrics-server-*****` pod status. It should be `Running`

```
kubectl top nodes
kubectl top pods -n kube-system
```

!!! result
    CPU and Memory information is available for pods and node through the metrics API.


Let’s create a horizontal pod autoscaler now and configure it to scale pods
based on their CPU utilization.


**Step 1** Create a `deployment.yaml` as showed below.
This is regular deployment with  `resources` configured
```
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  selector:
    matchLabels:
      app: kubia
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 100m
EOF
```

**Step 2** Deploy application
```
kubectl create -f deployment.yaml
```


**Step 3** After creating the deployment, to enable horizontal autoscaling of its pods, you need to create a HorizontalPodAutoscaler (HPA) object and point it to the deployment.

```
kubectl autoscale deployment kubia --cpu-percent=30 --min=1 --max=5
```

!!! note
    This creates the HPA object for us and sets the deployment called `kubia` as the scaling target. We’re setting the target CPU utilization of the pods to 30% and specifying the minimum and maximum number of replicas. The autoscaler will thus constantly keep adjusting the number of replicas to keep their CPU utilization around 30%, but it will never scale down to less than 1 or scale up to more than 5 replicas.



**Step 4** Verify definition of the Horizontal Pod Autoscaler resource to gain a better understanding of it:

```
kubectl get hpa kubia -o yaml
```

**Result:**

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
...
spec:
  maxReplicas: 5
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kubia
  targetCPUUtilizationPercentage: 30
status:
  currentReplicas: 0
  desiredReplicas: 0
```
**Step 5** Take a closer look at the HPA and notice that it is still not ready to do the autoscaling.
```
kubectl describe hpa kubia
```
**Results**
```
Events:
  Type     Reason                        Age                   From                       Message
  ----     ------                        ----                  ----                       -------
  Warning  FailedGetResourceMetric       2m29s                 horizontal-pod-autoscaler  unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Warning  FailedComputeMetricsReplicas  2m29s                 horizontal-pod-autoscaler  failed to compute desired number of replicas based on listed metrics for Deployment/default/kubia: invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Warning  FailedGetResourceMetric       118s (x3 over 2m14s)  horizontal-pod-autoscaler  did not receive metrics for any ready pods
  Warning  FailedComputeMetricsReplicas  118s (x3 over 2m14s)  horizontal-pod-autoscaler  failed to compute desired number of replicas based on listed metrics for Deployment/default/kubia: invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: did not receive metrics for any ready pods
```
Given that historical data is not available yet, you will see the above in the events section.

Give it a minute or so and try again. Eventually, you will see the following in the `Events` section.
```
  Normal   SuccessfulRescale             41s                    horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```
If you take a look at the `kubia` deployment, you will see it was scaled down from 3 pods to 1 pod.

**Step 6** Create a service
```
kubectl expose deployment kubia --port=80 --target-port=8080
```

**Step 7** Start another terminal session and run:
```
watch -n 1 kubectl get hpa,deployment
```

**Step 8** Generate load to the Application

```
kubectl run -it --rm --restart=Never loadgenerator --image=busybox \
-- sh -c "while true; do wget -O - -q http://kubia.default; done"
```
**Step 9** Observe autoscaling
In the other terminal you will start noticing that the deployment is being scaled up.

**Step 10** Terminate both sessions by pressing `Ctrl+c`

## 1.3 Scale size of pods with Vertical Pod Autoscaling
**Step 1** Verify that Vertical Pod Autoscaling has already been enabled on the cluster. We enabled VPA when we created the cluster, by using `--enable-vertical-pod-autoscaling`. This command can be handy if you want to check VPA on an existing cluster.
```
gcloud container clusters describe k8s-scaling --zone us-central1-c | grep ^verticalPodAutoscaling -A 1
```

**Step 2** Apply the hello-server deployment to your cluster
```
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0
```
**Step 3** Ensure the deployment was successfully created
```
kubectl get deployment hello-server
```
**Step 4** Assign a CPU resource request of 100m to the deployment
```
kubectl set resources deployment hello-server --requests=cpu=100m
```
**Step 5** Inspect the container specifics of the `hello-server` pods, find `Requests` section, and notice that this pod is currently requesting the 450m CPU we assigned.
```
kubectl describe pod hello-server | sed -n "/Containers:$/,/Conditions:/p"
```

**Output**
```
Containers:
  hello-app:
    Image:      gcr.io/google-samples/hello-app:2.0
    Port:       <none>
    Host Port:  <none>
    Requests:
      cpu:        100m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-rw2gr (ro)
Conditions:
Containers:
  hello-app:
    Container ID:   containerd://e9bb428186f5d6a6572e81a5c0a9c37118fd2855f22173aa791d8429f35169a6
    Image:          gcr.io/google-samples/hello-app:2.0
    Image ID:       gcr.io/google-samples/hello-app@sha256:37e5287945774f27b418ce567cd77f4bbc9ef44a1bcd1a2312369f31f9cce567
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 09 Jun 2021 11:34:15 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-rw2gr (ro)
Conditions:
```
**Step 6** Create a manifest for you Vertical Pod Autoscale
```
cat << EOF > hello-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hello-server-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       hello-server
  updatePolicy:
    updateMode: "Off"
EOF
```
**Step 7** Apply the manifest for `hello-vpa`
```
kubectl apply -f hello-vpa.yaml
```
**Step 8** Wait a minute, and then view the VerticalPodAutoscaler
```
kubectl describe vpa hello-server-vpa
```
**Step 9** Locate the "Container Recommendations" at the end of the output from the `describe` command. If you don't see it, wait a little longer and try the previous command again. When it appears, you'll see several different recommendation types, each with values for CPU and memory:

  * Lower Bound: this is the lower bound number VPA looks at for triggering a resize. If your pod utilization goes below this, VPA will delete the pod and scale it down.
  * Target: this is the value VPA will use when resizing the pod.
  * Uncapped Target: if no minimum or maximum capacity is assigned to the VPA, this will be the target utilization for VPA.
  * Upper Bound: this is the upper bound number VPA looks at for triggering a resize. If your pod utilization goes above this, VPA will delete the pod and scale it up.

Notice that the VPA is recommending new values for CPU instead of what we set, and also giving you a suggested number for how much memory should be requested. We can at this point manually apply these suggestions, or allow VPA to apply them.

**Step 10** Update the manifest to set the policy to Auto and apply the configuration
```
sed -i 's/Off/Auto/g' hello-vpa.yaml
kubectl apply -f hello-vpa.yaml
```
In order to resize a pod, Vertical Pod Autoscaler will need to delete that pod and recreate it with the new size. By default, to avoid downtime, VPA will not delete and resize the last active pod. Because of this, you will need at least 2 replicas to see VPA make any changes.

**Step 11** Scale hello-server deployment to 2 replicas:
```
kubectl scale deployment hello-server --replicas=2
```

**Step 12** Watch your pods
```
kubectl get pods -w
```

**Step 13** The VPA should have resized your pods in the hello-server deployment. Inspect your pods:
```
kubectl describe pod hello-server | sed -n "/Containers:$/,/Conditions:/p"
```



## 1.7 Cleaning Up

**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-scaling
```
