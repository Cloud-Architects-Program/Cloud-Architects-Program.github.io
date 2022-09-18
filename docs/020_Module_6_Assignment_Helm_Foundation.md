Lab 4 Learning Helm

**Objective:**

  * Installing and Configuring the Helm Client
  * Deploy Kubernetes apps with Helm Charts
  * Learn Helm Commands
  * Learn Helm Repository
  * Create Helm chart
  * Learn Helm Plugins
  * Learn HelmFile


## Prerequisite

### Locate Assignment 4

**Step 1**  Clone `ycit020` repo with Kubernetes manifests, which going to use for our work:

```
cd ~/ycit020
# Alternatively: cd ~ & git clone https://github.com/Cloud-Architects-Program/ycit020
git pull
cd ~/ycit020/Assignment4/
ls
```

!!! result 
    You can see Kubernetes manifests and terraform configs


**Step 1** Go into your personal Google Cloud Source Repository:

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

**Step 3 (Optional)** If you terraform config is not working you can copy working config from `Assignment4` folder
to your `notepad-infrastructure` folder.

**Step 4** Create structure for Helm Deployments for supporting applications

```
cd ~/$MY_REPO/notepad-infrastructure
mkdir helm

cat <<EOF> helm/README.md
# Helm Files and values to deploy supporting applications
EOF
```


**Step 5** Copy Assignment 4 `deploy_a4` folder to your repo:

```
cd ~/$MY_REPO/
cp -r ~/ycit020/Assignment4/deploy_a4 deploy_ycit020_a4
ls deploy_ycit020_a4
```

!!! result
    You should see `k8s-manifest` folder

**Step 6** Create structure for `NotePad` Helm Chart locations

```
cd ~/$MY_REPO/deploy_ycit020_a4
mkdir helm

cat <<EOF> helm/README.md
# Helm Chart, HelmFiles and values to deploy `NotePad` application.
EOF
```


### Reserve Static IP addresses

In the next few assignments we might deploy many applications and so we will require expose them using Ingress.

When you create an Ingress object, you get a stable external IP address that clients can use to access your Services and in turn, your running containers. The IP address is stable in the sense that it lasts for the lifetime of the Ingress object. If you delete your Ingress and create a new Ingress from the same manifest file, you are not guaranteed to get the same external IP address.

We would like to have for each student a  a permanent IP address that stays the same across deleting your Ingress and creating a new one

For that you must reserve a `Regional` static `External` IP address and provide it teacher, so we can setup `A Record` on the DNS for your behalf.

Reference: 

1. [Reserving a static external IP address](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address)
2. Terraform resource [google_compute_address](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_address)

**Step 1:** Create folder for static ip terraform configuration:

```
cd ~/$MY_REPO/
mkdir ip-infrastructure
```

!!! important
    The reason we creating a new folder for static_ip creation is because we don't want to destroy or delete this IP throughout our training.

**Step 1:** Create Terraform configuration:

Set you current `PROJECT_ID` value here:

```
export PROJECT_ID=<YOUR_PROJECT_ID>
```

Declare Provider:


```
cd ~/$MY_REPO/ip-infrastructure
cat << EOF>> provider.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 3.70.0"
    }
  }
}
EOF
```

Configure global address resource:

```
cat << EOF>> static_ip.tf
provider "google" {
  project = var.gcp_project_id
}

resource "google_compute_address" "regional_external_ip" {
  provider      = google
  name          = "static-ingres-ip"
  address_type  = "EXTERNAL"
  region        = "us-central1"
}
EOF
```

Configure variables:

```
cat <<EOF> variables.tf
variable "gcp_project_id" {
  type        = string
  description = "The GCP Seeding project ID"
  default     = ""
}
EOF
```

Configure `tfvars`:

```
gcp_project_id = "$PROJECT_ID"
```

Configure outputs:

```
cat <<EOF >> outputs.tf
output "addresses" {
  description = "Global IPv4 address for proxy load balancing to the nearest Ingress controller"
  value       = google_compute_address.regional_external_ip.address
}

output "name" {
  description = "Static IP Name"
  value       = google_compute_address.regional_external_ip.name
}
EOF
```

**Step 2:** Apply Terraform configuration:

Initialize:

```
terraform init
```

Plan and Deploy Infrastructure:

```
terraform plan -var-file terraform.tfvars
terraform apply -var-file terraform.tfvars
```

Set variable:

```
cd ~/$MY_REPO/ip-infrastructure
export STATIC_IP_ADDRESS=$(terraform output | grep 'addresses' |awk '{ print $3}')
export STATIC_IP_NAME=$(terraform output | grep 'name' |awk '{ print $3}')
```

Create readme:

```
cat <<EOF> README.md

Generated Static IP for Ingress:
 
    * Name - $STATIC_IP_NAME # Used Ingress Manifest
    * Address - $STATIC_IP_ADDRESS # Used to Configure DNS A Record
EOF
```

