# Deploy Kubernetes

In this Lab, we are going to:

* Deploy Single Node Kubernetes cluster using **kubeadm** on a Google Compute Engine node
* Deploy an application to  Kubernetes

## 1. Deploy Kubernetes and Calico with Kubeadm
In general to deploy Kubernetes with `kubeadm` it is required to use following official Kubernetes [documentation](https://kubernetes.io/docs/setup/independent/install-kubeadm/).



**Step 1** Set the Project ID in  Environment Variable: 

```
export PROJECT_ID=<project_id>
```


Here is how you can find you project_ID:

![Project ID](/images/porject_id_png.png "Locate project_id")


Set the project ID as default
```
gcloud config set project $PROJECT_ID
```

### 1.1 Create a VM, and ssh into it
**Step 1** Create the Compute instance

The compute instances in this lab will be provisioned using Ubuntu Server 20.04, which has good support for the containerd container runtime. 

```
gcloud compute instances create k8s-cluster \
--zone us-central1-c \
--machine-type=e2-standard-4 \
--can-ip-forward \
--image-family ubuntu-2004-lts \
--image-project ubuntu-os-cloud
```


**Step 2** SSH in to VM where we going to install Kubernetes:

```
gcloud compute ssh --zone "us-central1-c" "k8s-cluster"
```


### 1.2 Install Containerd

Docker has been deprecated from Kubernetes starting K8s 1.24, so we will need to intall containerd instaed

**Step 1** Update the apt package index and install containerd:

```
sudo su
apt update
```

```
apt install containerd
```

Pres Y, to install packages.

**Step 2** Enable systemd to start on reboot:

```
systemctl enable containerd
systemctl start containerd
systemctl status containerd
```

**Output:**
```
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-05-18 21:33:39 UTC; 8s ago
       Docs: https://containerd.io
   Main PID: 2106 (containerd)
      Tasks: 15
     Memory: 23.3M
     CGroup: /system.slice/containerd.service
             └─2106 /usr/bin/containerd
```
**Step 3** To interact with containerd Install nerdctl:

```
wget https://github.com/containerd/nerdctl/releases/download/v0.18.0/nerdctl-0.18.0-linux-amd64.tar.gz
tar zxvf nerdctl-0.18.0-linux-amd64.tar.gz nerdctl
mv nerdctl /usr/local/bin
```


**Step 4** Install CNI Plugins to test containerd can start containers:

```
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
mkdir -p /opt/cni/bin/
tar zxvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/
```

**Step 5** Run a Docker like command with Nerdctl to start test `ubuntu` container:

See Nerdctl CLI reference [here](https://github.com/containerd/nerdctl)

```
nerdctl run -d ubuntu bash
```

```
nerdctl ps -a
```

!!! sucess
    We can see our docker container been started few seconds ago.


### 1.3 Let iptables see bridged traffic
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```


### 1.4 Install kubeadm and prerequisite packages on each node

The next step is to install `kubeadm` and prerequisite packages as showed [here.](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

**Step 1** Deploy `kubeadm` and prerequisite packages


Update the apt package index and install packages needed to use the Kubernetes apt repository:


```
apt-get update && apt-get install -y apt-transport-https ca-certificates curl
```

Download the Google Cloud public signing key:

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Add the Kubernetes apt repository:

```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:

```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


**Step 2** Verify `kubeadm` version

```
kubeadm version
```
Output:
```
kubeadm version: &version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:44:24Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}
```

!!! result
		Latest available version of Kubernetes/kubeadm has been installed from [GitHub Kubernetes repo release page.](https://github.com/kubernetes/kubernetes/releases)


### 1.5 'Kubeadm init' the Master

**Run On the Master node only:**

**Step 1:** Build kubeadm Custom Config
```
cat <<EOF > kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.24.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF
```

Make sure the IP address was updated:
```
cat  kubeadm-config.yaml
```

!!! note
    To expose custom Config you can create a  kubeadm.conf and specify
		during `kubecadm init` execution. For instance:

			* ControllerManager configs
			* Custom Subnet
			* Custom version
			* Apiserver configs such as authentication, authorization and etc.


**Step 2:** Create a cluster

```
kubeadm init --config=kubeadm-config.yaml
```

!!! result
		Once the command completes, configure the KUBECONFIG env variable with the path to admin.conf (recommend adding it to your .bashrc):

```
  mkdir -p $HOME/.kube
  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config
  export KUBECONFIG=$HOME/.kube/config
```

Let's validate that the installation was successful. You should now be able to run kubectl commands and see that all cluster Pods
are running (except DNS one):

```
watch kubectl get pods --all-namespaces -o wide
```
To exit back to the terminal, press ctrl+c

### 1.6 Deploy Cilium CNI

**Step 1** Download Helm Package manager:

```
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
```

```
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
```

```
mv linux-amd64/helm /usr/local/bin/helm
```


**Step 1**  Setup Helm Cilium repository:

```
helm repo add cilium https://helm.cilium.io/
```

**Step 3**  Now lets deploy Cilium Networking

```
helm install cilium cilium/cilium --version 1.9.16 --namespace kube-system
```

Watch the Cilium/node pod for the master get created (hopefully successfully)

```
watch kubectl get pods --all-namespaces -o wide
```

### 1.7 Join worker node

If you have other nodes around you can run the 'kubeadm join ...' command from
the output of kubeadm init on each worker node (incl token).
Watch the calico/node pods get created for each worker node automatically.

```
e.g. kubeadm join --token ****
```

For this lab, we are creating a one node kubernetes clusters, so in order to be able to deploy applications on the same node as the control plane, we need to remove the taint that prevent such deployment.

```
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

### 1.8 Now lets create a test deployment with 2 replicas

```
kubectl create deployment nginx --replicas=2 --image=nginx --port=8080
```

Lets get some more detail about the deployment:

```
kubectl describe deployment nginx
kubectl get deployment nginx
```

And pods that has been created by `nginx` deployment:
```
kubectl get pods
```



**Congrats. Now you have a working Kubernetes+Calico cluster.**

### 2.1 Verify Kubernetes components deployed by kubeadm
#### 2.1.1 Check Kubernetes version

**Step 1** Verify that Kubernetes is deployed and working.

```shell
kubectl get nodes
```


!!! result
    * Kubernetes has single node for workload scheduling.
    * Kubernetes running version 1.24.0

!!! note
		At Kubernetes community, we define 3 types of Kubernetes releases:

		* Major (x.0.0)
		* Minor (x.x.0)
		* Patch (x.x.x)

!!! note
		At a single point of time, we develop the new "Major"/"Minor" version of Kubernetes (today - Kubernetes 1.21), and we support three existing releases as the "Patch" releases (today - 1.19.x, 1.20.x and 1.21.x).

#### 2.1.2 Verify Cluster default namespaces.
**Step 1** Verify namespaces created in K8s systems
```shell
$ kubectl get ns
NAME              STATUS   AGE
default           Active   5h50m
kube-node-lease   Active   5h50m
kube-public       Active   5h50m
kube-system       Active   5h50m
```

!!! info
    Namespaces are intendent to isolate groups/teams and give them access to
    a set of resources. They avoid name collisions between resources. Namespaces
    provides with a soft Multitenancy, meaning they not provide full isolation.

!!! result
    By default Kubernetes deployed by `kubeadm` starts with 4 namespaces:

    * `default` The default namespace for objects with no other namespace. When
    listing resources with the kubectl get command, we’ve never specified the
    namespace explicitly, so kubectl always defaulted to the default namespace,
    showing us just the objects inside that namespace.
    * `kube-system` The namespace for objects created by the Kubernetes system
    * `kube-public` Readable by all users, and mostly reserved for cluster usage.
    * `kube-node-lease` This namespace for the lease objects associated with each node which improves the performance of the node heartbeats as the cluster scales.


#### 2.1.3 Verify kubelet
**Step 1** Verify that `kubelet` installed in K8s Cluster:

```
systemctl -l | grep kubelet
systemctl status kubelet
```

!!! note
    Service and its config file can be found in `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`


**Step 2** Find manifests file for other master Node components:

Once `kubelet` is deployed, all the rest master node components are
deployed as a `static pods` on Kubernetes Master node. Setting `--pod-manifest-path=`
specifies from where to read Static Pod manifests used for spinning up the
control plane.

**Step 3** List K8s components manifest files that is going to be used for cluster deployment
and run as `Static Pods` by `kubelet`:

```
sudo ls /etc/kubernetes/manifests
```
```
etcd.yaml  kube-apiserver.yaml	kube-controller-manager.yaml  kube-scheduler.yaml
```

!!! result
    We see etcd, api-server, controller-manager and scheduler that has been
    used to deploy on this cluster and managed by `kubelet`.


**Step 4** Verify K8s Components deployed as containers on K8s:

```
kubectl get pods -n kube-system
```

```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6d7b4db76c-h242g   1/1     Running   0          27m
calico-node-gwnng                          1/1     Running   0          27m
coredns-74ff55c5b-5s7rp                    1/1     Running   0          5h59m
coredns-74ff55c5b-l6hd4                    1/1     Running   0          5h59m
etcd-k8s-cluster                           1/1     Running   0          5h59m
kube-apiserver-k8s-cluster                 1/1     Running   0          5h59m
kube-controller-manager-k8s-cluster        1/1     Running   0          5h59m
kube-proxy-f8647                           1/1     Running   0          5h59m
kube-scheduler-k8s-cluster                 1/1     Running   0          5h59m
```

!!! result
    * We can see that Kubernetes components: etcd, api-server, controller-manager
     and scheduler deployed on K8s cluster via kubelet.
    * Calico Networking including calico-etcd, calico-node, calico-policy-controller
    has been deployed as a last step of kubeadm installation
    * Or Cilium Networking containers


#### 2.1.4 Verify etcd database deployment.

**Step 1** Verify `etcd` config file

```
sudo cat /etc/kubernetes/manifests/etcd.yaml
```

**Step 2**  Overview `etcd` pod deployed on K8s cluster:
```
kubectl get pods -n kube-system | grep etcd
kubectl describe pods/etcd-k8s-cluste -n kube-system
```

!!! result
    * `etcd` has been deployed as a static pod.
    * Annotation `Priority Class Name:  system-node-critical` tells to K8s that this
    Pod is  critical and will have highest `QOS`.

**Step 3** Check the location of etcd db and snapshot dumps.

```
sudo ls /var/lib/etcd/member
```

!!! result
    The data directory has two sub-directories in it:

    * wal: write ahead log files are stored here.
    * snap: log snapshots are stored here.

    When first started, etcd stores its configuration into a data directory specified by the data-dir configuration parameter. Configuration is stored in the write ahead log and includes: the local member ID, cluster ID, and initial cluster configuration. The write ahead log and snapshot files are used during member operation and to recover after a restart.


#### 2.1.5 Verify `api-server` deployment on the K8s cluster.

**Step 1** Review configuration file:

```
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Step 2** Overview `api-server` pod and its parameters.
```
kubectl describe pods/kube-apiserver-k8s-cluster -n kube-system
```

#### 2.1.6 Verify `Controller-manager` and `scheduler` deployment.
**Step 1** `Controller-manager` and `scheduler` deployed on K8s cluster via kubelet
the same way `api-server`. Verify both configuration files and pods running on K8s Cluster.

!!! Summary
    * K8s is an orchestration system for containers. Since most of the k8s
    components are the `go` binaries that can be containerized, K8s has been
    designed to run itself. This makes system  itself HA, easily deployable,
    scaleable and upgradable.