# Lab 5 Cloudrun for Anthos

**Objective:**

  * Enable CloudRun on a Kubernetes cluster
  * Deploy a web application
  * Autoscale applications from 0-to-1 and back to 0

## 0 Create Regional GKE Cluster on GCP and Enable CloudRun

**Step 1** Enable the Google Kubernetes Engine API.
```
gcloud services enable \
container.googleapis.com \
containerregistry.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node and enable CloudRun:

```
gcloud container clusters create cloudrun-demo \
  --zone=us-central1-b \
  --num-nodes=1 \
  --machine-type=n1-standard-4 \
  --enable-autoscaling --min-nodes=1 --max-nodes=5 \
  --enable-ip-alias \
  --addons=HttpLoadBalancing,CloudRun \
  --enable-stackdriver-kubernetes \
  --release-channel stable
```

Enabling CloudRun will install Knative components on the cluster. As we discussed in class, Knative makes it possible to:

* Deploy and serve applications using Knative API. 
* Allow applications automatically scale from zero-to-N, and back to zero, based on requests.
* Build and package the application code inside the cluster.
* Deliver events to your application; However, this feature is not enabled by CloudRun.

**Step 3** Verify the Knative components are properly installed.

```
kubectl get pods --namespace=knative-serving
```

## 1 Deploy a Knative Service

**Step 1** deploy a Knatvie service to the cluster you just created. For this example, we will use `kubectl`; however, you can also use the CloudRun UI. To expose an application on Knative, we need to define a Service object, which is different than the Kubernetes Service type.

```
cat <<EOF> helloworld.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: gcr.io/cloudrun/hello
          env:
            - name: TARGET
              value: "Go Sample v1"
EOF 
```

```
kubectl apply -f helloworld.yaml
```

**Step 2** Verify the service is deployed by querying "ksvc" (Knative Service) objects
```
kubectl get ksvc
```

**Step 3** Although the Knative service is deployed, there are no Pod created yet, given that there are no requests on this service.

```
kubectl get pod
```

## 2 Test the Knative Service

**Step 1** Let's test the service by making a request using `curl`.

To avoid having to set up DNS, we'll test the deployed service by sending a request to the Istio ingress gateway with the target host that should handle the request as an HTTP header. That hostname should be the URL listed in the previous deployment step and of the form: http://*service-name*.*namespace*.example.com

**Step 2** Obtain the IP address of the ingress gateway
```
kubectl get svc -n gke-system istio-ingress
```

**Step 3** From Cloud Shell, use curl to access the service. Use the URL we obtained in step 5, and replace [YOUR-IP] with the IP address you obtained in the previous step.

```
curl -v -H "Host: helloworld-go.default.example.com" http://[YOUR-IP]
```

**Step 4** Verify that a pod was created and that this pod will get termintated if there are no more requests on this service.

```
kubectl get pod
```

## 3 Knative Serving API
When we deploy the helloworld Service to Knative, it creates three kinds of objects: Configuration, Route, and Revision.

```
kubectl get configuration,revision,route
```

* **Service** Describes an application on Knative.
* **Revision** Read-only snapshot of an application's image and other settings
* **Configuration** Created automatically by the service. It creates a new Revision when the revisionTemplate field changes.
* **Route** Configures how the traffic coming to the Service should be split between Revisions.

## 4 Cleanup
Delete the GKE cluster:

```
gcloud container clusters delete cloudrun-demo --region us-central1
```