# Istio Traffic Control - Canary Deployments

**Objective:**

  * Install Istio
  * Deploy an application and enable automatic sidecar injection
  * Deploy a canary service and use Istio to route some traffic to the new service


## Prerequisite  Delete Private Cluster

**Step 1:** Locate Terraform Configuration directory:

```
cd ~/$MY_REPO/notepad-infrastructure
```

**Step 2:** Delete Private Cluster that we've deployed with terraform:

```
terraform destroy -var-file terraform.tfvars
```

!!! result
    Private GKE Cluster, VPC and Subnets has been deleted

## 0 Create Regional GKE Cluster on GCP

**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:


```
gcloud compute networks create default \
    --subnet-mode=auto \
    --bgp-routing-mode=regional \
    --mtu=1460
```

```
gcloud container clusters create istio-traffic-lab \
--region us-central1 \
--enable-ip-alias \
--enable-network-policy \
--num-nodes 1 \
--machine-type "e2-standard-2" \
--release-channel stable
```

```
gcloud container clusters get-credentials istio-traffic-lab --region us-central1
```

**Step 3:** Setup kubectx

```
sudo apt install kubectx
```

!!! note
    we've installed `kubectx` + `kubens`: Power tools for `kubectl`:
    - `kubectx` helps you switch between clusters back and forth
    - `kubens` helps you switch between Kubernetes namespaces smoothly


## 1  Deploy Istio using Custom Resources


This installation guide uses the istioctl command line tool to provide rich customization of the Istio control plane and of the sidecars for the Istio data plane.


Download and extract the latest release:


```
curl -L https://istio.io/downloadIstio | sh -\
cd istio-1.11.1
export PATH=$PWD/bin:$PATH
```

!!! success
    The above command will fetch Istio packages and untar them in the same folder.

!!! note
    Istio installation directory contains:

    * `bin` directory contains `istioctl` client binary
    * `manifests` Installation Profiles, configurations and Helm Charts
    * `samples` directory contains sample applications deployment
    * `tools`  directory contains auto-completion tooling and etc.


**Step 2** Deploy Istio Custom Resource Definitions (CRDs)

Istio extends Kubernetes using Custom Resource Definitions (CRDs). 
CRDs allow registration of new/non-default Kubernetes resources. When Istio CRDs are deployed, Istio’s objects are registered as Kubernetes objects, providing a highly integrated experience with 
Kubernetes as a deployment platform and thus allowing Kubernetes to store configuration of Istio features such as routing, security and telemetry and etc.


We going to install using using `demo` configuration profile. The `demo` configuration profile allows to experiment with most of Istio features with modest resource requirements. 
Since it enables high levels of tracing and access logging, it is not suitable for production use cases.

```
export STATIC_IP=<set your Static IP>
```

```
istioctl install --set profile=demo --set values.gateways.istio-ingressgateway.loadBalancerIP=$STATIC_IP
```


**Output:**
```
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
Thank you for installing Istio 1.11.  Please take a few minutes to tell us about your install/upgrade experience!  
```


!!! note
    Wait a few seconds for the CRDs to be committed in the Kubernetes API-server.

**Step 3** Verify Istio CRDs successfully applied to Kubernetes Cluster.

```
kubectl get crds | grep istio
```

**Step 3** Verify that Istio Ingressgateway is setup using STATIC_IP Load Balancer:

List `STATIC_IP` Load Balancer
```
gcloud compute addresses list
```

**Output:**
```
NAME              ADDRESS/RANGE  TYPE      PURPOSE  NETWORK  REGION       SUBNET  STATUS
static-ingres-ip  35.192.101.76  EXTERNAL                    us-central1          IN_USE
```
!!! result 
    Our static IP showed as in use

Verify that Istio Ingressgateway using STATIC_IP:

```
kubectl get svc istio-ingressgateway -n istio-system
```

```
NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP     Ports
istio-ingressgateway   LoadBalancer   10.32.1.153   35.192.101.76   15021:32714/TCP,80:30042/TCP,443:31889/TCP,...  62s
```

!!! result
    Istio Gateway using 


##  2 Deploy an Micorservices Application

**Step 1** Get the source code for a sample application. For this lab, we will use the Online Boutique sample app.

