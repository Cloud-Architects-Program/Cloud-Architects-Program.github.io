# K8s Networking

**Objective:**

  * Kubernetes `Network Policy`
  * Run applications and expose them via service via `Ingress` resource.


## 0 Create GKE Cluster
**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:
```
gcloud container clusters create k8s-networking \
--zone us-central1-c \
--enable-network-policy \
--num-nodes 2
```

!!! note
    To use GKE Ingress, you must have the HTTP(S) Load Balancing add-on enabled. GKE clusters have HTTP(S) Load Balancing enabled by default;

**Output:**
```
NAME          LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
k8s-scaling  us-central1-c  1.19.9-gke.1400  34.121.222.83  e2-medium     1.19.9-gke.1400  2          RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-networking --zone us-central1-c
``` 

## 1. Network Policy
### 1.1 Basic Policy Demo

This guide will deploy pods in a Kubernetes Namespaces. Let’s create the
Namespace object for this guide.

**Step 1** Create `policy-demo` Namespace:
```
kubectl create ns policy-demo
```

**Step 2** Than create a `policy-demo` nginx deployment in `policy-demo`
namespace:

```
kubectl create deployment --namespace=policy-demo nginx --replicas=2 --image=nginx
```


Then create a `policy-demo` service:

```
kubectl expose --namespace=policy-demo deployment nginx --port=80
```

**Step 3** Ensure  nginx service is accessible. In order to do that
create `busybox` pod and try to access the `nginx` service.

```
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
```

Output: 
```
Waiting for pod `policy-demo/access-472357175-y0m47` to be running, status is
Pending, pod ready: false
```


If you don't see a command prompt, try pressing enter.

```
/ # wget -q nginx -O -
```

!!! success
    You should see a response from `nginx`. Great! Our service is accessible.
    You can exit the Pod now.

!!! result
    Pods in a given namespace can be accessed by anyone.

**Step 4** Now let’s turn `on` isolation in our policy-demo Namespace.
Calico will then prevent connections to pods in this Namespace.
In order for Calico to prevent connection to pods in a namespace, we first need
to enable network isolation in the namespace by creating following `NetworkPolicy`:

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: policy-demo
spec:
  podSelector:
    matchLabels: {}
EOF
```

!!! note
    Current Lab is using Calico v3.0. Older versions of Calico
    (2.1 and prior) used namespace annotation to deny all traffic.


**Step 5** Verify that all access to the nginx Service is blocked.
We can see the effect by trying to access the Service again.

In order to do that, run a `busybox` deployment and try to access the `nginx`
service.



```
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
```


```
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is
Pending, pod ready: false
```

If you don't see a command prompt, try pressing enter.


```
/ # wget -q --timeout=5 nginx -O -
```

!!! result 
    `download timed out`


The request should time out after 5 seconds. By enabling isolation on the
Namespace, we’ve prevented access to the Service.

**Step 6** Allow Access using a NetworkPolicy

Now, let’s enable access to the nginx Service using a NetworkPolicy.
This will allow incoming connections from our access Pod, but not from anywhere
else.

Create a network policy `access-nginx` with the following contents:

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
  namespace: policy-demo
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: access
EOF
```

!!! notice
    The NetworkPolicy allows traffic from Pods with the label `run: access` to Pods with the label `app: nginx`. The labels are automatically added by kubectl and are based on the name of the resource.
**Step 7:** We should now be able to access the Service from the `access` Pod.

In order to check that create `busybox` deployment and try to access the `nginx`
Service.



```
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
```


```

Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is
Pending, pod ready: false
```

If you don't see a command prompt, try pressing enter.
```
/ # wget -q --timeout=5 nginx -O -
```


**Step 8** Without closing a prompt to a container, open a new terminal and run

```
kubectl get pods --show-labels -n policy-demo 
```

**Output:** 
```
NAME                     READY   STATUS    RESTARTS   AGE   LABELS
access                   1/1     Running   0          5s    run=access
nginx-6799fc88d8-5r64l   1/1     Running   0          17m   app=nginx,pod-template-hash=6799fc88d8
nginx-6799fc88d8-dbtgk   1/1     Running   0          17m   app=nginx,pod-template-hash=6799fc88d8
```

