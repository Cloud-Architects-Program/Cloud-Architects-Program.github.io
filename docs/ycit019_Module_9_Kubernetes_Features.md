Lab 8 Kubernetes Features


**Objective**

* Use Liveness Probes to healthcheck you application while it is running
* Learn about secrets and configmaps
* Deploy a Daemonset and jobs

## 0 Create GKE Cluster
**Step 1** Enbale the Google Kubernetes Engine API.
```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:
```
gcloud container clusters create k8s-features \
--zone us-central1-c \
--num-nodes 2
```

**Output:**
```
NAME          LOCATION       MASTER_VERSION   MASTER_IP      MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
k8s-features  us-central1-c  1.19.9-gke.1400  34.121.222.83  e2-medium     1.19.9-gke.1400  2          RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-features --zone us-central1-c
``` 

# 1 Kubernetes Features

## 1.1 Using Liveness Probes

Many applications running for long periods of time eventually transition to
broken states, and cannot recover except by being restarted. Kubernetes p
rovides liveness probes to detect and remedy such situations.

As we already discussed Kubernetes provides 3 types of `Probes` to perform
Liveness checks:

  * HTTP GET
  * EXEC
  * tcpSocket

In below example we are going to use HTTP GET probe for a Pod that runs a container
based on the `gcr.io/google_containers/liveness` image.

** Step 1** Create `http-liveness.yaml` manifest with below content:

```
cat <<EOF > http-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: gcr.io/google_containers/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
EOF
```

Based on the manifest, we can see that the Pod has a single Container.
The `periodSeconds` field specifies that the kubelet should perform a liveness
probe every 3 seconds. The `initialDelaySeconds` field tells the kubelet that it
should wait 3 seconds before performing the first probe. To perform a probe, the
kubelet sends an HTTP GET request to the server that is running in the Container
and listening on port 8080. If the handler for the server's `/healthz` path
returns a success code, the kubelet considers the Container to be alive and
healthy. If the handler returns a failure code, the kubelet kills the Container
and restarts it.

Any code greater than or equal to 200 and less than 400 indicates success. Any
other code indicates failure.

Full code for for a reference in  [server.go](https://github.com/kubernetes/kubernetes/blob/master/test/images/liveness/server.go).

For the first 10 seconds that the Container is alive, the `/healthz` handler
returns a status of 200. After that, the handler returns a status of 500.

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

The kubelet starts performing health checks 3 seconds after the Container starts.
So the first couple of health checks will succeed. But after 10 seconds, the health
checks will fail, and the kubelet will kill and restart the Container.

**Step 2** Let's create a Pod and see how HTTP liveness check works:

```shell
kubectl create -f http-liveness.yaml
```


**Step 3** Monitor the Pod

```
watch kubectl get pod
```

!!! result
    After 10 seconds Pods has beed restarted.

Exit the shell session by using ctrl+c

**Step 4** Verify status of the Pod and review the Events happened after restart

```shell
kubectl describe pod liveness-http
```

!!! result
    Pod events shows that liveness probes have failed and the Container has been restarted.

**Step 5** Clean up

```
kubectl delete -f http-liveness.yaml
```

## 1.2 Using ConfigMaps

In Kubernetes ConfigMaps could be use in several cases:

  * Storing configuration values as key-values in ConfigMap and referencing them
  in a Pod as environment variables
  * Storing configurations as a file inside of ConfigMap and referencing it in
  a Pod as a Volume


Let's try second option and deploy nginx pod while storing its config in a ConfigMap.


**Step 1** Create nginx `my-nginx-config.conf` config file as below:

```
cat <<EOF > my-nginx-config.conf
server {
  listen              80;
  server_name         www.cloudnative.tech;

  gzip on;
  gzip_types text/plain application/xml;

  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
  }
}
EOF
```

**Step 2** Create ConfigMap from this file

```
kubectl create configmap nginxconfig --from-file=my-nginx-config.conf
```

**Step 3** Review the ConfigMap

```
kubectl describe cm nginxconfig
```

**Result:**

      ```
      Name:		nginxconfig
      Namespace:	default
      Labels:		<none>
      Annotations:	<none>

      Data
      ====
      my-nginx-config.conf:
      ----
      server {
        listen              80;
        server_name         _;

        gzip off;
        gzip_types text/plain application/xml;

        location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
        }
      }

      Events:	<none>
      ```

**Step 4** Create Nginx Pod `website.yaml` file, where ConfigMap referenced as a Volume

```
cat <<EOF > website.yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
spec:
  containers:
  - image: nginx:alpine
    name: website
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: false
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: nginxconfig
EOF
```

**Step 5** Deploy Nginx Pod `website.yaml`

```
kubectl apply -f website.yaml
```

You can expose a service as we usually do with NodePort or LoadBalancer or by using the port-forward technicque in the next step

**Step 6** Open second `SSH` terminal by pressing "+" icon in the cloud shell and run following command

```
kubectl port-forward website 8080:80
```

!!! result
    This opens a tunnel and expose our application on port '80' to localhost:8080.


**Step 7** Test that website is running
Navigate back to the first terminal tab and run the following
```
curl localhost:8080
```
**Result:**

      ```
      <html>
      <head><title>403 Forbidden</title></head>
      <body>
      <center><h1>403 Forbidden</h1></center>
      <hr><center>nginx/1.21.0</center>
      </body>
      </html>
      ```

!!! success
    You've just learned how to use ConfigMaps.

You can also use the `preview` feature with the console

**Step 8** start a shell in the pod by using `exec`
```
kubectl exec website -it -- sh
```
Your terminal now should have `/#`