```
cd ~
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
```
**Step 2** Create a namespace `online-boutique` for example
```
kubectl create namespace online-boutique
```
**Step 3** Enable automatic sidecar injection for the online-boutique namespace

```
kubectl label namespace online-boutique istio-injection=enabled --overwrite
```

Check namespace:

```
kubectl get ns online-boutique --show-labels
```

**Output:**
```
NAME              STATUS   AGE     LABELS
online-boutique   Active   4m30s   istio-injection=enabled
```

!!! result
    We've enabled Automatic Sidecar Injection on `online-boutique` namespace

**Step 5** Switch to `online-boutique` namespace:

List namespaces:

```
kubens
```

**Output:**
```
**default**
istio-system
kube-node-lease
kube-public
kube-system
online-boutique
```

!!! result we listed all namespaces and we see that active namespace is `default`

Switch to `online-boutique` namespace and make it active:


```
kubens online-boutique
```

```
kubens
```

**Output:**
```
**default**
istio-system
kube-node-lease
kube-public
kube-system
**online-boutique**
```

!!! result
    We switched context to `online-boutique` and it's now became active 
 

**Step 4** Deploy the application by creating the kubernetes manifests

```
kubectl apply -f ./release/kubernetes-manifests.yaml
```
**Step 5** Watch the pods and verify that all pods were started successfully with the sidecar injected

```
watch kubectl get pods
```

We can see containers starting as `Init:0/1` in this phase pod firewall rules gets update to establish communication with future Envoy Sidecar. Than `PodInitializing` start `Envoy` and application container.

**Output:**
```
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-5844cffbd4-h7lkk               2/2     Running   0          84s
cartservice-fdc659ddc-fzmn9              2/2     Running   0          86s
checkoutservice-64db75877d-hk25l         2/2     Running   0          88s
currencyservice-9b7cdb45b-9xzhl          2/2     Running   0          85s
emailservice-64d98b6f9d-v44rm            2/2     Running   0          89s
frontend-76ff9556-p62g9                  2/2     Running   0          87s
loadgenerator-589648f87f-4dfkp           2/2     Running   0          85s
paymentservice-65bdf6757d-zd4w4          2/2     Running   0          87s
productcatalogservice-5cd47f8cc8-c6hdn   2/2     Running   0          86s
recommendationservice-b75687c5b-clcbt    2/2     Running   0          88s
redis-cart-74594bd569-9gpjt              2/2     Running   0          84s
shippingservice-778554994-ttwf2          2/2     Running   0          85s
```

!!! result 
    Pods Started with sidecar auto-injected

##  3 Allow External traffic to Online Boutique Application

### 3.1 Create Gateway resource
In order for traffic to flow to the frontend of the application, we need to create an ingress gateway and a virtual service attached to it.


**Step 1** Apply the istio manifests to configure the Istio `Gateway`:

```
cat <<EOF> frontend-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: frontend-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```

```
kubectl apply -f frontend-gateway.yaml
```

!!! info
    Gateway resource important fields:
    
    * `name` - name of gateway
    * `spec.selector.istio` - which gateway to use (can be several)
    * `spec.servers` - defines which hosts will use this gateway proxy
    * `spec.servers.port.number` - ports to expose
    * `spec.servers.port.host` - host(s) for this port
    
!!! note 
    At this point our gateway is listening for request that match any host  on port 80 and it's accessible on the public internet.


Check the created gateway:

```
kubectl get gateway
kubectl get gateway -o yaml
```

Check the Envoy listeners been created:

```
export INGRESS_POD=$(kubectl get pods -n istio-system | grep 'ingress' |awk '{ print $1}')
istioctl proxy-config listener $INGRESS_POD  -n istio-system
```

**Output:**
```
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 8080  ALL   Route: http.8080
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*
```

!!! success
    HTTP port correctly exposed correctly


Verify what Ingress Gateway spec.selector.istio selects for use:

```
kubectl get pod --selector="istio=ingressgateway" --all-namespaces
```

!!! result 
    istio-ingressgateway-xxx pod in istio-system (our edge envoy pod) will proxy traffic further to Kubernetes services.


