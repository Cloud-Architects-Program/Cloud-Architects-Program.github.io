# Kubernetes Concepts

Objective:

Learn basic Kubernetes concepts:

  * Create a GKE Cluster
  * Pods
  * Labels, Selectors and Annotations
  * Create Deployments
  * Create Services
  * namespaces

## 0 Create GKE Cluster
**Step 1** Enbale the Google Kubernetes Engine API.
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

## 1 Pods
### 1.1 Create a Pod with manifest

**Reference:** [Pod Overview](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)


**Step 1** Printout explanation of the object and lists of attributes:

```
kubectl explain pods
```
See all possible fields available for the pods:
```
kubectl explain pods.spec --recursive
```

!!! note
    It's not require to provide all possible fields for the Pods or any other
    resources. Most of the fields will be added by default if not specified.
    For the Pods at minimum it is required to specify `image`, `name`, `ports`
    inside of spec.containers.

**Step 2** Define a new pod in the file `echoserver-pod.yaml` :
```
cat <<EOF > echoserver-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: echoserver
  labels:
    app: echoserver
spec:
  containers:
  - name: echoserver
    image: gcr.io/google_containers/echoserver:1.10
    ports:
    - containerPort: 8080
EOF
```
Here, we use the existing image `echoserver`. This is a simple server that
responds with the http headers it received. It runs on nginx server and
implemented using lua in the nginx configuration: [https://github.com/kubernetes/contrib/tree/master/ingress/echoheaders](https://github.com/kubernetes/contrib/tree/master/ingress/echoheaders)

**Step 3** Create the `echoserver` pod:
```
kubectl apply -f echoserver-pod.yaml
```
**Step 4** Use `kubectl get pods` to watch the pod get created:
```
kubectl get pods
```
**Result:**
```
NAME         READY     STATUS    RESTARTS   AGE
echoserver   1/1       Running   0          5s
```

**Step 5** Use `kubectl describe pods/podname` to watch the details about scheduled pod:
```
kubectl describe pods/echoserver
```

!!! note
    Review and discuss the following fields:

      * Namespace

      * Status

      * Containers

      * QoS Class

      * Events

**Step 6** Now let’s get the pod definition back from Kubernetes:
```
kubectl get pods echoserver -o yaml > echoserver-pod-created.yaml
cat echoserver-pod-created.yaml
```

Compare echoserver-pod.yaml and echoserver-pod-created.yaml to see additional
properties that have been added to the original pod definition.

## 2 Labels & Selectors
Organizing pods and other resources with labels.

### 2.1 Label and Select Pods
Reference: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

**Step 1** Label Pod `hello-world` with label `app=hello` and `env=test`

```
kubectl label pods echoserver dep=sales
kubectl label pods echoserver env=test
```

**Step 2** See all Pods and all their Labels.

```
kubectl get pods --show-labels
```

**Step 3** Select all Pods with labels `env=test`

```
kubectl get pods -l env=test
```


### 2.2 Label Nodes
Reference: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

**Step 1** List available nodes
```
kubectl get nodes
```

**Step 2** List a detailed view of nodes
```
kubectl get nodes -o wide
```

**Step 3** List Nodes and their labels

```
kubectl get nodes --show-labels
```

**Step 4**  Label the node as `size: small`. Make sure to replace YOUR_NODE_NAME with one of the nodes you have.

```
kubectl label node YOUR_NODE_NAME size=small
```

**Step 5** Check the labels for this node

```
kubectl get node YOUR_NODE_NAME --show-labels | grep size
```

!!! note
    In the upcoming classes we will use node labels to make sure our applications run on eligible nodes only.

## 3 Services
### 3.1 Create a Service

We have three running `echoserver` pods, but we cannot access them yet,
because the container ports are not accessible. Let’s define a new service that
will expose echoserver ports and make them accessible.

**Step 1** Create a new file `echoserver-service.yaml` with the following content:

```
cat <<EOF > echoserver-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: echoserver
spec:
  selector:
    app: echoserver
  type: "NodePort"
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: echoserver
EOF
```

**Step 2** Create a new service:

```
kubectl create -f echoserver-service.yaml
```


**Step 3** Check the service details:

```
kubectl describe services/echoserver
```

**Output:**

```
Name:               echoserver
Namespace:          default
Labels:             <none>
Selector:           app=echoserver
Type:               NodePort
IP:                 ...
Port:               <unset> 8080/TCP
NodePort:           <unset> 30366/TCP
Endpoints:          ...:8080,...:8080,..:8080
Session Affinity:   None
No events.
```

!!! note
    The above output contains one endpoint and a node port, 30366, but it can be different in your case. Remember this port to use it in the next step.

**Step 4** We need to open the node port on one of the cluster nodes to be able to access the service externally. Let's first find the exteran IP address of one of the nodes.
```
kubectl get nodes -o wide
```

**Output:**
```
NAME                                          STATUS   ROLES    AGE   VERSION            INTERNAL-IP   EXTERNAL-IP    OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-k8s-concepts-default-pool-ad96fd50-1rf1   Ready    <none>   20m   v1.19.9-gke.1400   10.128.0.32   34.136.1.22
gke-k8s-concepts-default-pool-ad96fd50-jpd2   Ready    <none>   20m   v1.19.9-gke.1400   10.128.0.31   34.69.114.67
```

**Step 5** Create a firewall rule to allow TCP traffic on your node port. Make sure to replace YOUR_NODE_PORT.
```
gcloud compute firewall-rules create echoserver-node-port --allow tcp:YOUR_NODE_PORT
```

**Step 6** To access a service exposed via a node port, specify the node port
from the previous step and use one of the IP addresses of the cluster nodes. Make sure to replace both NODE_IP and YOUR_NODE_PORT

```
curl http://NODE_IP:YOUR_NODE_PORT
```

### 3.2 Cleanup Services and Pods

**Step 1** Before diving into Kubernetes deployment, let’s delete our service and pods. To delete the service execute the following command:

```
kubectl delete service echoserver
```
**Step 2** delete the pod
```
kubectl delete pod echoserver
```

**Step 3** Check that there are no running pods:
```
kubectl get pods
```

## 4 Deployments
### 4.1 Deploy hello-app on Kubernetes using Deployments

#### 4.1.1 Create a Deployment

**Step 1** The simplest way to create a new deployment for a single-container
pod is to use `kubectl run`:

```
kubectl create deployment hello-app \
--image=gcr.io/google-samples/hello-app:1.0 \
--port=8080 \
--replicas=2
```

!!! note
    `--port` Deployment opens port 8080 for use by the Pods.

    `--replicas` number of replicas.

**Step 2** Check pods:

```
kubectl get pods
```

**Step 3** To access the `hello-app` deployment, create a new service of type LoadBalancer this time using
`kubectl expose deployment`:

```
kubectl expose deployment hello-app --type=LoadBalancer
```

To get the external IP for the loadbalancer that got created:
```
kubectl get services/hello-app
```
The Loadbalancer might take few minutes to get created, and it'll show pending status.

**Step 4** Check that the hello-app is accessible:
Make sure to replace the LB_IP.
```
curl http://LB_IP:8080
```
**Output:**
```
Hello, world!
Version: 1.0.0
Hostname: hello-app-76f778987d-rdhr7
```

**Step 5** You can open the app in the browser by navigating to LB_IP:8080

!!! Summary
    We learned how to create a deployment and expose our container.

#### 4.1.2 Scale a Deployment

Now, let's scale our application as our website get popular.

**Step 1** Deployments using replica set (RS) to scale the containers.
Let's check how replica set (RS) looks like:

```
kubectl get rs,deploy
```
```
NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-app-76f778987d   2         2         2       5m12s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-app   2/2     2            2           5m12s
```

**Step 2** Let’s scale number of pods in replica for the deployment.
Use going to use `kubectl scale` to change the number of replicas to 5:

```
kubectl scale  deployment hello-app --replicas=5
```

**Step 3** View the deployment details:

```
kubectl describe deployment hello-app
```

**Step 4** Check that there are 5 running pods:

```
kubectl get pods
```

#### 4.1.3 Rolling Update of Containers

To perform rolling upgrade we need a new version of our application and
then  perform Rolling Upgrade using `deployments`

**Step 4** Use `kubectl rollout history deployment` to see revisions of the
deployment:

```
kubectl rollout history deployment hello-app
```
**Output:**
```
deployment.apps/hello-app 
REVISION    CHANGE-CAUSE
1           <none>
```

!!! result
    Since we've just deployed there is only 1 revision that currenly running.

**Step 5** Now we want to replace our `hello-app` with a new implementation.
We want to use a new version of `hello-app` image. We are going to use `kubectl set`
command this time around.

!!! Hint
    `kubectl set` used only to change image name/version.
    You can use this for command for CI/CD pipeline.

Suppose that we want to update the webk8sbirthday Pods to use the
hello-app:2.0 image instead of the hello-app:1.0 image.

```
kubectl set image deployment/hello-app hello-app=gcr.io/google-samples/hello-app:2.0 --record
kubectl get pods
```

!!! note
    It is a good practice to paste `--record` at the end of the rolling upgrade
    command as it will record the action in the `rollout history`

**Result:** We can see that the Rolling Upgraded was recorded:
```
kubectl rollout history deployment hello-app
```
**Output:**
```
deployment.apps/hello-app 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/hello-app hello-app=gcr.io/google-samples/hello-app:2.0 --record=true
```

**Step 6** Refresh browser and see new version of app deployed
```
http://LB_IP:8080
```

**Step 7** Let's assume there was something wrong with this new version and we need to **rollback** with `kubectl rollout undo` our deployment:

```
kubectl rollout undo deployment/hello-app
```

Refresh the browser again to see how we rolledback to version 1.0.0

We have successfully rolled back the deployment and our pods are based on the
hello-app:1.0.0 image again.

**Step 8** Let's delete the deployment and the service:

```
kubectl delete deployment hello-app
kubectl delete services/hello-app
```



!!! success
    You are now up to speed with Kubernetes Concepts such as Pods, Services and Deployments.  Let's move on to Kubernetes
    Features to learn what else Kubernetes is capable of!

## 5 NameSpace

Namespace can be used for:

  + Splitting complex systems with several components into smaller groups
  + Separating resources in a multi-tenant env: production, development and
QA environments
  + Separating resources per production
  + Separating per-user or per-department or any other logical group

Some other rules and regulations:

  + Resource names only need to be unique within a namespace.
  + Two different namespaces can contain resources of the same name.
  + Most of the Kubernetes resources (e.g. pods, svc, rcs, and others) are
  namespaced.
  + However, some resource can be cluster-wide e.g nodes, persistentVolumes and
  PodSecurityPolicy.

### 5.1 Viewing namespaces

**Step 1** List the current namespaces in a cluster using:

```
kubectl get ns
```

**Output:**
```
NAME              STATUS   AGE
default           Active   71m
kube-node-lease   Active   71m
kube-public       Active   71m
kube-system       Active   71m
```

**Step 2** You can also get the summary of a specific namespace using:

```
kubectl get namespaces <name>
```

Or you can get detailed information with:

```
kubectl describe namespaces <name>
```
A namespace can be in one of two phases:

   * `Active` the namespace is in use
   * `Terminating` the namespace is being deleted, and can not be used for new
    objects


!!! note
    These details show both `resource quota` (if present) as well as `resource
    and limit` ranges.

    `Resource quota` tracks aggregate usage of resources in the *Namespace* and
    allows cluster operators to define *Hard* resource usage limits that a
    *Namespace* may consume.

     A `limit range` defines min/max constraints on the amount of resources a
     single entity can consume in a *Namespace*.

**Step 3** Let’s have a look at the pods that belong to the `kube-system`
namespace, by telling kubectl to list pods in that namespace:

```
kubectl get po --namespace kube-system
```

**Output:**
```
NAME                                                        READY   STATUS    RESTARTS   AGE
event-exporter-gke-67986489c8-5fsdv                         2/2     Running   0          71m
fluentbit-gke-fqcsx                                         2/2     Running   0          71m
fluentbit-gke-ppb9j                                         2/2     Running   0          71m
gke-metrics-agent-5vl7t                                     1/1     Running   0          71m
gke-metrics-agent-bxt2r                                     1/1     Running   0          71m
kube-dns-5d54b45645-9srx6                                   4/4     Running   0          71m
kube-dns-5d54b45645-b7njm                                   4/4     Running   0          71m
kube-dns-autoscaler-58cbd4f75c-2scrv                        1/1     Running   0          71m
kube-proxy-gke-k8s-concepts-default-pool-ad96fd50-1rf1      1/1     Running   0          71m
kube-proxy-gke-k8s-concepts-default-pool-ad96fd50-jpd2      1/1     Running   0          71m
l7-default-backend-66579f5d7-dsbdt                          1/1     Running   0          71m
metrics-server-v0.3.6-6c47ffd7d7-mtls4                      2/2     Running   0          71m
pdcsi-node-knlqp                                            2/2     Running   0          71m
pdcsi-node-vh4tx                                            2/2     Running   0          71m
stackdriver-metadata-agent-cluster-level-6f7d66dc98-zcd25   2/2     Running   0          71m
```

!!! tip
    You can also use -n instead of --namespace

Yot may already know some of the pods, the rest we will cover later. It’s clear
from the name of the namespace, that resources inside `kube-system` related to
the Kubernetes system itself. By having them in this separate namespace, it
keeps everything nicely organized. If they were all in the default namespace,
mixed in with the resources we create ourselves, we’d have a hard time seeing
what belongs where and we might inadvertently delete some system resources.

**Step 4** Now you know how to view resources in specific namespaces.
Additionally, it is also possible to view list all resources in all namespaces.
For example below is example to list all pods in all namespaces:

```
kubectl get pods --all-namespaces
```

### 5.2 Creating Namespaces
A namespace is a Kubernetes resource, therefore it is possible to create it by
posting a YAML file to the Kubernetes API server or using `kubectl create ns`.

**Step 1** First, create a custom-namespace.yaml file with the following
content:
```
cat <<EOF > custom-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
EOF
```

**Step 2** Than, use kubectl to post the file to the Kubernetes API server:
```
kubectl create -f custom-namespace.yaml
```

**Step 3**
A much easier and faster way to create a namespaces using `kubectl create ns`
command, as shown below:

```
kubectl create namespace custom-namespace2
```


### 5.3 Setting the namespace preference.
By default, a Kubernetes cluster will instantiate a `default namespace` when
provisioning the cluster to hold the default set of Pods, Services, and
Deployments used by the cluster. So by default all `kubectl` calls such as
list or create resources will end up in `default namespace`.

However sometimes you want to list or create resources in other namespaces than
`default namespace`. As we discussed in previous exercise this can be done by
specifying `-n`  or `--namespace` to point in which namespaces action has to
be done. However it is not convenient to do this action every time. Below example will show
how to create 2 namespaces `dev` and `prod` and switch between each other.

**Step 1** `kubectl` API uses so called `kubeconfig context` where you can
controls which namespace, user or cluster needs to be accessed.

In order to display which context is currently in use run:
```
KUBECONFIG=~/.kube/config
kubectl config current-context
```

!!! result
    We running in `kubernetes-admin@kubernetes` context, which is default for
    lab environment.

**Step 3** To see full view of the `kubeconfig context` run:

```
kubectl config view
```

!!! result
    Our context named as `kubernetes-admin@kubernetes` uses cluster `cluster`,
    and user `kubernetes-admin`


**Step 4** The result of the above command comes from kubeconfig file, in our
case we defined it under `~/.kube/config`.

```
echo $KUBECONFIG
```

!!! result
    KUBECONFIG is configured to use following ~/.kube/config file
```
cat ~/.kube/config
```

!!! note
    The KUBECONFIG environment variable is a list of paths to configuration
    files. The list is colon-delimited for Linux and Mac, and
    semicolon-delimited for Windows. We already set KUBECONFIG environment
    variable in a first step of exercise to be `~/.kube/config`


!!! tip
    You can use use multiple `kubeconfig` files at the same time and view merged
    config:

    ```
    $ KUBECONFIG=~/.kube/config:~/.kube/kubconfig2 kubectl config view
    ```

**Step 5** Create two new namespaces `dev`:

```
kubectl create namespace dev
```
And  `prod` namespace:
```
kubectl create namespace prod
```

**Step 7** Let’s switch to operate in the `development` namespace:

```
kubectl config set-context --current --namespace=dev
```

We can see now that our current context is switched to dev:

```
kubectl config view | grep namespace
```

Output:
```
    namespace: dev
```
!!! result
    At this point, all requests we make to the Kubernetes cluster from the
    command line are scoped to the development namespace.

**Step 8** Let's test that all resources going to be created in `dev` namespace.

```
kubectl run devsnowflake --image=nginx
```

**Step 9** Verify result of creation:
```
kubectl get pods
```

!!! success
    Developers are able to do what they want, and they do not have to worry
    about affecting content in the production namespace.

**Step 10** Now switch to the `production` namespace and show how resources in
one namespace are hidden from the other.

```
kubectl config set-context --current --namespace=prod
```

The production namespace should be empty, and the following commands should
return nothing.

```
kubectl get pods
```

**Step 11** Let's create some `production` workloads:

```
kubectl run prodapp --image=nginx
```

```
kubectl get pods -n prod
```


!!! summary
    At this point, it should be clear that the resources users create in one
    namespace are hidden from the other namespace.

### 6.4 Deleting Namespaces

**Step 1** Delete a namespace with

```
kubectl delete namespaces custom-namespace
kubectl delete namespaces dev
kubectl delete namespaces prod
```

!!! warning
    Unlike with OpenStack where when you delete a project/tenant, underlining
    resources will still exist as zombies and not deleted. In Kubernetes when you
    delete namespace it deletes _everything_ under it (pods, svc, rc, and etc.)!
    This is called `resource garbage collection` in Kubernetes.

Delete process is asynchronous, so you may see `Terminating` state for some
time.


### 6.5 Create a pod in a different namespace
Create `test` namespace:

```
kubectl create ns test
```
Create a pod in this namespaces:
```
cat <<EOF > echoserver-pod_ns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: echoserverns
spec:
  containers:
  - name: echoserver
    image: gcr.io/google_containers/echoserver:1.4
    ports:
    - containerPort: 8080
EOF
```
Create Pod in namespaces:
```
kubectl create -f echoserver-pod_ns.yaml -n test
```

Verify Pods created in specified namespaces:

```
kubectl get pods -n test
```




## 6 Cleaning Up
**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-concepts
```
**Step 2** Delete the firewall rule
```
gcloud compute firewall-rules delete echoserver-node-port
```