!!! result
    We can see the Labels `run=access` and `app=nginx`, that has been defined in Network Policy.

**Step 9** However, we still cannot access the Service from a Pod without the
label `run: access:`

Once again run `busybox` deployment and try to access the `nginx` service.


```
kubectl run --namespace=policy-demo cant-access --rm -ti --image busybox /bin/sh
```

```
Waiting for pod policy-demo/cant-access-472357175-y0m47 to be running, status
is Pending, pod ready: false
```

If you don't see a command prompt, try pressing enter.
```
/ # wget -q --timeout=5 nginx -O -
wget: download timed out
/ #
```
**Step 9** You can clean up the demo by deleting the demo Namespace:

```
kubectl delete ns policy-demo
```




### 1.2 (Demo) Stars Policy

#### 1.2.1 Deploy 3 tier-app
Let's Deploy 3 tier-app: UI, frontend and backend service, as well as a client
service.  And configures network policy on each service.

**Step 1** Deploy `Stars` Namespace and `management-ui` apps inside it:

```
kubectl create -f - <<EOF
---
kind: Namespace
apiVersion: v1
metadata:
  name: stars
---
apiVersion: v1
kind: Namespace
metadata:
  name: management-ui
  labels:
    role: management-ui
---
apiVersion: v1
kind: Service
metadata:
  name: management-ui
  namespace: management-ui
spec:
  type: LoadBalancer
  ports:
  - port: 9001
    targetPort: 9001
  selector:
    role: management-ui
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: management-ui
  labels:
    role: management-ui
  namespace: management-ui
spec:
  selector:
    matchLabels:
      role: management-ui
  replicas: 1
  template:
    metadata:
      labels:
        role: management-ui
    spec:
      containers:
      - name: management-ui
        image: calico/star-collect:v0.1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 9001
EOF
```
**Step 2** Deploy `backend` application inside of the  `stars` Namespace:

Backend:

```
kubectl create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: stars
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    role: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    role: backend
  namespace: stars
spec:
  selector:
    matchLabels:
      role: backend
  replicas: 1
  template:
    metadata:
      labels:
        role: backend
    spec:
      containers:
      - name: backend
        image: calico/star-probe:v0.1.0
        imagePullPolicy: Always
        command:
        - probe
        - --http-port=6379
        - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
        ports:
        - containerPort: 6379
EOF
```
**Step 3** Deploy `frontend` application inside of the `stars` Namespace:


```
kubectl create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: stars
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    role: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    role: frontend
  namespace: stars
spec:
  selector:
    matchLabels:
      role: frontend
  replicas: 1
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: frontend
        image: calico/star-probe:v0.1.0
        imagePullPolicy: Always
        command:
        - probe
        - --http-port=80
        - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
        ports:
        - containerPort: 80
EOF
```

**Step 4** Finally deploy `client` application inside of the `client` Namespace:

```
kubectl create -f - <<EOF
kind: Namespace
apiVersion: v1
metadata:
  name: client
  labels:
    role: client
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  labels:
    role: client
  namespace: client
spec:
  selector:
    matchLabels:
      role: client
  replicas: 1
  template:
    metadata:
      labels:
        role: client
    spec:
      containers:
      - name: client
        image: calico/star-probe:v0.1.0
        imagePullPolicy: Always
        command:
        - probe
        - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status
        ports:
        - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: client
  namespace: client
spec:
  ports:
  - port: 9000
    targetPort: 9000
  selector:
    role: client
EOF
```

**Step 5** Wait for all the pods to enter `Running` state.

```
kubectl get pods --all-namespaces --watch
```

!!! note
    It may take several minutes to download the necessary Docker images for this
    demo.

**Step 6** Locate LoadBalancer IP:


```
kubectl get svc -n management-ui
```


**Step 9** Open UI in you browser `http://EXTERNAL-IP:9001` in a browser.


!!! result
    Once all the pods are started, they should have full connectivity. You can see
    this by visiting the UI.  Each service is represented by a single node in the
    graph.

    - `backend` -> Node "B"
    - `frontend` -> Node "F"
    - `client` -> Node "C"