**Step 9** view the content of the folder where we added the volume
```
ls /etc/nginx/conf.d
```

**Step 10** check the content of the file in that folder and notice that it's the same as the config file for nginx we mounted.
```
cat /etc/nginx/conf.d/my-nginx-config.conf
```
**Step 11** Exit the shell
```
exit
```
Make sure you're back to the cloud shell terminal.

## 1.3 Using Secrets
### 1.3.1 Kubernetes Secrets

Kubernetes secrets allow users to define sensitive information outside of containers and expose that information to containers through environment variables as well as files within Pods. In this section we will declare and create secrets to hold our database connection information that will be used by Wordpress to connect to its backend database.

**Step 1** Open up two terminal windows. We will use one window to generate encoded strings that will contain our sensitive data. The other window will be used to create the secrets YAML declaration.

**Step 2** In the first terminal window, execute the following commands to encode our strings:
```
echo -n "admin" | base64
echo -n "t0p-Secret" | base64
```

**Step 3** create the secret. For this lab, we added the encoded values for you. Feel free to change the username and password, and replace the values in the file below:


```
cat <<EOF > app-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  username: YWRtaW4=
  password: dDBwLVNlY3JldA==
EOF
```
**Step 4** Create the secret:
```
kubectl create -f app-secrets.yaml
```
**Step 5** Verify secret creation and get details:

```
kubectl get secrets
```

**Step 6** Get details of this secret:
```
kubectl describe secrets/app-secrets
```

**Result:**

```
Name:         app-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  10 bytes
username:  5 bytes
```

**Step 7** Create a pod that will reference the secret as a volume.
Use either nano or vim to create secret-pod.yaml, and copy the following to it

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: alpine
      command: ["/bin/sh", "-ec", "export LOGIN=$(cat /etc/secret-volume/username);export PWD=$(cat /etc/secret-volume/password);while true; do echo hello $LOGIN your password is $PWD; sleep 10; done"]
      volumeMounts:
          - name: secret-volume
            mountPath: /etc/secret-volume
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
  restartPolicy: Never