Try to connect to the Istio Ingress IP:

```
curl -i 35.192.101.76
```

**Output:**
```
HTTP/1.1 404 Not Found
```

!!! result
    It's returning 404 error.


!!! summary 
    We've create gateway resource that is listening for request that match any host on port 80 and it's accessible on the public internet, however it's returning 404 error because once the gateway receives the request it doesn't know where it needs to send it.

    In order to tell `Gateway` which Kubernets `service` it is supposed to receive the requests from  it is require to attach `Gateway` to `VirtualService`.


### 3.2 Create VirtualService resource

**Step 1** Apply the `istio` manifests to configure `VirtualService` to allow external traffic into the mesh and route it to the frontend service.

When traffic comes into the `gateway`, it is required to get it to a specific service within the `service` mesh and to do that, we’ll use the `VirtualService` resource.

In Istio, a `VirtualService` defines the rules that control how requests for a service are routed within an Istio service mesh. For instance, a `VirtualService` can route requests to different versions of a `service`, 
Requests can be routed based on the request source and destination, HTTP paths and header field and weights associated with individual service versions. 
As well as use advanced routing properties such as retries, request timeouts, circuit braking, fault injection and etc.



export STUDENT=<your_name> #Used to configure DNS

```
cat <<EOF> frontend-vs.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-ingress
spec:
  hosts:
  - "$STUDENT.cloud-montreal.ca"
  gateways:
  - frontend-gateway
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
EOF
```


```
kubectl apply -f frontend-vs.yaml
```

Acquire `frontend` app URL:


```
kubectl get virtualservices
```


Access `frontend` app URL from your browser:

```
$STUDENT.cloud-montreal.ca
```

!!! success 
    We can access `frontend` web app via Istio Ingress Gateway



* frontend.yaml: defines another virtual service to be used by our load generator to make sure traffic directed to the frontend from within the mesh stays internal, and doesn't go through the gateway.
*  Whitelist-egress-googleapis.yaml: Used to whitelist various google APIs used by services within the mesh.

**Step 3** Defines another `Virtualservice` to be used by our load generator to make sure traffic directed to the frontend from within the mesh stays internal, 
and doesn't go through the gateway.

```
cat <<EOF> frontend-load-vs.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend
spec:
  hosts:
  - "frontend.default.svc.cluster.local"
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
EOF
```

```
kubectl apply -f frontend-load-vs.yaml
```

!!! summary
    We've 

##  4 Routing traffic to services outside of the Mesh


**Step 1** Create `ServiceEntry` custom resource to allow access to external APIs `googleapis.com` and GCE metadata service `metadata.google.internal` that app is using:

Because all outbound traffic from an Istio-enabled pod is redirected to its sidecar proxy by default, accessibility of URLs outside of the cluster depends on the configuration of the proxy. 
By default, Istio configures the Envoy proxy to passthrough requests for unknown services. 
Although this provides a convenient way to get started with Istio, configuring stricter control is usually preferable in production scenarios.


Istio has an option, `global.outboundTrafficPolicy.mode`, that configures the sidecar handling of external services, that is, those services that are not defined in Istio’s internal service registry. 
If this option is set to `ALLOW_ANY`, the Istio proxy lets calls to unknown services pass through. If the option is set to `REGISTRY_ONLY`, 
then the Istio proxy blocks any host without an HTTP service or service entry defined within the mesh.



```
cat <<EOF> allow-egress-googleapis.yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: allow-egress-googleapis
spec:
  hosts:
  - "accounts.google.com" # Used to get token
  - "*.googleapis.com"
  ports:
  - number: 80
    protocol: HTTP
    name: http
  - number: 443
    protocol: HTTPS
    name: https
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: allow-egress-google-metadata
spec:
  hosts:
  - metadata.google.internal
  addresses:
  - 169.254.169.254 # GCE metadata server
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
EOF
```


```
kubectl apply -f allow-egress-googleapis.yaml
```


```
kubectl get serviceentries  
```

!!! result 
    `ServiceEntry` resource inserts an entry into Istio’s service registry which makes explicit that clients within the mesh are allowed to call a `metadata.google.internal`, `accounts.google.com`, `*.googleapis.com` regardless the namespaces