#### 1.2.2 Enable isolation
**Step 1** Enable isolation

Running the following commands will prevent all Ingress access to the frontend, backend,
and client Services located in namespaces `starts` and `client`

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: stars
spec:
  podSelector:
    matchLabels: {}
  ingress: []
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: client
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
EOF
```

!!! result
    Now that we've enabled isolation, the UI can no longer access the pods, and
    so they will no longer show up in the UI.

!!! note
    You need to Refresh web-browser to see result.

**Step 2** Allow apps from `stars` and `client` namespace access `UI` service via `NetworkPolicy` rules:

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: allow-ui
spec:
  podSelector:
   matchLabels: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: management-ui
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: client
  name: allow-ui
spec:
  podSelector:
   matchLabels: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: management-ui
EOF
```

!!! result
    refresh the UI - it should now show the Services, but they
    should not be able to access each other any more.

**Step 4** Create the `NetworkPolicy` to allow traffic from the
frontend to the backend.


```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      role: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
EOF
```
!!! result
    Refresh the UI.  You should see the following:

    - The frontend can now access the backend (on TCP port 80 only).
    - The backend cannot access the frontend at all.
    - The client cannot access the frontend, nor can it access the backend.

**Step 5** Expose the frontend service to the `client` namespace.

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: stars
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      role: frontend
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: client
      ports:
        - protocol: TCP
          port: 80
EOF
```

!!! success
    We isolated our app using `NetworkPolicy` ingress rules so that:

    - The client can now access the frontend, but not the backend.
    - Neither the frontend nor the backend can initiate connections to the client.
    - The frontend can still access the backend.


#### 1.2.3 Cleanup
**Step 1** You can clean up the demo by deleting the demo Namespaces:

```
kubectl delete ns client stars management-ui
```

## 2. Ingress
In GKE, Ingress is implemented using Cloud Load Balancing. In other words, when yiu create an Ingress resource in your cluster, GKE creates an `HTTP(S)` LB for you and configure it.
In this lab, we will create a fanout Ingress with two backends for two verisons of the `hello-app` application.

!!! note
    If you want to experiment with a different type of Ingress, you can install `Nginx Controller` using Helm. You can also follow these [instructions]
    (https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md).

### 2.1 Deploy an Application
**Step 1** Deploy the web application version 1, and its service

```
cat <<EOF > web-service-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v1
  namespace: default
spec:
  selector:
    matchLabels:
      run: web
      version: v1
  template:
    metadata:
      labels:
        run: web
        version: v1
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:1.0
        imagePullPolicy: IfNotPresent
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: web-v1
  namespace: default
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
    version: v1
  type: NodePort
EOF
```

**Step 2** Apply the resources to the cluster
```
kubectl apply -f web-service-v1.yaml
```

**Step 3** Deploy the web application version 2, and its service

```
cat <<EOF > web-service-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v2
  namespace: default
spec:
  selector:
    matchLabels:
      run: web
      version: v2
  template:
    metadata:
      labels:
        run: web
        version: v2
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:2.0
        imagePullPolicy: IfNotPresent
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: web-v2
  namespace: default
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: web
    version: v2
  type: NodePort
EOF
```

**Step 4** Apply the resources to the cluster
```
kubectl apply -f web-service-v2.yaml
```
**Step 5** Take a look at the pods created and view their IPs
```
kubectl get pod -o wide
```

**Step 6** Take a look at the endpoints created and notice how they match the IP for the Pods in the matching service.
```
kubectl get ep
```


### 2.2 Creating Single Service Ingress rule

Let's expose our web-v1 service using Ingress!

**Step 1** To start, let's create an Ingress resource that directs traffic to v1 of the `web` Service

```
cat <<EOF > single-svc-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: single-svc-ingress
spec:
  defaultBackend:
    service:
      name: web-v1
      port:
        number: 8080