```

**Step 8** View the logs of the pod.
```
kubectl logs secret-test-pod
```

Notice how we were able to pull the content of the secret using a command. Not very secure, right?

**Step 9** Delete both the pod and the secret
```
kubectl delete pod secret-test-pod
kubectl delete secret app-secret
```

!!! summary
    Secrets has been created and they not visiable when you view them via
    `kubectl describe` This does not make Kubernetes secrets secure as we have experienced. Additonal measures need to be in place to protect secrets.




## 1.4 Jobs
A Job creates one or more pods and ensures that a specified number of them successfully complete. A job keeps track of successful completion of a pod. When the specified number of pods have successfully completed, the job itself is complete. The job will start a new pod if the pod fails or is deleted due to hardware failure. A successful completion of the specified number of pods means the job is complete.

This is different from a replica set or a deployment which ensures that a certain number of pods are always running. So if a pod in a replica set or deployment terminates, then it is restarted again. This makes replica set or deployment as long-running processes. This is well suited for a web server, such as NGINX. But a job is completed if the specified number of pods successfully completes. This is well suited for tasks that need to run only once. For example, a job may convert an image format from one to another. Restarting this pod in replication controller would not only cause redundant work but may be harmful in certain cases.

Jobs are complementary to Replica Set. A Replica Set manages pods which are not expected to terminate (e.g. web servers), and a Job manages pods that are expected to terminate (e.g. batch jobs).

### 1.4.1 Non-parallel Job

Only one pod per job is started, unless the pod fails. Job is complete as soon as the pod terminates successfully.

Use the image "busybox" and have it sleep for 10 seconds and then complete.
Run your job to be sure it works.

**Step 1** Create a Job Manifest
```
cat <<EOF > busybox.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: busybox
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "20"]
      restartPolicy: Never
EOF
```

**Step 2** Run a Job
```
kubectl create -f busybox.yaml
```

**Step 3** Look at the job:
```
kubectl get jobs
```
**Output:**

```
NAME      DESIRED   SUCCESSFUL   AGE
busybox    1         0            0s
```

!!! result
    The output shows that the job is not successful yet.

**Step 4** Watch the pod status

```
kubectl get -w pods
```

**Output:**
```
NAME         READY     STATUS    RESTARTS   AGE
busybox-lk49x   1/1       Running   0          7s
busybox-lk49x   0/1       Completed   0         24s
```

!!! result
    It starts with pod for the job is `Running`.
    Then pod successfully exits after a few seconds and shows the `Completed` status.

**Step 5** Watch the job status again:

```
kubectl get jobs
```
**Output:**

```
NAME      COMPLETIONS   DURATION   AGE
busybox   1/1           21s        1m
```

**Step 6** Delete a Job

```
kubectl delete -f busybox.yaml
```

### 1.4.2 Parallel Job

Non-parallel jobs run only one pod per job. This API is used to run multiple pods in parallel for the job. The number of pods to complete is defined by `.spec.completions` attribute in the configuration file. The number of pods to run in parallel is defined by `.spec.parallelism` attribute in the configuration file. The default value for both of these attributes is 1.

The job is complete when there is one successful pod for each value in the range in 1 to `.spec.completions`. For that reason, it is also called as _fixed completion count_ job.

**Step 1** Create a Job Manifest
```
cat <<EOF > job-parallel.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: wait
spec:
  completions: 6
  parallelism: 2
  template:
    metadata:
      name: wait
    spec:
      containers:
      - name: wait
        image: ubuntu
        command: ["sleep",  "10"]
      restartPolicy: Never