!!! important

    Add information about your static_ip_address in column 3 under your name in this spreadsheet:
    [https://docs.google.com/spreadsheets/d/1XV-wW9Z8KvfWo4io11V_ax2__OImF27Rmlfbyj03T0I/edit?usp=sharing](https://docs.google.com/spreadsheets/d/1XV-wW9Z8KvfWo4io11V_ax2__OImF27Rmlfbyj03T0I/edit?usp=sharing)



**Step 6** Commit `deploy` folder using the following Git commands:

```
cd ~/$MY_REPO
git status
git add .
git commit -m "adding documentation for ycit020 assignment 4"
```


### Commit to Repository


**Step 1** Commit `ip-infrastructure` and `helm` folders using the following Git commands:

```
cd ~/$MY_REPO
git status 
git add .
git commit -m "Assignement 4"
```

**Step 2** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```




## 1 Install and Configure Helm Client

### 1.1 Install Helm 3


GCP Cloud Shell comes with many common tools pre-installed including `helm`.


**Step 1:** Verify and validate the version of `Helm` that is installed:

```
helm --version
```

**Output:**
```
version.BuildInfo{Version:"v3.5.0"}
```

!!! result
    Helm 3 is installed


!!! note
    Until November 2020, two different major versions of Helm were actively maintained. The current stable major version of Helm is version 3. 
    
!!! note
    Helm follows a versioning convention known as [Semantic Versioning](https://semver.org/) (SemVer). In Semantic Versioning, the version number conveys meaning about what you can expect in the release. Because Helm follows this specification, users can expect certain things out of releases simply by carefully reading the version number. At its core, a semantic version has three numerical components and an optional stability marker (for alphas, betas, and release candidates). Here are some examples:

    * v1.0.0
    * v3.3.2

    SemVer represents format of  `X.Y.Z`, where `X` is a major version, `Y` is a minor version and `Z` is a patch release:

    * The major release number tends to be incremented infrequently. It indicates that major changes have been made to Helm, and that some of those changes may break compatibility with previous versions. The difference between Helm 2 and Helm 3 is substantial, and there is work necessary to migrate between the versions.
    
    * The minor release number indicates feature additions. The difference between 3.2.0 and 3.3.0 might be that a few small new features were added. However, there are no breaking changes between versions. (With one caveat: a security fix might necessitate a breaking change, but we announce boldly when that is the case.)
    
    * The patch release number indicates that only backward compatible bug fixes have been made between this release and the last one. It is always recommended to stay at the latest patch release.

**(OPTIONAL) Step 2:** If you want to use specific version of `helm` or want to install `helm` in you local machine (On macOS and Linux,) use following [link](https://helm.sh/docs/intro/install/#from-script) to install Helm. 

The usual sequence of commands for installing this way is as follows:

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

The preceding commands fetch the latest version of the get_helm.sh script, and then use that to find and install the latest version of Helm 3.


Alternatively, you can download latest binary of you OS choice [here](https://github.com/helm/helm/releases).


### 1.3 Deploy GKE cluster

We going to reuse Terraform configuration to deploy our GKE Cluster. And we assume that `terraform.tfvars` has been properly configured already with required values.

**Step 1:** Locate Terraform Configuration directory.

```
cd ~/$MY_REPO/notepad-infrastructure
```

**Step 2:** Initialize Terraform Providers
s
```
terraform init
```




**Step 3:**  Increase GKE Node Pool VM size


```
edit terraform.tfvars
```

Update `gke_pool_machine_type` from `e2-small` to `e2-highcpu-4`, to support larger workloads.

**Step 4:** Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

**Step 5:** Create GKE Cluster and Node Pool:

```
terraform apply -var-file terraform.tfvars
```

```
!!! result
    GKE Clusters has been created
```

### 1.4 Configure Helm

Helm interacts directly with the Kubernetes API server. For that reason, Helm needs to be able to connect to a Kubernetes cluster. Helm attempts to do this automatically by reading the same configuration files used by `kubectl`.

Helm will try to find this information by reading the environment variable $KUBECONFIG.

**Step 1** Authenticate to the cluster.

export STUDENT_NAME=<YOUR_PROJECT_ID>
```
gcloud container clusters get-credentials gke-$STUDENT_NAME-notepad-dev --region us-central1 
``` 

**Step 2** Test that `kubectl` client connected to GKE cluster:

```
kubectl get pods
```

**Output:**
```
No resources found in default namespace.
```


**Step 3** Test that helm client connected to GKE cluster:

```
helm list
```
**Output:**
```
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
```

!!! result
    As expected there are no Charts has been Installed on our cluster yet


!!! summary
    Helm 3 has been installed and configured to work with our cluster. Let's deploy some charts!


## 2 Basic Helm Chart Installation

### 2.1 Searching Chart

A Helm chart is an package that can be installed into your Kubernetes cluster. During chart development, you will often just work with a chart that is stored on your local system and later pushed to GitHub.

But when it comes to sharing charts, Helm describes a standard format for indexing and sharing information about Helm `charts`. A Helm chart `repository` is simply a set of files, reachable over the network, that conforms to the Helm specification for indexing packages.


There are huge number of chart repositories on the internet. The easiest way to find the popular repositories is to use your web browser to navigate to the [Artifact Hub](https://artifacthub.io/). There you will find thousands of Helm charts, each hosted on an appropriate repository.


!!! deprecation
    In the past all charts were located and maintained by Helm Kubernetes Community Git repositories,
    known as https://github.com/kubernetes/charts, and had 2 types:

    * [Stable](https://github.com/kubernetes/charts/tree/master/incubator)
    * [Incubator](https://github.com/kubernetes/charts/tree/master/incubator)

    All the charts were rendered in Web UI via [Helm Hub](https://hub.helm.sh) page.

    While the central Git Repository to maintain charts was great idea, with fast growing Helm popularity, it become hard to impossible to manage
    and maintain it by small group of maintainers, as they had to aprove hundreds of PR per day and frustrating for chart contributors as they had to wait several weeks their PR to be reviewed and approved.

    As a result GitHub project for Helm `stable` and `incubator` charts as well as [Helm Hub] has been deprecated and archived.
    All the charts are now maintained by independent contributors in their subsequent repo's. (e.g vault, is under hashicorp/vault-helm repo)
    and the central place to find all active and official Charts can be foind in [Artifact Hub](https://artifacthub.io/).

!!! important
    Helm 2 came with a Helm repository installed by default. The `stable` chart repository was at one time the official source of production-ready Helm charts.
    As we discussed above  `stable` chart repository has been now deprecated.
    
    In Helm 3, there is no `default` repository. Users are encouraged to use the Artifact Hub to find what they are looking for and then add their preferred repositories.


**Step 1:** Helm provides native way to search charts from CLI in [artifacthub.io](artifacthub.io)

```
cd ~/$MY_REPO/notepad-infrastructure/helm
```
```
helm search hub drupal
```

**Output:**
```
https://artifacthub.io/packages/helm/bitnami/drupal	10.2.30      	9.2.3      	One of the most versatile open source content m...
https://artifacthub.io/packages/helm/cetic/drupal 	0.1.0        	1.16.0     	Drupal is a free and open-source web content ma..
```

!!! result
    Search is a good way to find existing Helm packages

**Step 2:** Use the link to navigate to Artifact Hub Drupal chart:

```
https://artifacthub.io/packages/helm/bitnami/drupal
```

!!! note
    Drupal is OSS content management systems that can be installed this days on K8s.


**Result:** The Artifact Page opened with information about the chart, it's parameters and how to use the chart.

**Step 3:** Review Github source Code of the chart:

```
https://github.com/bitnami/charts/tree/master/bitnami/drupal
```

**Result:** Github page with Drupal Helm Chart itself, where you can browse the `templates`, `values.yaml`, `Chart.yaml` to
understand how this chart is actually working and if you have any issues with the chart this would be the right location to open the issue
or send a PR with new feature if you decide to contribute.

### 2.2 Adding a Chart Repository

Once you found a chart, it's logical to install it. However first step you need to do is to add a Chart Repository.

**Step 1:** Adding a Helm chart is done with the `helm repo add` command:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

!!! note
    [Bitnami](https://bitnami.com/) is company well known to package application for Any Platforms and Cloud environments. With popularity of Helm, Bitnami developers were among the core contributors who designed the Helm repository system. They have contributed to the establishment of Helm’s best practices for chart development and have written many of the most widely used charts. Bitnami is now part of VMware, provides IT organizations with an enterprise offering that is secure, compliant, continuously maintained and customizable to your organizational policies.

**Step 2:** Now, we can verify that the Bitnami repository exists by running a `helm repo list` command:
    
```
helm repo list
```

**Output:**
```
NAME    URL                               
bitnami https://charts.bitnami.com/bitnami
```

!!! result
    This command shows us all of the repositories installed for Helm. Right now, we see only the Bitnami repository that we just added.


### 2.3 Helm Chart Installation

**Step 1:** At very minimum, installing a chart in Helm requires just 2 pieces of information: the `name` of the installation and the `chart` you want to install:

```
helm install mywebsite bitnami/drupal
```

**Output:**

```
NAME: mywebsite
LAST DEPLOYED: Sun Aug  8 08:25:29 2021
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*******************************************************************
*** PLEASE BE PATIENT: Drupal may take a few minutes to install ***
*******************************************************************

1. Get the Drupal URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace prometheus -w mywebsite-drupal'

  export SERVICE_IP=$(kubectl get svc --namespace prometheus mywebsite-drupal --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "Drupal URL: http://$SERVICE_IP/"

2. Get your Drupal login credentials by running:

  echo Username: user
  echo Password: $(kubectl get secret --namespace prometheus mywebsite-drupal -o jsonpath="{.data.drupal-password}" | base64 --decode)
```

!!! result
    When using Helm, you will see that output for each installation. A good chart would provide helpful `notes` on how to connect to the deployed solution.

!!! tip
    You can get `notes` information any time after helm installation using `helm get notes mywebsite` command.

**Step 2:** Follow your Drupal Helm Chart `notes` Instruction to access Website

```
1. Get the Drupal URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace test -w mywebsite-drupal'

  export SERVICE_IP=$(kubectl get svc --namespace test mywebsite-drupal --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "Drupal URL: http://$SERVICE_IP/"
```


!!! success
    We can access `My blog` website and login with provider user and password.
    You can start your own blog now, that runs on Kubernetes

**Step 3:** List deployed Kubernetes resources:

```
kubectl get all --namespace test
kubectl get pvc --namespace test
```

!!! summary
    Our Drupal website consist of MariaDB `statefulset`, Drupal `deployment`, 2 `pvc`s and 
    `services`. `Mywebsite` Drupal service is type `LoadBalancer` and that's how we able to access it.

**Step 4:** List installed chart:

```
helm list
```

**Output:**
```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
mywebsite       default         1               2021-08-08 08:25:29.625862 -0400 EDT    deployed        drupal-10.2.30          9.2.3
```

!!! note
    Like other commands, helm list is `namespace` aware. By default, Helm uses the `namespace` your Kubernetes configuration file sets as the default. Usually this is the namespace named `default`.

**Step 5:** Deploy same chart in namespace `test`: 

```
kubectl create ns test
helm install --namespace test mywebsite bitnami/drupal --create-namespace
```

!!! tip
    By adding `--create-namespace`, indicates to Helm that we acknowledge that there may not be a namespace with that name already, and we just want one to be created.

List deployed chart in namespace `test`: 

```
helm list --namespace test
```

!!! summary
    In Helm 2, instance names were cluster-wide. You could only have an instance named `mywebsite` once per cluster. 
    
    In Helm 3, naming has been changed. Now instance names are scoped to Kubernetes namespaces. We could install 2 instances named `mywebsite` as long as they each lived in a different namespace.

**Step 5:** Cleanup Helm Drupal deployments using `helm uninstall` command:


```
helm uninstall mywebsite
helm uninstall mywebsite -n test 
```

```
kubectl delete ns test
```


## 3 Advanced Helm Chart Installation


### 3.2.1 Deploy NGINX Ingress Chart with Custom Configuration

Let's deploy another Helm application on our Kubernetes cluster: [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/) - is an Ingress controller for Kubernetes using NGINX as a
reverse proxy and load balancer.

In the previous classes we used Google implementation of Ingress - [GKE Ingress for HTTP(S) Load Balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress).
While this solutions provides managed Ingress experience and advanced features like Cloud Armor, DDoS protection and Identity aware proxy.

Ingress Nginx Controller is popular solution and has a lot of features and integrations. If you want to deploy Kubernetes Application on different cloud providers or
On-prem the same way, Nginx Ingress Controller becomes a default option.


Our task is to configure Ingress Nginx using `type:LoadBalancer` with Regional Static IP configured in the section [### Reserve Static IP addresses](#reserve-static-ip-addresses).
Additionally we want to **enable metrics** service that will fetch Prometheus monitoring metrics from ingress and **disable admissionWebhooks** configuration.

**Step 1:** Let's search `ingress-nginx` in [artifacthub.io](artifacthub.io)

```
helm search hub ingress-nginx
```

**Output:**
```
URL                                                                  CHART VERSION   APP VERSION     DESCRIPTION                                       
https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx      4.0.0           1.0.0-beta.1    Ingress controller for Kubernetes using NGINX 
https://artifacthub.io/packages/helm/api/ingress-nginx                3.29.1          0.45.0          Ingress controller for Kubernetes using NGINX 
```

!!! result
    We going to select the first `ingress-nginx` chart that is maintained by Kubernetes Community


**Step 2:** Review Hub Page details about this chart and locate information how to add repository:

```
https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx
```

**Result:** The Artifact Page opened with information about the chart, it's parameters, and how to use the chart itself


**Step 3:** Add `ingress-nginx`  Repo:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update  
```

**Step 4:** Oftentimes, searching is a useful way to find not only what `charts` can be installed, but what `versions` are available:

```
helm search repo ingress-nginx --versions | head
```

**Output:**
```
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
ingress-nginx/ingress-nginx     3.35.0          0.48.1          Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     3.34.0          0.47.0          Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     3.33.0          0.47.0          Ingress controller for Kubernetes using NGINX a...
....
```

!!! note
    By default, Helm tries to install the latest `stable` release of a chart, but you can override this behavior and install a specific version of a chart. Thus it is often useful to see not just the summary info for a chart, but exactly which versions exist for a chart. Every new version of the chart can be bring fixes and new changes, so for production use it's better to go with tested version and pin installation version.

!!! summary 
    We going with latest listed official version of the chart `3.35.0` of `ingress-nginx` Chart.


### 3.2.2 Download and inspect Chart locally

**Step 1:** Pull `ingress-nginx` Helm Chart of specific version to Local filesystem:

```
cd ~/$MY_REPO/notepad-infrastructure/helm
helm pull ingress-nginx/ingress-nginx --version 3.35.0
tar -xvzf  ingress-nginx-3.35.0.tgz
```

**Step 2:**  See the tree structure of the chart

```
sudo apt-get install tree
tree -L 2 ingress-nginx
```

**Output:**
```
ingress-nginx
├── CHANGELOG.md
├── Chart.yaml
├── OWNERS
├── README.md
├── ci
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── admission-webhooks
│   ├── clusterrole.yaml
│   ├── clusterrolebinding.yaml
│   ├── controller-configmap-addheaders.yaml
│   ├── controller-configmap-proxyheaders.yaml
│   ├── controller-configmap-tcp.yaml
│   ├── controller-configmap-udp.yaml
│   ├── controller-configmap.yaml
│   ├── controller-daemonset.yaml
│   ├── controller-deployment.yaml
│   ├── controller-hpa.yaml
│   ├── controller-ingressclass.yaml
│   ├── controller-keda.yaml
│   ├── controller-poddisruptionbudget.yaml
│   ├── controller-prometheusrules.yaml
│   ├── controller-psp.yaml
│   ├── controller-role.yaml
│   ├── controller-rolebinding.yaml
│   ├── controller-service-internal.yaml
│   ├── controller-service-metrics.yaml
│   ├── controller-service-webhook.yaml
│   ├── controller-service.yaml
│   ├── controller-serviceaccount.yaml
│   ├── controller-servicemonitor.yaml
│   ├── default-backend-deployment.yaml
│   ├── default-backend-hpa.yaml
│   ├── default-backend-poddisruptionbudget.yaml
│   ├── default-backend-psp.yaml
│   ├── default-backend-role.yaml
│   ├── default-backend-rolebinding.yaml
│   ├── default-backend-service.yaml
│   ├── default-backend-serviceaccount.yaml
│   └── dh-param-secret.yaml
└── values.yaml
```

!!! Result 
    Typical structure of the helm chart, where:
    
    *  The `Chart.yaml` file contains metadata and some functionality controls for the chart
    *  `templates` folder contains all `yaml` manifests in [Go templates](https://pkg.go.dev/text/template) and used to generate Kubernetes manifests
    *  `values.yaml` contains `default` values applied during the rendering.
    *  The `NOTES.txt` file is a special template. When a chart is installed, the NOTES.txt template is rendered and displayed rather than being installed into a cluster.



### 3.2.2 Helm Template

As a reminder our goal it customize Nginx deployment to configure Ingress Nginx using `type:LoadBalancer` with Regional Static IP configured in the section [### Reserve Static IP addresses](#reserve-static-ip-addresses).
Additionally we want to **enable metrics** service that will fetch Prometheus monitoring metrics from ingress and **disable admissionWebhooks** configuration.

**Step 1:** Review `values.yaml`

```
cd ~/$MY_REPO/notepad-infrastructure/helm/ingress-nginx
edit values.yaml
```

!!! result
    It is pretty confusing at this point to understand what this chart is going to deploy,
    it would of been great to render to YAML format first.

**Step 2:** Execute `helm template` command and store output in `render.yaml`

```
helm template ingress-nginx/ingress-nginx --version 3.35.0  > render.yaml
```

!!! note
    During helm template, Helm never contacts a remote Kubernetes server, hence the chart only has access to default Kubernetes kinds.
    The template command always acts like an installation.

Review rendered values:

```
edit render.yaml
```

```
grep "Source" render.yaml
```

!!! result 
    `helm template` designed to isolate the template rendering process of Helm from the installation. 
    The template command performs following phases of Helm Lifecycle:

    * loads the chart
    * determines the values
    * renders the templates
    * formats to YAML


**Step 3:** Let's first create a configuration that will disable `admissionWebhooks`.
Review looking in `values.yaml` for admissionWebhooks configuration: 

```
grep -A10 admissionWebhooks values.yaml
```

**Output:**
```
  admissionWebhooks:
    annotations: {}
    enabled: true
    failurePolicy: Fail
    # timeoutSeconds: 10
    port: 8443
    certificate: "/usr/local/certificates/cert"
    key: "/usr/local/certificates/key"
    namespaceSelector: {}
    objectSelector: {}
```

!!! result
    admissionWebhooks `enabled: true`

**Step 3:** Let's create `custom_values.yaml` file with  admissionWebhooks disabled

```
cat << EOF>> custom_values.yaml
controller:
  admissionWebhooks:
    enabled: false
EOF
```

Execute `helm template` and store output in new file `render2.yaml`:

```
helm template ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0  > render2.yaml
```

Now comparing this 2 render files you can see the what change in configuration has resulted in the rendered file:

```
diff render.yaml render2.yaml
```

This is the YAML files that will be generated after this change:

```
grep "Source" render2.yaml
```

**Output:**
```
# Source: ingress-nginx/templates/controller-serviceaccount.yaml
# Source: ingress-nginx/templates/controller-configmap.yaml
# Source: ingress-nginx/templates/clusterrole.yaml
# Source: ingress-nginx/templates/clusterrolebinding.yaml
# Source: ingress-nginx/templates/controller-role.yaml
# Source: ingress-nginx/templates/controller-rolebinding.yaml
# Source: ingress-nginx/templates/controller-service.yaml
# Source: ingress-nginx/templates/controller-deployment.yaml
```

**Step 4:** Now, let's add a custom Configuration to `controller-service.yaml` to use Static `loadBalancerIP`, 
we've configured in step [### Reserve Static IP addresses](#reserve-static-ip-addresses). First we need to locate 
the correct service parameters:

```
grep -A20 "service:" values.yaml 
```

!!! output
    Shows that we have 4 different services in values file

Let's narrow our search:

```
grep -A20 "service:" values.yaml | grep  "List of IP address"
```

**Output:**
```
    ## List of IP addresses at which the controller services are available
      ## List of IP addresses at which the stats-exporter service is available
    ## List of IP addresses at which the default backend service is available
```

**Output:**
```
  service:
    enabled: true

    annotations: {}
    labels: {}
    # clusterIP: ""

    ## List of IP addresses at which the controller services are available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []

    # loadBalancerIP: ""
    loadBalancerSourceRanges: []
```

!!! result
    It seems controller is first `service:` in values.yaml and currently is  it has `loadBalancerIP` section commented



**Step 5:** Let's add `loadBalancerIP` IP configuration to `custom_values.yaml`

Set variable:

```
cd ~/$MY_REPO/ip-infrastructure
export STATIC_IP_ADDRESS=$(terraform output | grep 'addresses' |awk '{ print $3}')
echo $STATIC_IP_ADDRESS
```

Add custom values:
```
cd ~/$MY_REPO/notepad-infrastructure/helm/ingress-nginx
cat << EOF>> custom_values.yaml
  service:
    loadBalancerIP: $STATIC_IP_ADDRESS
EOF
```

Our final `custom_values.yaml` should look as following:

```
controller:
  admissionWebhooks:
    enabled: false
  service:
    loadBalancerIP: 35.X.X.X
  metrics:
    enabled: true
```


Execute `helm template` and store output in new file `render3.yaml`:

```
helm template ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0  > render3.yaml
```

Show the difference in rendered file:

```
diff render2.yaml render3.yaml
```

**Output:**
```
>   loadBalancerIP: 35.X.X.X
```


Ensure that this change is in fact belongs to `controller-service.yaml`:

```
grep -C16  "loadBalancerIP:" render3.yaml
```

!!! result
    We configured correct service that will Ingress Controller service with `type: Loadbalancer` and 
    static IP with previously generated with terraform.


!!! summary
    `helm template` is great tool to render you charts to YAML.

!!! tip
    You can use Helm as packaging you charts only. And then use  `helm template` to generate actual YAMLs and apply them with `kubectl apply`
    
### 3.2.3 Dry-Runs

Before applying configuration to Kubernetes Cluster it is good idea to Dry-Run it for Errors. This is especially important if you doing Upgrades.

The dry-run feature provides Helm users a way to debug the output of a chart before it is sent on to Kubernetes. With all of the templates rendered, 
you can inspect exactly what would have been submitted to your cluster. And with the release data, you can verify that the release would have been created as you expected.

Here is some of the `dry-run` working principals:

  * `--dry-run` mixes non-YAML information with the rendered templates. This means the data has to be cleaned up before being sent to tools like kubectl.
  * A `--dry-run` on upgrade can produce different YAML output than a --dry-run on install, and this can be confusing.
  * It contacts the Kubernetes API server for validation, which means Helm has to have Kubernetes credentials even if it is just used to --dry-run a release.
  * It also inserts information into the template engine that is cluster-specific. Because of this, the output of some rendering processes may be cluster-specific.


Difference between `dry-run` and `template` is that the `dry-run` contacts Kubernetes API. It is good idea to use `dry-run` prior deployment and upgrade as it creates mutated output, while `template`
doesn't contacts API and can to pure rendering.


**Step 1:** Execute our deployment with `dry-run` command:


```
helm install ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --dry-run
```

**Output:**

```
NAME: ingress-nginx
LAST DEPLOYED: Tue Aug 10 07:22:56 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: ingress-nginx/templates/controller-serviceaccount.yaml
........
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace default get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
...
If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

At the top of the output, it will print some information about the release in our case it tells what phase of the release it is in (pending-install),
and the revision number. Next, after the informational block, all of the rendered templates are dumped to standard output. Finally, at the bottom of the dry-run output,
Helm prints the user-oriented release notes:


!!! note
    `--dry-run` dumps the output validates, but doesn't deploy actual chart.


**Step 2:** Finally let's deploy Nginx Ingress Charts with our parameters in `ingress-nginx` namespace


```
kubectl create ns ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx
```


**Step 3:** Verify Helm Deployment:

```
helm list
```

Follow Installation note and verify Ingress-Controller Service:

```
kubectl --namespace ingress-nginx get services -o wide ingress-nginx-controller
```

**Output:**
```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   172.10.0.213   35.192.101.76   80:30930/TCP,443:30360/TCP   3m9s
```

### 3.2.4 Deploy Drupal using Nginx Ingress Controller

Let's test our newly configured `ingress-nginx-controller` and expose `drupal` application using ingress instead of `LoadBalancer`. 


Check if STATIC_IP_ADDRESS variable is set:

```
echo $STATIC_IP_ADDRESS
```

If not set it with you Static IP address value.


```
cd ~/$MY_REPO/notepad-infrastructure/helm/
kubectl delete pvc data-mywebsite-mariadb-0
cat << EOF>> drupal_ing_values.yaml
ingress:
  annotations: {kubernetes.io/ingress.class: "nginx"}
  enabled: true
  hostname: $STATIC_IP_ADDRESS.nip.io
  path: /
  pathType: ImplementationSpecific
EOF
```


!!! note
    Here we using `nip.io` domain that provides simple wildcard DNS for any IP Address,
    however if you provided you STATIC_IP to teacher, they will generate following DNS,
    and you can replace `hostname:` with value `$student-name.cloud-montreal.ca`

```
helm install  mywebsite bitnami/drupal --values drupal_ing_values.yaml 
```

```
1. Get the Drupal URL:

  You should be able to access your new Drupal installation through

  http://35.X.X.X.nip.io/
```

!!! success
    Drupal site is now accessible via Ingress. Our Nginx Ingress Controller has been setup correctly!


Uninstall Drupal:
```
helm uninstall mywebsite
kubectl delete pvc data-mywebsite-mariadb-0
```

### 3.2.5  Using  `--set` values and Upgrading Charts

In addition to `--value` option, there is a second flag that can be used to add individual parameters to an `install` or `upgrade`. 
The `--set` flag takes one or more values directly. They do not need to be stored in a YAML file.


**Step 1:** Let's update our `ingress-nginx` release with new parameter `--set controller.image.pullPolicy=Always`, but first we want to render template with and without parameter to see what the change will be applied:


```
helm template ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx  > render4.yaml
```

```
helm template ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx --set controller.image.pullPolicy=Always > render5.yaml
```

```
diff render4.yaml render5.yaml
```

Output:

```
<           imagePullPolicy: IfNotPresent
---
>           imagePullPolicy: Always
```

**Step 2:** Let's apply update to our `ingress-nginx` release. For that we going to use `helm upgrade` command:


```
helm upgrade ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx --set controller.image.pullPolicy=Always 
```

Verify that upgrade was successful:

```
helm list --namespace ingress-nginx
```

**Output:**
```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                APP VERSION
ingress-nginx   ingress-nginx   2               2021-08-10 10:48:35.265506 -0400 EDT    deployed        ingress-nginx-3.35.0 0.48.1 
```

!!! result
    We can see that Revision change to `2`

!!! note
    When we talk about upgrading in Helm, we talk about upgrading an installation, not a chart. An installation is a particular instance of a chart in your cluster. 
    When you run `helm install`, it creates the installation. To modify that installation, use `helm upgrade`.
    This is an important distinction to make in the present context because upgrading an installation can consist of two different kinds of changes:
      
        * You can upgrade the version of the chart
        
        * You can upgrade the configuration of the installation

    In this case we upgrading the configuration of the installation.


!!! extra
    If you interested upgrade chart version of ingress-nginx to the next available release or `3.34.0`, give it a try.

### 3.2.5  Listing Releases, History, Rollbacks

**Step 1:** Let's see if we can upgrade our release with wrong value `pullPolicy=NoSuchPolicy` that doesn't exist on Kubernetes.
We will first run in `dry-run` mode

```
helm upgrade ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx --set controller.image.pullPolicy=NoSuchPolicy --dry-run
```

!!! result
    Release "ingress-nginx" has been upgraded. Happy Helming!

Let's now apply same config without `--dry-run` mode:

```
helm upgrade ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx --set controller.image.pullPolicy=NoSuchPolicy
```

**Output:**
```
Error: UPGRADE FAILED: cannot patch "ingress-nginx-controller" with kind Deployment: Deployment.apps "ingress-nginx-controller" is invalid: spec.template.spec.containers[0].imagePullPolicy: Unsupported value: "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
```

!!! failed
    As the error message indicates, a pull policy cannot be set to NoSuchPolicy. This error came from the Kubernetes API server, which means Helm submitted the manifest, 
    and Kubernetes rejected it. So our release should be in a failed state.


Verify with `helm list` to confirm `failed` state:

```
helm list
```

**Output:**
```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS  CHART                APP VERSION
ingress-nginx   ingress-nginx   3               2021-08-10 11:03:05.224405 -0400 EDT    failed  ingress-nginx-3.35.0 0.48.1   
```

**Step 2:** List all releases with `helm history`:

```
helm history ingress-nginx -n ingress-nginx
```

```
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Tue Aug 10 08:47:32 2021        superseded      ingress-nginx-3.35.0    0.48.1          Install complete
2               Tue Aug 10 10:48:35 2021        deployed        ingress-nginx-3.35.0    0.48.1          Upgrade complete
3               Tue Aug 10 11:03:05 2021        failed          ingress-nginx-3.35.0    0.48.1          Upgrade "ingress-nginx" failed: cannot patch "ingress-nginx-controller" with kind Deployment: Deployment.apps "ingress-nginx-controller" is invalid: spec.template.spec.containers[0].imagePullPolicy: Unsupported value: "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
```

!!! info
    During the life cycle of a release, it can pass through several different statuses. Here they are, approximately in the order you would likely see them:

    * `pending-install` Before sending the manifests to Kubernetes, Helm claims the installation by creating a release (marked version 1) whose status is set to pending-install.
    
    * `deployed` As soon as Kubernetes accepts the manifest from Helm, Helm updates the release record, marking it as deployed.
    
    * `pending-upgrade` When a Helm upgrade is begun, a new release is created for an installation (e.g., v2), and its status is set to pending-upgrade.
    
    * `superseded` When an upgrade is run, the last deployed release is updated, marked as superseded, and the newly upgraded release is changed from pending-upgrade to deployed.
    
    * `pending-rollback` If a rollback is created, a new release (e.g., v3) is created, and its status is set to pending-rollback until Kubernetes accepts the release manifest. Then it is marked deployed and the last release is marked superseded.
    
    * `uninstalling` When a helm uninstall is executed, the most recent release is read and then its status is changed to uninstalling.
    
    * `uninstalled` If history is preserved during deletion, then when the helm uninstall is complete, the last release’s status is changed to uninstalled.
    
    * `failed` Finally, if during any operation, Kubernetes rejects a manifest submitted by Helm, Helm will mark that release failed.


**Step 3:** Rollback release with `helm rollback `.

From the error, we know that the release failed because we supplied an invalid image pull policy. So of course we could correct this by simply running another helm upgrade. 
But imagine a case where the cause of error was not readily available. Rather than leave the application in a failed state while diagnosing the problem, it would be nice to 
simply revert back to the release that worked before.


```
helm rollback ingress-nginx 2 -n ingress-nginx
helm history ingress-nginx -n ingress-nginx
helm list
```

```
4               Tue Aug 10 11:16:24 2021        deployed        ingress-nginx-3.35.0    0.48.1          Rollback to 2  
---
ingress-nginx   ingress-nginx   4               2021-08-10 11:16:24.424371 -0400 EDT    deployed        ingress-nginx-3.35.0    0.48.1  
```

```
Rollback was a success! Happy Helming!
```

!!! success
    This command tells Helm to fetch the ingress-nginx version 2 release, and resubmit that manifest to Kubernetes. 
    A rollback does not restore to a previous snapshot of the cluster. Helm does not track enough information to do that. What it does is resubmit the previous configuration, 
    and Kubernetes attempts to reset the resources to match.


### 3.2.5  Upgrade Release with Helm `Diff` Plugin

While Helm provides a lot of functionality out of the box. Community contributes a lot of great functionality via helm plugins.

Plugins allow you to add extra functionality to Helm and integrate seamlessly with the CLI, making them a popular choice for users with unique workflow requirements. 
There are a number of third-party plugins available online for common use cases, such as secrets management. In addition, plugins are incredibly easy to build on your own for unique, one-off tasks.

Helm plugins are external tools that are accessible directly from the Helm CLI. They allow you to add custom subcommands to Helm without making any modifications to Helm’s Go source code.

Many third-party plugins are made open source and publicly available on GitHub. Many of these plugins use the “helm-plugin” tag/topic to make them easy to find. Refer to the documentation for [Helm plugins on GitHub](https://github.com/topics/helm-plugin)

Let's Upgrade our running Ingress Nginx chart with metrics service that will enable Prometheus to scrape metrics from our Nginx.

**Step 1:** Update `custom_values.yaml` with following information:

```
cat << EOF>> custom_values.yaml
  metrics:
    enabled: true
EOF
```

**Step 2:** So far we've used `helm template` to render and compare manifests. However most of the time you might not be able to do it during upgrades.
However you want to have a clear understanding what will be upgraded if you change something.

That's where [Helm Diff Plugin](https://github.com/databus23/helm-diff) can be handy. 
Helm Diff plugin giving your a preview of what a helm upgrade would change. It basically generates a diff between the latest deployed version of a release and a `helm upgrade --dry-run`


Install Helm Diff Plugin:

```
helm plugin install https://github.com/databus23/helm-diff
```

```
helm diff upgrade ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx
```

**Output:**

```
-           imagePullPolicy: Always
+           imagePullPolicy: IfNotPresent
+               protocol: TCP
+             - name: metrics
+               containerPort: 1025
+ # Source: ingress-nginx/templates/controller-service-metrics.yaml
+ apiVersion: v1
+ kind: Service
...
```

!!! result
    Amazing! This looks like `terraform plan` but for helm :) 

!!! important
    Another discovery from above printout is that since we forgot to add `--set imagePullPolicy` command, our value will be reverted with upgrade.
    This is really important to understand as you configuration maybe lost if you use `--set`



**Step 3:** Let's upgrade our `ingress-nginx` release:

```
helm  upgrade ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0 --namespace ingress-nginx
 helm list -n ingress-nginx
```

!!! success
    Our `ingress-nginx` is ready to run. Going forward we going to always deploy this chart.



!!! summary
    So far we've learned:
    
    * How to customize helm chart deployment using `--set` and `--values` using values.yaml

    * Using `values.yaml` preferable mehtod in order to achieve reproducible deployments
 
    * How to upgrade and rollback releases

    * Helm template and dry-run

    * Helm Diff Plugin


### 3.2.6 Install Minio with Helm

Let's deploy another Helm application on our Kubernetes cluster: [Minio](https://min.io/) - high-performance, S3 compatible object storage.

```
helm install myminio bitnami/minio
```

Follow your `Minio` Helm Chart `notes` Instruction to access Website:

```
To access the MinIO&reg; web UI:

- Get the MinIO&reg; URL:

   echo "MinIO&reg; web URL: http://127.0.0.1:9000/minio"
   kubectl port-forward --namespace default svc/myminio 8080:9000
```

In Gcloud console execute `kubectl port-forward`:

```
kubectl port-forward --namespace default svc/myminio 8080:9000
```

!!! result
    We can access Minio UI using private IP via tunnel and we have to use autogenerated ACCESS_KEY and SECRET_KEY.



Uninstall the Minio chart:

```
helm uninstall myminio bitnami/minio
```

While it looks easy to start with helm and deploy any charts, most of the time you want to have a full control of the deployment process.
You want to specify, tested version, safe values or configurations that make sense for you or your organization.


### 3.2.7 Customize Minio Installation Task


**Task N1:** Customize Minio Installation with following values:
    
    * Deploy `Minio` of specific version: `7.1.7`
    * Expose Minio using `Ingress` resource:
        * ingress.class: "nginx"
        * path: "/"
        * pathType: "ImplementationSpecific"
        * hostname: $STATIC_IP_ADDRESS.nip.io
    * Specify custom `ACCESS_KEY` for `Minio` frontend: `myaccesskey`    
    * Specify custom `SECRET_KEY` for `Minio` frontend: `mysecretkey`


**Step 1:** Use the following Minio Chart:

```
helm search hub minio
```

```
URL                                                     CHART VERSION   APP VERSION                                     DESCRIPTION                                       
https://artifacthub.io/packages/helm/bitnami/minio      7.1.7           2021.6.17                                       Bitnami Object Storage based on MinIO&reg
.....
```

!!! result
    We going to choose `bitnami` package again as it's top of the list and seems maintained the most.

**Step 2:** Since we already added `bitnami` repository locally, let's verify if we can list `minio` in it:

```
helm search repo minio
```

!!! result
    We can now go ahead and install `minio` 

**Step 4:** Pull `Minio` Helm Chart of specific version to Local filesystem to work with values file.

```
cd ~/$MY_REPO/notepad-infrastructure/helm
helm pull bitnami/minio --version 7.1.7

```
cd minio
```
cat <<EOF> custom_values.yaml
TODO: Finish the remaining steps
EOF
```


**TODO:** Complete `custom_values.yaml` and test deployment of `Minio`. Test Minio by creating `bucket` and uploading `object` to it. 



**Step 5:** Test and Install Minio Chart with Custom Configuration

```
helm install myminio bitnami/minio --values custom_values.yaml --version 7.1.6
```
Follow your `Minio` Helm Chart `notes` Instruction to access Website.

**Step 6** Commit `deploy_ycit020_a4/helm` and `notepad-infrastructure/helm` folder using the following Git commands:


```
cd ~/$MY_REPO
```

```
git add .
git commit -m "Helm values for Minio"
```

**Step 7** Push commit to the Cloud Source Repositories:

```
git push origin master
```


**Step 8**  Uninstall the Minio chart:

```
helm uninstall myminio bitnami/minio
```


## 4 Creating NotePad Helm Charts

So far with learned how to deploy existing charts from the Artifact Hub.

However sometimes you or your company needs to build software that require's to be distributed and shared externally,
as well as have lifecycle and release management. Helm Charts becomes are valuable option for that, specifically if you application is based
of containers!

Imagine that our solution is Go-based NotePad has first customer that wants to deploy it on their system and they requested delivery via
Helm Charts that available on GCP based Helm Repository. To achieve such task we going to create 2 Charts:

    * `gowebapp-mysql` chart
    * `gowebapp` chart

And store them in [Google Artifact Registry](https://cloud.google.com/artifact-registry/docs/helm/manage-charts) that provides Helm 3 OCI Repository.



### 4.1 Design `gowebapp-mysql` chart

#### 4.1.1  Create `gowebapp-mysql` chart

As scary as it sounds creating a new basic helm chart is 5 minute thing! We going to learn 2 quick methods to create helm charts, that will help you to save time and get started with helm quickly.

Helm includes the `helm create` command to make it easy for you to create a chart of your own, and it’s a great way to get started.

The create command creates a chart for you, with all the required chart structure and files. These files are documented to help you understand what is needed, and the templates it provides 
showcase multiple Kubernetes manifests working together to deploy an application. In addition, you can install and test this chart right out of the box.


**Step 1:** Create a new chart:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm
```

```
helm create gowebapp-mysql
```
```
cd gowebapp-mysql
tree
```

**Output:**
```
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

!!! result 
     This command creates a new Nginx chart, with a name of your choice, following best practices for a chart layout. Since Kubernetes clusters can have different methods to expose an application, this chart makes the way Nginx is exposed to network traffic configurable so it can be exposed in a wide variety of clusters. The chart has following structure:
    
    *  The `Chart.yaml` file contains metadata and some functionality controls for the chart
    *  The `charts` folder currently empty may container charts dependency of the top level chart. For example we could of make our `gowebapp-mysql` as dependency chart for `gowebapp`
    *  `templates` folder contains all `yaml` manifests in [Go templates](https://pkg.go.dev/text/template) and used to generate Kubernetes manifests
    *  `values.yaml` contains `default` values applied during the rendering.
    *  The `NOTES.txt` file is a special template. When a chart is installed, the NOTES.txt template is rendered and displayed rather than being installed into a cluster.
    * `_helpers.tpl` - helper templates for your other templates (e.g creating same labers across all charts). Files with `_` are not rendered to Kubernetes object definitions, but are available everywhere within other chart templates for use.


Here we going to show a quick way to create a Helm chart without adding any templating.

**Step 2:** Delete `templates` folder and create empty one:

```
rm -rf templates
mkdir templates
cd templates
```

**Step 3:** Copy existing `gowebapp-mysql` manifests to `templates` folder:

```
cp -r ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/gowebapp-mysql-pvc.yaml  .         # PVC
cp -r ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/secret-mysql.yaml   .              # Secret
cp -r ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/gowebapp-mysql-service.yaml .      # Service
cp -r ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/gowebapp-mysql-deployment.yaml .   # Deployment
```


**Step 4:** Lint Helm Chart:

When developing charts, especially when working with YAML templates, it can be easy to make a mistake or miss something. 
To help you catch errors, bugs, style issues, and other suspicious elements, the Helm client includes a linter. 
This linter can be used during chart development and as part of any testing processes.

To use the linter, use the `lint` command on a chart as a directory or a packaged archive:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
helm lint gowebapp-mysql
```

**Output:**
```
==> Linting gowebapp-mysql
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```


**Step 5:**  Install Helm Chart locally:

Create `dev` namespace:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/gowebapp-mysql
kubectl create ns dev
kubectl config set-context --current --namespace=dev
```

Render the manifest locally and compare to original manifests:

```
helm template gowebapp-mysql . > render.yaml
```

Install the chart:
```
helm install gowebapp-mysql .
```

**Output:**
```
NAME: gowebapp-mysql
LAST DEPLOYED: Tue Aug 10 15:25:44 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

```
helm list
kubectl get all
kubectl get pvc
```

!!! success
    We've deployed our own helm chart locally. While we haven't used templating of the chart at all, we have a helm chart that can be installed and upgraded using helm release features.


    
### 4.2  Design `gowebapp` chart.

#### 4.2.1  Create `gowebapp` chart

We going to use another method to deploy our second chart `gowebapp`. In this case we going to use `nginx` template provided by help team. 

**Task N2:** Create `gowebapp` Helm chart:

    * Configure `Deployment` template in `values.yaml`
    * Configure `Service` template in `values.yaml`
    * Disable `Service account` template in `values.yaml`
    * Configure `Ingress` template in `values.yaml`:
        * ingress.class: "nginx"
        * path: "/"
        * pathType: "ImplementationSpecific"
        * host: $STATIC_IP_ADDRESS.nip.io
    * Templatize the ConfigMap Resource
    * Ensure the chart is deployable
    * Ensure `gowebapp` in browser

    
**Step 1** Create a new chart:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
```

```
helm create gowebapp
```

```
cd gowebapp
tree
```

**Output:**
```
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

```
edit values.yaml
```

#### 4.2.2 Template the Deployment

**Step 1** Update the replicaCount value to 2 in the `values.yaml` file:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/values.yaml
```

**Step 2** Update the repository and tag section to point to your `gowebapp` docker image in the `values.yaml` file:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/values.yaml
```

**Step 3** Update the resources section in the `values.yaml` file to include the resource `requests` and `limits` the gowebapp application needs:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/values.yaml
```

See the reference `gowebapp-deployment.yaml`:

```
cat ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/gowebapp-deployment.yaml
```

**Step 4** Update the livness and readiness sections to match what you have in the `gowebapp` `deployment.yaml`

See the reference `gowebapp-deployment.yaml`:

```
cat ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/gowebapp-deployment.yaml
```

Update livness and readiness sections for template `deployment.yaml`:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/templates/deployment.yaml
```

**Step 5** Notice that the `deployment.yaml` file does not have an environment variable section for `secrets`, so let's add one. For this chart we will assume that this section is optional based on whether or not a secrets section exist in the Values.yaml file.

**Step 5-a** Include the following in the deployment template:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/templates/deployment.yaml
```

And add following code snippet in the appropriate location:

```
{{- if .Values.secrets }}
- env:
  - name: {{.Values.secrets.name}}
    valueFrom:
      secretKeyRef:
        name: {{.Values.secrets.secretReference.name}}
        key: {{.Values.secrets.secretReference.key}}
{{- end}}
```

**Step 5-b** Include a section in the `values.yaml` file:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/values.yaml
```

Add following snippet:

```
secrets:
  enabled: true
  name: DB_PASSWORD
  secretReference:
    name: mysql
    key: password
```

**Step 6** For this lab, we will include the `volumes` and `volumeMounts` sections without templating, so just copy the required sections to the appropriate location in the `deployment.yaml` template.

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/templates/deployment.yaml
```

See the reference `gowebapp-deployment.yaml`:

```
cat ~/$MY_REPO/deploy_ycit020_a4/k8s-manifests/gowebapp-deployment.yaml
```

**Step 7**  Render the Chart locally and compare Deployment to original `gowebapp-deployment.yaml` manifests:

```
helm template gowebapp-mysql . > render.yaml
```


####  4.2.3 Template the Service

**Step 1:** The `service.yaml` template doesn't have an `annotation` section, so modify the template to add an `annotation` section, that looks following:
```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/templates/service.yaml
```

```
      {{- with .Values.annotations }}
      annotations:
        {{- toYaml . | nindent 4 }}
      {{- end }}
```

**TODO:** Next, modify the `values.yaml` file to allow chart users to add annotations for the service. Make sure to use the right section in the `values.yaml` file, based on how you modified your `/templates/service.yaml` file.

Reference documentation: 
    
    * https://helm.sh/docs/chart_template_guide/control_structures/#modifying-scope-using-with

**Step 2:** Under the `service:` section in the `values.yaml` file, update the service `port:` to 9000.

**Step 3** In the `values.yaml` file, update the service type to `NodePort`

**Step 4**  Render the Chart locally and compare Service to original `gowebapp-service.yaml` manifests:

```
helm template gowebapp-mysql . > render.yaml
```


####  4.2.4 Disable the Service account 

**Step 1** Update the `values.yaml` file to `disable` the service account creation for the `gowebapp` deployment.


####  4.2.5 Template the Ingress Resource


**Step 1** `enable` the ingress the `values.yaml` file and configur according requirements: :

    * Expose `gowebapp` using `Ingress` resource:
        * ingress.class: "nginx"
        * path: "/"
        * pathType: "ImplementationSpecific"
        * host: $STATIC_IP_ADDRESS.nip.io

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
edit gowebapp/values.yaml
```

**Step 2**  Render the Chart locally and verify if any issues:

```
helm template gowebapp-mysql . > render.yaml
```


####  4.2.5 Templatize the ConfigMap Resource

**Step 1**  Create and templatize `configmap` resource for our `gowebapp` that provides connection to Mysql:


```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/gowebapp/templates/
cat <<EOF>> configmap.yaml
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: {{ .Values.configMap.name }}
data:
  webapp-config-json: |-
{{ .Files.Get "config.json" | indent 4 }}
EOF
```

Store the `config.json` inside the chart repository:

```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/gowebapp
cat <<EOF>> config.json
{
	"Database": {
		"Type": "MySQL",
		"Bolt": {
 			"Path": "gowebapp.db"
  		},
		"MongoDB": {
			"URL": "127.0.0.1",
			"Database": "gowebapp"
		},
		"MySQL": {
			"Username": "root",
			"Password": "rootpasswd",
			"Name": "gowebapp",
			"Hostname": "gowebapp-mysql",
			"Port": 3306,
			"Parameter": "?parseTime=true"
		}
	},
	"Email": {
		"Username": "",
		"Password": "",
		"Hostname": "",
		"Port": 25,
		"From": ""
	},
	"Recaptcha": {
		"Enabled": false,
		"Secret": "",
		"SiteKey": ""
	},
	"Server": {
		"Hostname": "",
		"UseHTTP": true,
		"UseHTTPS": false,
		"HTTPPort": 80,
		"HTTPSPort": 443,
		"CertFile": "tls/server.crt",
		"KeyFile": "tls/server.key"
	},
	"Session": {
		"SecretKey": "@r4B?EThaSEh_drudR7P_hub=s#s2Pah",
		"Name": "gosess",
		"Options": {
			"Path": "/",
			"Domain": "",
			"MaxAge": 28800,
			"Secure": false,
			"HttpOnly": true
		}
	},
	"Template": {
		"Root": "base",
		"Children": [
			"partial/menu",
			"partial/footer"
		]
	},
	"View": {
		"BaseURI": "/",
		"Extension": "tmpl",
		"Folder": "template",
		"Name": "blank",
		"Caching": true
	}
}
EOF
```

And finally add following snippet inside `values.yaml`:


```
edit gowebapp/values.yaml
```

```
configMap:
  name: gowebapp
```


**Step 2**  Render the Chart locally and verify if any issues:

```
helm template gowebapp-mysql . > render.yaml
```



####  4.2.6 Deploy `gowebapp`

Before deployment make sure test chart with `dry-run`, `template` and `lint` the chart

Lint: 
```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/
helm lint gowebapp
```

Install:
```
cd ~/$MY_REPO/deploy_ycit020_a4/helm/gowebapp
helm install gowebapp .
```

**Output:**
```
NAME: gowebapp
LAST DEPLOYED: Tue Aug 10 15:25:44 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

```
helm ls
kubectl get all
kubectl get pvc
kubectl get ing
```

Access `gowebapp` with Ingress


!!! success
    You've now became a chart developer! And learned, how to create basic helm charts with 2 methods.
    You can farther customize the chart as needed and add more templates as you go.
    The next step would be to look contribute back to Helm community, use charts in Artificatory, find issue or add missing feature,
    help grow help project!
    
####  4.2.7 Submit assignement

**Step 1** Commit chart configuration in `deploy_ycit020_a4/helm` folder using the following Git commands:


```
cd ~/$MY_REPO
```

```
git add .
git commit -m "Gowebapp Helm Chart"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```


**Step 3** Uninstall chart `gowebapp` and `gowebapp-mysql` chart

```
helm install gowebapp
helm uninstall gowebapp-mysql
```

**Step 4** Resize GKE Cluster to `0` nodes, to avoid charges:


```
cd ~/$MY_REPO/notepad-infrastructure
```

```
edit terraform.tfvars
```

And set `gke_pool_node_count`   = "0"

**Step 3:** Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

**Step 4:** Shutdown all Nodes in GKE Cluster Node Pool:

```
terraform apply -var-file terraform.tfvars
```


!!! result
    GKE Clusters has been scale down to 0 nodes.


# Part 2 WIP

## 5 Chart Repositories

## 6 HelmFile

## 7 Install Istio on GKE

Currently Istio community developing following options of Istio deployment on Kubernetes Clusters:

  * **Install with Istioctl** installation via `istioctl` command line tool used to showcase Istio functionality. Using the CLI, we generate a YAML file with all Istio resources and then deploy it to the Kubernetes cluster
  * **Istio Operator Install** Takes installation of Istio to the next level as it's managing not only Istio installation but overall lifecicle of the Istio deployment including Upgrades and configurations.
  * **Install with Helm (alpha)** Allows to farther simplify Istio deployment and it's customization.  


### 7.0 Prerequisite

Scaleup cluster back to 3 nodes and Update VPC firewall rules to enable `auto-injection` and the `istioctl version` and `istioctl ps` commands.

References:

1. [Opening ports on a private cluster](https://cloud.google.com/service-mesh/v1.7/docs/private-cluster-open-port)
2. Istio deployment on [GKE Private Clusters](https://istio.io/latest/docs/setup/platform-setup/gke/)

**Step 1:** Locate Terraform Configuration directory.

```
cd ~/$MY_REPO/notepad-infrastructure
```

**Step 2:** Configure GKE Cluster to Scale back to `1` node per zone in a region.

```
edit terraform.tfvars
```

And set `gke_pool_node_count`   = "1"


**Step 3:**  Create new firewall rule for source range (master-ipv4-cidr) of the cluster, that will open required ports for Istio installation:

```
cat <<EOF> istio_firewall.tf
resource "google_compute_firewall" "istio_specific" {
  name =   format("allow-istio-in-privategke-%s-%s-%s", var.org, var.product, var.environment)
  network = google_compute_network.vpc_network.self_link
  source_ranges = ["172.16.0.0/28"]
  allow {
    protocol = "tcp"
    ports = ["10250", "443", "15017", "15014", "8080"]
  }
}
EOF
```

**Step 4:** Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

**Step 5:** Scale up GKE Cluster Node Pool and update firewall rules for required range:

```
terraform apply -var-file terraform.tfvars
```

!!! result
    GKE Clusters has been scalled to 3 nodes and has firewall rules opened

**Step 6:** Verify firewall rule:

```
gcloud compute firewall-rules list --filter="name~allow-istio-in-privategke*"
```

**Output:**
```
NAME                                            NETWORK                   DIRECTION  PRIORITY  ALLOW                                           DENY  DISABLED
allow-istio-in-privategke-$student-notepad-dev  vpc-$student-notepad-dev  INGRESS    1000      tcp:10250,tcp:443,tcp:15017,tcp:15014,tcp:8080        False
```



### 7.1 Deploy Istio using Custom Resources


This installation guide uses the istioctl command line tool to provide rich customization of the Istio control plane and of the sidecars for the Istio data plane.


Download and extract the latest release:

```
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.11.0
export PATH=$PWD/bin:$PATH
```

!!! success
    The above command will fetch Istio packages and untar them in the same folder.


```
tree -L 1
```


```
.
├── LICENSE
├── README.md
├── bin
├── manifest.yaml
├── manifests
├── samples
└── tools
```

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
istioctl install --set profile=demo -y
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

```kubec
kubectl get crds | grep istio
```

```
authorizationpolicies.security.istio.io          2021-08-16T19:38:16Z
destinationrules.networking.istio.io             2021-08-16T19:38:16Z
envoyfilters.networking.istio.io                 2021-08-16T19:38:16Z
gateways.networking.istio.io                     2021-08-16T19:38:17Z
istiooperators.install.istio.io                  2021-08-16T19:38:17Z
peerauthentications.security.istio.io            2021-08-16T19:38:17Z
requestauthentications.security.istio.io         2021-08-16T19:38:17Z
serviceentries.networking.istio.io               2021-08-16T19:38:17Z
sidecars.networking.istio.io                     2021-08-16T19:38:17Z
telemetries.telemetry.istio.io                   2021-08-16T19:38:18Z
virtualservices.networking.istio.io              2021-08-16T19:38:18Z
workloadentries.networking.istio.io              2021-08-16T19:38:18Z
workloadgroups.networking.istio.io               2021-08-16T19:38:19Z
```


!!! info 
    Above CRDs will be avaialable as a new Kubernetes Resources and stored in Kubernetes ETCD database. The Kubernetes API will represent these new resources as endpoints that 
    can be used as other native Kubernetes object (such as Pod, Services) levereging  kubectl, RBAC and other features and  admission controllers of Kubernetes.


**Step 4** Count total number of Installed CRDs:

```
kubectl get crds | grep istio | wc -l
```

!!! note
    CRDs count will vary based on Istio version and profile deployed.


**Step 5** Verify that Istio control plane has been installed successfully.

```
kubectl get pods -n istio-system
```

**Output:**
```
istio-egressgateway-9dc6cbc49-5wkqt     1/1     Running   0          2m54s
istio-ingressgateway-7975cdb749-xjc5g   1/1     Running   0          2m54s
istiod-77b4d7b55d-j6kb5                 1/1     Running   0          3m9s
```


!!! info
    Istio control-plane include following components:

    * `istiod` - contains components such as `Citadel` and `Pilot`
    * `istio-ingressgateway` Istio Ingress Gateway
    * `istio-ingressgateway` Istio Egress Gateway


**Step 6** Verify installation with `istioctl` CLI:


```sh
istioctl version
```


```
istioctl verify-install
```

**Output:**
```
Checked 13 custom resource definitions
Checked 3 Istio Deployments
✔ Istio is installed and verified successfully
```


**Step 5**  Deploy  `Bookinfo` sample application:

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

```
kubectl get pods
```
**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-cjj5b       1/1     Running   0          12s
productpage-v1-6b746f74dc-sxdxz   1/1     Running   0          10s
ratings-v1-b6994bb9-jd985         1/1     Running   0          11s
reviews-v1-545db77b95-5kjr4       1/1     Running   0          11s
reviews-v2-7bf8c9648f-5v2s9       1/1     Running   0          11s
reviews-v3-84779c7bbc-5khlt       1/1     Running   0          11s
```

!!! note
    This is a typical Kubernetes deployment, and has nothing specific to Istio. Each Pod has 1 container.


**Step 6** Remove `Bookinfo` sample application:

```
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```


**Step 7** Enable automatic Envoy sidecar injection.

Currently there are 2 method to inject `Envoy` sidecar inside Istio Mesh:
  
  * Manual sidecar injection - modifies the controller configuration, e.g. deployment. It does this by modifying the pod template spec such that all pods for that deployment are created with the injected sidecar. Adding/Updating/Removing the sidecar requires modifying the entire deployment.
  * Automatic sidecar injection via a Mutating Admission Webhook - The Deployment resource is unmodified. Sidecars can be updated selectively by manually deleting a pods or systematically with a deployment rolling update.


Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:

```
kubectl label namespace default istio-injection=enabled
```

**Step 8** Deploy  `Bookinfo` sample application on Istio with Auto Sidecar Injection

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

**Step 8** Verify deployed application:

The application will start. As each pod becomes ready, the Istio sidecar will be deployed along with it.

```
kubectl get pods
```

**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-zqjbg       2/2     Running   0          13m
productpage-v1-6b746f74dc-dj959   2/2     Running   0          12m
ratings-v1-b6994bb9-29n4w         2/2     Running   0          12m
reviews-v1-545db77b95-p9pgf       2/2     Running   0          12m
reviews-v2-7bf8c9648f-4ltkg       2/2     Running   0          12m
reviews-v3-84779c7bbc-vhsmw       2/2     Running   0          12m
```


!!! note
    Each Pod  has now 2 containers. One application container and another is `istio-proxy` sidecar container.


**Step 9** Access UI `productpage` via console:

```
kubectl get services
```

In Gcloud console execute `kubectl port-forward`:

```
kubectl port-forward --namespace default svc/productpage 8080:9080
```

Click `Web-Preview` button in Gcloud, and select preview on port `8080`
Click `Normal user` URL.

!!! success
    We can access `Bookinfo` application configured with Istio and Envoy sidecar using private IP via tunnel


**Step 10** Remove `Bookinfo` sample application:

```
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```


**Step 11** Uninstall Istio


```
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```


### 7.2 Deploy Istio using Istio Operator


**Step 1:** Deploy the Istio operator:

```
istioctl operator init
```

!!! note
    This command runs the operator by creating the following resources in the `istio-operator` namespace:

    * The operator custom resource definition
    * The operator controller deployment
    * A service to access operator metrics
    * Necessary Istio operator RBAC rules

**Step 2:** To install the Istio demo configuration profile using the operator, run the following command:

```
kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
EOF
```

!!! note
    The Istio operator controller begins the process of installing Istio within 90 seconds of the creation of the IstioOperator resource. The Istio installation completes within 120 seconds.


**Step 3** Verify Operator resource status on Kubernetes Cluster:

```
kubectl get istiooperators  -n istio-system 
```

**Output:**
```
NAMESPACE      NAME                        REVISION   STATUS    AGE
istio-system   example-istiocontrolplane              HEALTHY   25m
```


```
kubectl describe istiooperators example-istiocontrolplane -n istio-system
```
```
Spec:
  Profile:  demo
Status:
  Component Status:
    Base:
      Status:  HEALTHY
    Egress Gateways:
      Status:  HEALTHY
    Ingress Gateways:
      Status:  HEALTHY
    Pilot:
      Status:  HEALTHY
  Status:      HEALTHY
Events:        <none>
```


```
kubectl get pods -n istio-system
```


**Output:**
```
istio-egressgateway-9dc6cbc49-5wkqt     1/1     Running   0          2m54s
istio-ingressgateway-7975cdb749-xjc5g   1/1     Running   0          2m54s
istiod-77b4d7b55d-j6kb5                 1/1     Running   0          3m9s
```


!!! info
    Istio control-plane include following components:

    * `istiod` - contains components such as `Citadel` and `Pilot`
    * `istio-ingressgateway` Istio Ingress Gateway
    * `istio-ingressgateway` Istio Egress Gateway


Verify installation with `istioctl` CLI:

```
istioctl verify-install
```

**Step 5** Enable automatic Envoy sidecar injection.

Currently there are 2 method to inject `Envoy` sidecar inside Istio Mesh:
  
  * Manual sidecar injection - modifies the controller configuration, e.g. deployment. It does this by modifying the pod template spec such that all pods for that deployment are created with the injected sidecar. Adding/Updating/Removing the sidecar requires modifying the entire deployment.
  * Automatic sidecar injection via a Mutating Admission Webhook - The Deployment resource is unmodified. Sidecars can be updated selectively by manually deleting a pods or systematically with a deployment rolling update.


Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:

```
kubectl label namespace default istio-injection=enabled
```

**Step 8** Deploy  `Bookinfo` sample application on Istio with Auto Sidecar Injection

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

**Step 8** Verify deployed application:

The application will start. As each pod becomes ready, the Istio sidecar will be deployed along with it.

```
kubectl get pods
```

**Output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-zqjbg       2/2     Running   0          13m
productpage-v1-6b746f74dc-dj959   2/2     Running   0          12m
ratings-v1-b6994bb9-29n4w         2/2     Running   0          12m
reviews-v1-545db77b95-p9pgf       2/2     Running   0          12m
reviews-v2-7bf8c9648f-4ltkg       2/2     Running   0          12m
reviews-v3-84779c7bbc-vhsmw       2/2     Running   0          12m
```


!!! summary
    We've leaned to deploy Istio Control plane using `istioctl` cli and using Istio Operator. We also deployed regular k8s application on the namespace marked with auto injection!
     


**Step 9** Cleanup  Uninstall Istio Operator

```
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```


**Step 10**  Uninstall Istio Operator

```
kubectl delete istiooperators.install.istio.io -n istio-system example-istiocontrolplane
istioctl operator remove
kubectl delete ns istio-system --grace-period=0 --force
```




## 8 Commit Readme doc to repository and share it with Instructor/Teacher

**Step 1** Commit `deploy_ycit020_a4/helm` and `notepad-infrastructure/helm` folder using the following Git commands:


```
cd ~/$MY_REPO
```

```
git add .
git commit -m "HelmFile Configuration to deploy NotePad and Suppporting Applications"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```

## 9 Cleanup

We going to cleanup GCP Service foundation layer with GKE Cluster to avoid excessive cost.

```
cd ~/$MY_REPO/notepad-infrastructure
terraform destroy -var-file terraform.tfvars
```