EOF
```

This manifest defines a `Single Service Ingress rule`, which makes sure all HTTP requests
that will hit the external IP for the Cloud LB created for the Ingress, will be proxied to the `web-v1` service on port `8080`.

**Step 2** Create the Ingress resource
```
kubectl apply -f single-svc-ingress.yaml
```
**Step 3**  Verify Ingress Resource

```
kubectl get ing
```

**Output**
```
NAME                 CLASS    HOSTS   ADDRESS          PORTS   AGE
single-svc-ingress   <none>   *       34.117.137.144   80      110s
```
Notice that the Ingress controllers and load balancers may take a minute or two to allocate an IP address.

Bingo!

**Step 4** descirbe the Ingress object created and notice the Google Cloud LB resources created.
```
kubectl describe ing single-svc-ingress
```
Notice that you might need to give it some time to get a similar output to what I have below.
**Output:**
```
Name:             single-svc-ingress
Namespace:        default
Address:          34.117.137.144
Default backend:  web-v1:8080 (10.32.0.10:8080)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           *     web-v1:8080 (10.32.0.10:8080)
Annotations:  ingress.kubernetes.io/backends: {"k8s-be-30059--a2b52ac0331abe29":"HEALTHY"}
              ingress.kubernetes.io/forwarding-rule: k8s2-fr-8rwy2qu4-default-single-svc-ingress-74hvjigm
              ingress.kubernetes.io/target-proxy: k8s2-tp-8rwy2qu4-default-single-svc-ingress-74hvjigm
              ingress.kubernetes.io/url-map: k8s2-um-8rwy2qu4-default-single-svc-ingress-74hvjigm
Events:
  Type    Reason     Age                  From                     Message
  ----    ------     ----                 ----                     -------
  Normal  Sync       10m                  loadbalancer-controller  UrlMap "k8s2-um-8rwy2qu4-default-single-svc-ingress-74hvjigm" created
  Normal  Sync       10m                  loadbalancer-controller  TargetProxy "k8s2-tp-8rwy2qu4-default-single-svc-ingress-74hvjigm" created
  Normal  Sync       10m                  loadbalancer-controller  ForwardingRule "k8s2-fr-8rwy2qu4-default-single-svc-ingress-74hvjigm" created
  Normal  IPChanged  10m                  loadbalancer-controller  IP is now 34.117.137.144
  Normal  Sync       4m27s (x6 over 11m)  loadbalancer-controller  Scheduled for sync
```


**Step 5** Access deployed app via Ingress by navigating to the public IP of the ingress, which we got in step 3.


**Step 6** Finally execute the following command to cleanup the ingress.

```
kubectl delete -f single-svc-ingress.yaml
```

### 2.3 Simple fanout

Let's explore more sophisticated Ingress rules. This time we going to deploy `Simple fanout`type that allows for exposing multiple services on same host, but via different paths.
This type is very handy when you running in CloudProvider and want to cut cost
on creating LoadBalancers for each of you application. Or when running on
prem. and it's required to expose multiple services via same host.


**Step 1** Let's start by a simple fanout with one service. Notice that we are not using a `hostname` here as for this lab, we don't want to go through setting up DNS. If you have a DNS setup, you can try with an actual `hostname`. Just make sure to update you DNS records with the Ingress external IP.

```
cat <<EOF > fanout-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: fanout-ingress
spec:
  rules:
  - http:
      paths:
      - path: /v1
        pathType: ImplementationSpecific
        backend:
          service:
            name: web-v1
            port:
              number: 8080
EOF
```
**Step 2** Create the Ingress resource
```
kubectl apply -f fanout-ingress.yaml
```

**Step 3**  Verify Ingress Resource

```
kubectl get ing fanout-ingress
```
**Step 4** Verify that you can navigate to http://INGRESS_PUBLIC_IP/v1 and get the following
```
Hello, world!
Version: 1.0.0
```

**Step 5** Verify that ingress is returning `default backend - 404` page in a browser.

Open the browser with public_ip:
```
http://INGRESS_PUBLIC_IP/test
```

**Step 6** On your own modify the Ingress resource to add a second path so when we navigate to  `http://INGRESS_PUBLIC_IP/v2` users will get routed to `web-v2`


**Step 7** Delete the demo
Finally execute the following command to cleanup the application and ingress.

```
kubectl delete -f fanout-ingress.yaml
kubectl apply -f web-service-v1.yaml
kubectl apply -f web-service-v2.yaml

```
### 2.4 Cleaning Up

**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-networking
```