EOF
```

!!! note
    This job specification is similar to the non-parallel job specification above.
    However it has two new attributes added: `.spec.completions` and `.spec.parallelism`.
    This means the job will be complete when six pods have successfully completed.
    A maximum of two pods will run in parallel at a given time.


**Step 2** Create a parallel job using the command:

```
kubectl apply -f job-parallel.yaml
```

**Step 3** Watch the status of the job as shown:

```
kubectl get -w jobs
```

**Output:**
```
NAME   COMPLETIONS   DURATION   AGE
wait   0/6           6s         6s
wait   1/6           12s        12s
wait   2/6           12s        12s
wait   3/6           24s        24s
wait   4/6           24s        24s
wait   5/6           36s        36s
wait   6/6           36s        36s
```

!!! results
    The output shows that 2 pods are created about every 12 seconds.

**Step 4**  In another terminal window, watch the status of pods created:
```
kubectl get -w pods -l job-name=wait
```
**Output:**
```
NAME         READY   STATUS      RESTARTS   AGE
wait-5blwm   0/1     Completed   0          17s
wait-stmk4   0/1     Completed   0          17s
wait-ts6xt   1/1     Running     0          5s
wait-xlhl6   1/1     Running     0          5s
wait-xlhl6   0/1     Completed   0          12s
wait-rq6z5   0/1     Pending     0          0s
wait-ts6xt   0/1     Completed   0          12s
wait-rq6z5   0/1     Pending     0          0s
wait-rq6z5   0/1     ContainerCreating   0          0s
wait-f85bj   0/1     Pending             0          0s
wait-f85bj   0/1     Pending             0          0s
wait-f85bj   0/1     ContainerCreating   0          0s
wait-rq6z5   1/1     Running             0          2s
wait-f85bj   1/1     Running             0          2s
wait-f85bj   0/1     Completed           0          12s
wait-rq6z5   0/1     Completed           0          12s
```
**Step 6** Once the job is completed, you can get the list of completed pods
```
kubectl get pods -l job-name=wait
```

Result:
```
NAME         READY   STATUS      RESTARTS   AGE
wait-5blwm   0/1     Completed   0          2m55s
wait-f85bj   0/1     Completed   0          2m31s
wait-rq6z5   0/1     Completed   0          2m31s
wait-stmk4   0/1     Completed   0          2m55s
wait-ts6xt   0/1     Completed   0          2m43s
wait-xlhl6   0/1     Completed   0          2m43s
```

**Step 5** Similarly, `kubectl get jobs` shows the status of the job after it
has completed:

```
kubectl get jobs
```

Result:
```
NAME   COMPLETIONS   DURATION   AGE
wait   6/6           36s        3m54s
```

**Step 6** Deleting a job deletes all the pods as well. Delete the job as:

```
kubectl delete -f job-parallel.yaml
```

## 1.5 Cron Jobs
A Cron Job is a job that runs on a given schedule, written in Cron format.
There are two primary use cases:

  * Run jobs once at a specified point in time
  * Repeatedly at a specified point in time

### 1.5.1 Create Cron Job

**Step 1** Create `CronJob` manifest that prints the current timestamp and the
message "`Hello World`" every minute.

```
cat <<EOF > cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: hello-cronpod
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello World!
          restartPolicy: OnFailure
EOF
```

**Step 2** Create the Cron Job as shown in the command:

```
kubectl create -f cronjob.yaml
```

**Step 3** Watch the status of the job as shown:
```
kubectl get -w cronjobs
```
**Output**:
```
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        <none>          6s
hello   */1 * * * *   False     1        5s              39s
hello   */1 * * * *   False     0        15s             49s
hello   */1 * * * *   False     1        5s              99s
hello   */1 * * * *   False     0        15s             109s
hello   */1 * * * *   False     1        6s              2m40s
hello   */1 * * * *   False     0        16s             2m50s
```

**Step 4** In another terminal window, watch the status of pods created:

```
kubectl get -w pods -l app=hello-cronpod
```
**Output**:
```
NAME                     READY   STATUS      RESTARTS   AGE
hello-1622584020-kc46c   0/1     Completed   0          118s
hello-1622584080-c2pcq   0/1     Completed   0          58s
hello-1622584140-hxnv2   0/1     Pending     0          0s
hello-1622584140-hxnv2   0/1     Pending     0          0s
hello-1622584140-hxnv2   0/1     ContainerCreating   0          0s
hello-1622584140-hxnv2   1/1     Running             0          1s
hello-1622584140-hxnv2   0/1     Completed           0          2s
```

**Step 5** Get logs from one of the pods:

```
kubectl logs hello-1622584140-hxnv2
```
**Output**:
```
Tue Jun  1 21:49:07 UTC 2021
Hello World!
```

**Step 6** Delete Cron Job
```
kubectl delete -f cronjob.yaml
```

## 1.6 Daemon Set
Daemon Set ensures that a copy of the pod runs on a selected set of nodes.
By default, all nodes in the cluster are selected. A selection critieria may be
specified to select a limited number of nodes.

As new nodes are added to the cluster, pods are started on them. As nodes are
removed, pods are removed through garbage collection.

The following is an example DaemonSet that runs a Prometheus exporter container
that used for collecting machine metrics from each node.

**Step 1** Create a DaemonSet manifest

```
cat <<EOF > daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-daemonset
spec:
  selector:
    matchLabels:
      tier: monitoring
      name: prometheus-exporter
  template:
    metadata:
      labels:
        tier: monitoring
        name: prometheus-exporter
    spec:
      containers:
      - name: prometheus
        image: prom/node-exporter
        ports:
        - containerPort: 80