##  4 Canary Deployments with Istio

We going to deploy a new version of `recommendationservice` with `canary` label.

Once we’ve deployed the new version, we can start releasing it gradually by routing 20% of all incoming requests to the latest version, while the rest of the requests (98%) still goes to the existing version.



**Step 1** Patch the Recommendation service deployment to give it a `production` label

```
kubectl patch deployment -n online-boutique recommendationservice --type='json' -p='[{"op": "add", "path": "/spec/template/metadata/labels/version", "value": "production"}]'
```

**Step 2** Check the labels on the deployment
```
export RECOMMMENDATIONSERVICE=$(kubectl get pods | grep 'recommendationservice' |awk '{ print $1}')
kubectl get pod -n online-boutique $RECOMMMENDATIONSERVICE --show-labels
```

**Step 3** Create a DestinationRule that defines two subsets `production` and `canary`

```
cat <<EOF>  destinationrule-recommendation.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: recommendationservice
  namespace: online-boutique
spec:
  host: recommendationservice
  subsets:
  - name: production
    labels:
      version: production
  - name: canary
    labels:
      version: canary
EOF
```

```
kubectl apply -f  destinationrule-recommendation.yaml
```


```
kubectl get destinationrules
```

!!! result
    We've defined DestinationRule with 2 subsets.

**Step 4** Make a change to the `Recommendation` service and by modifying it's `max_responses` parameter from `5` to `2`.

```
cd ~/microservices-demo
ls
cat src/recommendationservice/Dockerfile
```


```
sed -i 's/max_responses = 5/max_responses = 2/g' \
   src/recommendationservice/recommendation_server.py
```

**Step 5** Build a new version of the image
```
PROJECT_ID=$(gcloud config list --format "value(core.project)")

docker build -t \
  gcr.io/${PROJECT_ID}/microservices_demo/recommendationservice:canary \
  src/recommendationservice
```

**Step 6** Push the image to GCR
```
docker push gcr.io/${PROJECT_ID}/microservices_demo/recommendationservice:canary
```

**Step 7** Deploy the `canary` version of the Recommendation service:

```
cat <<EOF>  recommendation-deployment-vcanary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: recommendationservice
    version: canary
  name: recommendationservice-canary
  namespace: online-boutique
spec:
  replicas: 1
  selector:
    matchLabels:
      app: recommendationservice
      version: canary
  template:
    metadata:
      labels:
        app: recommendationservice
        version: canary
    spec:
      containers:
      - env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: productcatalogservice:3550
        - name: ENABLE_PROFILER
          value: "0"
        image: gcr.io/$PROJECT_ID/microservices_demo/recommendationservice:canary
        imagePullPolicy: Always
        livenessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:8080
          failureThreshold: 3
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        name: server
        ports:
        - containerPort: 8080
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - /bin/grpc_health_probe
            - -addr=:8080
          failureThreshold: 3
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 200m
            memory: 450Mi
          requests:
            cpu: 100m
            memory: 220Mi
EOF
```

```
kubectl apply -f recommendation-deployment-vcanary.yaml
```

```
kubectl get deploy | grep recommendationservice 
```


**Output:**
```
recommendationservice          1/1     1            1           16m
recommendationservice-canary   1/1     1            1           31s
```

!!! result
    2 versions of `recommendationservice` has been deployed


**Step 8** Create a virtual service to direct 20% of the traffic to the `canary` version

```
cat <<EOF>  vs-recommendation.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: recommendationservice
  namespace: online-boutique
spec:
  hosts:
  - recommendationservice
  http:
  - route:
    - destination:
        host: recommendationservice
        port:
          number: 8080
        subset: production
      weight: 80
    - destination:
        host: recommendationservice
        port:
          number: 8080
        subset: canary 
      weight: 20
EOF
```

```
kubectl apply -f vs-recommendation.yaml
```

```
kubectl get VirtualService
```

**Step 9** Refresh the page for one of the products where recommendations are shown multiple times, and notice how most of the time you get 5 recommendations while if 20% of the time you get only 2 recommendations.


## 5 Cleanup



Delete GKE cluster:

```
gcloud container clusters delete istio-traffic-lab --region us-central1
```