EOF
```
**Step 2**  Run the following command to create the ReplicaSet and pods:
```
kubectl create -f daemonset.yaml --record
```

!!! note
    The `--record` flag will track changes made through each revision.

**Step 3** Get basic details about the DaemonSet:

```
kubectl get daemonsets/prometheus-daemonset
```
**Output:**
```
NAME                   DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE-SELECTOR   AGE
prometheus-daemonset   1         1         1         1            1           <none>          5s
```

**Step 4** Get more details about the DaemonSet:

```
kubectl describe daemonset/prometheus-daemonset
```
**Output:**
```
Name:           prometheus-daemonset
Selector:       name=prometheus-exporter,tier=monitoring
Node-Selector:  <none>
Labels:         <none>
Annotations:    deprecated.daemonset.template.generation: 1
                kubernetes.io/change-cause: kubectl create --filename=daemonset.yaml --record=true
Desired Number of Nodes Scheduled: 2
Current Number of Nodes Scheduled: 2
Number of Nodes Scheduled with Up-to-date Pods: 2
Number of Nodes Scheduled with Available Pods: 2
Number of Nodes Misscheduled: 0
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=prometheus-exporter
           tier=monitoring
  Containers:
   prometheus:
    Image:        prom/node-exporter
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  9m     daemonset-controller  Created pod: prometheus-daemonset-f4xl2
  Normal  SuccessfulCreate  2m18s  daemonset-controller  Created pod: prometheus-daemonset-w596z
```

**Step 5** Get pods in the DaemonSet:

```
kubectl get pods -l name=prometheus-exporter
```

**Output:**
```
NAME                         READY   STATUS    RESTARTS   AGE
prometheus-daemonset-f4xl2   1/1     Running   0          8m27s
prometheus-daemonset-w596z   1/1     Running   0          105s
```

**Step 6** Verify that the Prometheus pod was successfully deployed to the cluster nodes:

```
kubectl get pods -o wide
```

**Output:**
```
NAME                         READY   STATUS    RESTARTS   AGE     IP          NODE                                          NOMINATED NODE   READINESS GATES
prometheus-daemonset-f4xl2   1/1     Running   0          6m51s   10.0.0.19   gke-k8s-features-default-pool-73e09df7-m2lf   <none>           <none>
prometheus-daemonset-w596z   1/1     Running   0          9s      10.0.1.2    gke-k8s-features-default-pool-73e09df7-k7hj   <none>           <none>
```

!!! notes
    It is possible to Limit DaemonSets to specific nodes
    by changing the `spec.template.spec` to include a `nodeSelector` to matche `node` label.


**Step 7**  Delete a DaemonSet

```
kubectl delete -f daemonset.yaml
```


## 1.7 Cleaning Up

**Step 1** Delete the cluster
```
gcloud container clusters delete k8s-concepts
```
