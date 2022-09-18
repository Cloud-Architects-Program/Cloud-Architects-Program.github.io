# Lab 6 GitOps



**Objective:**

  * Install `Flux`
  * Bootstrap `Flux` with a new `flux-infra` repository
  * Add a `GitRepository` source type to track the `microservices-demo`Public application repository which also contains the deployment code. Flux should regularly sync the code from it.
  * Use the `Kustomization` Controller to set up automated deployment to the Kubernetes environment. Any changes in the source should be automatically reconciled with the cluster.


## 0 Create Regional GKE Cluster on GCP

**Step 1** Enable the Google Kubernetes Engine API.

```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:

```
gcloud container clusters create k8s-gitops-lab \
--region us-central1 \
--enable-ip-alias \
--enable-network-policy \
--num-nodes 1 \
--machine-type "e2-standard-2" \
--release-channel stable
```

```
gcloud container clusters get-credentials k8s-gitops-lab --region us-central1
```


**Step 3: (Optional)** Setup kubectx

```
sudo apt install kubectx
```

!!! note
    we've installed `kubectx` + `kubens`: Power tools for `kubectl`:
    - `kubectx` helps you switch between clusters back and forth
    - `kubens` helps you switch between Kubernetes namespaces smoothly

## 1 Install and Configure Flux with GitHub Repository

To Install Flux we going to perform following tasks:
  
  * Create a Personal Access Token on GitHub.
  * Set up environment variables.
  * Install flux CLI.
  * Run pre-flight checks.
  * Bootstrap the Flux v2 infrastructure.
  * Validate the Flux installation.

### 1.1 Install the Flux CLI 

With `Bash` for macOS and Linux:

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

Reference: [Get Started with Flux](https://fluxcd.io/docs/get-started/)


### 1.2 Configure a GitHub personal access token with repo permissions. 

Follow the GitHub documentation on [creating a personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line).



!!! result
    You've created `GITHUB_TOKEN` that will be used to boostrap the Flux


### 1.3 Export your credentials 

Export your GitHub personal access token, generated in step 1.2 and `github` username:


```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

### 1.4 Install Flux into GKE cluster 

Run the bootstrap command:

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-infra \
  --branch=main \
  --path=./clusters/dev \
  --personal \
  --log-level=debug \
  --components=source-controller,kustomize-controller,helm-controller
```

**Output:**
```
✔ repository "https://github.com/archyufa/flux-infra" created
✔ cloned repository
✔ generated component manifests
✔ committed sync manifests to "main" ("6cd871ba49834e8106884f3c8daad2f75c6b341f")
✔ installed components
✔ reconciled components
✔ configured deploy key "flux-system-main-flux-system-./clusters/dev" for "https://github.com/archyufa/flux-infra"
✔ reconciled source secret
✔ generated sync manifests
✔ committed sync manifests to "main" ("7d0122f2e4e57ada4f88d7a968528879b07b56e8")
✔ reconciled sync configuration
✔ Kustomization reconciled successfully
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

Verify Install:

```
flux check
```

```
kubectl get crd | grep flux
```

**Output:**
```
buckets.source.toolkit.fluxcd.io                 2021-09-08T18:07:34Z
gitrepositories.source.toolkit.fluxcd.io         2021-09-08T18:07:34Z
helmcharts.source.toolkit.fluxcd.io              2021-09-08T18:07:34Z
helmreleases.helm.toolkit.fluxcd.io              2021-09-08T18:07:34Z
helmrepositories.source.toolkit.fluxcd.io        2021-09-08T18:07:35Z
kustomizations.kustomize.toolkit.fluxcd.io       2021-09-08T18:07:35Z
```

```
kubectl get gitrepositories -n flux-system
```

```
flux get all
```

!!! summary
    We've `flux bootstrap` command we've achieved:
    
      * flux agent components which includes (namespace, CRDs, Sources, Controllers, RBAC) on our cluster
        * 3 Controllers:
          * source
          * kustomize
          * helm
        * 6 CRDs: 
          * buckets.source
          * gitrepositories.source
          * helmcharts.source
          * helmrepositories.source
          * helmreleases.helm
          * kustomizations.kustomize
      * Created `flux-infra` Private Github repo with Deploy key
      * `flux-infra` repo contains flux-infra/clusters/dev/flux-system/ folder

### 1.5 Deploy `microservices` applications with GitOps using Kustomize Controller

Create Dev Team Repo that will store Applications Code and Kubernetes Yaml manifests.

**Step 1** Fork microservice application

Browse to:

```
https://github.com/GoogleCloudPlatform/microservices-demo
```

```
Click: Fork button
Select: Fork to a different account
```

**Step 2** Clone Microservices application from you repo using SSH

```
cd ~
git clone git@github.com:<your_repo>/microservices-demo.git
cd microservices-demo
```

```
rm release/istio-manifests.yaml
git add .
git commit -m "removed istio manifests"
git push
```


**Step 3** Create Namespace `onlineboutique`

```
kubectl create ns onlineboutique
```


**Step 4** Create a source from a public Git repository master branch:

```
flux create source git -h
```

```
flux create source git microservices-demo \
  --url=https://github.com/<your_repo>/microservices-demo-1 \
  --branch=master \
  --interval=30s
```

**Output:**
```
✚ generating GitRepository source
► applying GitRepository source
✔ GitRepository source created
◎ waiting for GitRepository source reconciliation
✔ GitRepository source reconciliation completed
✔ fetched revision: master/2e55bc94d35d17aa6b690d63353c38718bc31253
```

Verify:

```
flux get sources git microservices-demo
```


**Step 5**  Create a `Kustomization` resource from a source at a given path

```
flux create kustomization -h
```

```
flux create kustomization microservices-demo-dev \
  --source=GitRepository/microservices-demo \
  --path="./release/" \
  --prune=true \
  --interval=1m \
  --validation=client \
  --target-namespace=onlineboutique
```

```
kubens onlineboutique
kubectl get pods
```

!!! result
    Onlineboutique pods has been deployed with FluxCD via GitOps


```
kubectl get svc
```

Access Onlineboutique `Frontend` via `LoadBalancer` IP


!!! summary
    We've deployed microservices onlineboutique application using GitOps

 
### 1.6 Deploy OPA Gatekeeper using Helm Controller

Now let's install OPA Gatekeeper using Helm so that we can apply different Kubernetes Policy and set the guardrails on what's allowed to do on our cluster.

**Step 1** Configure `Helm` repository


```
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
```

**Step 2** Install Helm Chart to `monitoring` namespace


```
kubectl create ns gatekeeper
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper
kubens gatekeeper
kubectl get deploy
```

**Step 2** Uninstall Gatekeeper

```
helm delete gatekeeper
kubectl delete crd -l gatekeeper.sh/system=yes
```

### 1.7 Deploy OPA Gatekeeper with GitOps using Helm Controller

Now let's install OPA Gatekeeper via GitOps so that we can apply different Kubernetes Policy and set the guardrails on what's allowed to do on our cluster.


**Step 1** Create a source for a public Helm repository:

```
flux create source helm -h
```

```
flux create source helm gatekeeper \
  --url=https://open-policy-agent.github.io/gatekeeper/charts \
  --interval=1m
```


Verify:

```
flux get sources helm gatekeeper
```


**Step 5**  Create a `HelmRelease` with a chart from a HelmRepository source


```
flux create hr -h
```


```
flux create hr gatekeeper \
  --interval=1m \
  --source=HelmRepository/gatekeeper \
  --chart=gatekeeper \
  --target-namespace=gatekeeper \
  --chart-version="v3.7.0-beta.1"
```

Verify:

```
flux get hr
```

**Output:**
```
gatekeeper      True    Release reconciliation succeeded        3.7.0-beta.1    False
```

```
kubens gatekeeper
kubectl get deploy
```


!!! result
    We've deployed Gatekeeper to our GKE cluster. It's time to add some policies!

**Step 5**  Store Flux configurations:

```
git clone git@github.com:<your-user>/flux-infra.git
cd flux-infra/clusters/dev/
flux export source git microservices-demo >>  source-microservices-demo.yaml
flux export kustomization microservices-demo-dev >> kustomization-microservices-demo.yaml
flux export source helm gatekeeper >> source-helm-gatekeeper.yaml
flux export hr gatekeeper >> helmrelease-gatekeeper.yaml
cat source-microservices-demo.yaml kustomization-microservices-demo.yaml source-helm-gatekeeper.yaml helmrelease-gatekeeper.yaml
```

```
git add .
git commit -m "dev cluster flux config"
git push
```

### 1.8 Deploy and Test allowedrepos Policy

**Step 1** Let's create a `policy` folder where we going to store our cluster wide policies:

```
cd ~
mkdir -p policy/templates
mkdir -p policy/constraints
```


**Step 2** Download OSS Gatekeeper libraries:
```
cd ~
git clone https://github.com/open-policy-agent/gatekeeper-library
```

**Step 3** Review `allowedrepos` constraint template:

```
cd gatekeeper-library/library/general/allowedrepos
cat template.yaml
cp -r template.yaml ~/policy/templates
```

```
cd  ~/policy/templates
kubens gatekeeper
kubectl apply -f template.yaml
```

Review a `ConstraintTemplate` CRD and observe created resource:

```
kubectl get  ConstraintTemplate
```

!!! note
    Even if k8sallowedrepos `ConstraintTemplate` has been deployed in `gatekeeper` namespace in our case.
    The Policy will be applied Cluster Wide.

**Step 3** Let's create a Gatekeeper constraint that will block all Pods, which images are non  `gcr.io` registries. We first going to test our constraint in a current system and so we going to run it in `dryrun` enforcementAction mode. We will 

```
cd  ~/policy/constraints
cat << EOF>> allowed-repos.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  enforcementAction: dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["onlineboutique"]
  parameters:
    repos:
      - "gcr.io"
      - "gke.gcr.io"
      - "k8s.gcr.io"
EOF
```

!!! note
    We've added `excludedNamespaces` clause, as our microservices app has some redis image that is downloaded from dockerhub.


```
kubectl apply -f allowed-repos.yaml
```


```
kubectl describe K8sAllowedRepos
```

**Output:**
```
Total Violations:  12
  Violations:
    Enforcement Action:  dryrun
    Kind:                Pod
    Message:             container <manager> has an invalid image repo <ghcr.io/fluxcd/helm-controller:v0.11.2>, allowed repos are ["gcr.io", "gke.gcr.io"]
    Name:                helm-controller-5cbd57f468-cr7ht
    Namespace:           flux-system
```

!!! note
    We can observe ~12 Violations in our cluster. However since our Enforcement Action is `dryrun`, non the policy will take effect. 


```
kubectl describe K8sAllowedRepos | grep Namespace
```

**Output:**
```
    Namespace:           flux-system
    Namespace:           flux-system
    Namespace:           flux-system
    Namespace:           gatekeeper
    Namespace:           gatekeeper
    Namespace:           gatekeeper
    Namespace:           gatekeeper
    Namespace:           kube-system
    Namespace:           kube-system
    Namespace:           kube-system
    Namespace:           kube-system
    Namespace:           kube-system
```

!!! note
    We can observe that all violations are for `kube-system`, `flux-system` and `gatekeeper`.
    This is due to they using images from following repos `k8s.gcr.io`, `ghcr.io/fluxcd`, `openpolicyagent/gatekeeper` respectively. To workaround this issue we can add above repo's in trusted listed of repos.
    Another alternative is to `copy` required images to your personal `gcr` repo using tools like [crane](https://github.com/google/go-containerregistry/blob/main/cmd/gcrane/README.md).

**Step 4** Update K8sAllowedRepos constraint to support trusted registries:

```
rm allowed-repos.yaml
cat << EOF>> allowed-repos.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  enforcementAction: dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["onlineboutique"]
  parameters:
    repos:
      - "gcr.io"
      - "gke.gcr.io"
      - "k8s.gcr.io"
      - "ghcr.io/fluxcd"
      - "openpolicyagent/gatekeeper"
EOF
```

```
kubectl apply -f allowed-repos.yaml
```

```
kubectl describe K8sAllowedRepos
```

!!! note
    All violations has been resolved, however it may take some time until the messages will be cleared in the Events.

**Step 5** Update K8sAllowedRepos constraint and set `enforcementAction` to `deny` in order to block all Pods with registry violations:

```
rm allowed-repos.yaml
cat << EOF>> allowed-repos.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["onlineboutique"]
  parameters:
    repos:
      - "gcr.io"
      - "gke.gcr.io"
      - "k8s.gcr.io"
      - "ghcr.io/fluxcd"
      - "openpolicyagent/gatekeeper"
EOF
```

```
kubectl apply -f allowed-repos.yaml
```

```
kubectl describe K8sAllowedRepos
```

!!! result
    Total Violations is 0, as expected.


**Step 6** Deploy `nginx` Pod from dockerhub in `default` namespace and thus violate our cluster Policy

```
kubectl run nginx --image=nginx -n default
```

**Output:**

```
Error from server ([allowed-repos] container <nginx> has an invalid image repo <nginx>, allowed repos are ["gcr.io", "gke.gcr.io", "k8s.gcr.io", "ghcr.io/fluxcd", "openpolicyagent/gatekeeper"]): admissionwebhook "validation.gatekeeper.sh" denied the request: [allowed-repos] container <nginx> has an invalid image repo <nginx>, allowed repos are ["gcr.io", "gke.gcr.io", "k8s.gcr.io", "ghcr.io/fluxcd", "openpolicyagent/gatekeeper"]
```

!!! error
    Gatekeeper Admission Controller blocked nginx Pod deployment due to it's violating the `allowedrepos` constraint.


!!! summary
    We've secured our cluster from deployments of dangerous and unauthorized images

### 1.8  (Extra) Deploy `allowedrepos` with Gitops

Create Ops/Platform Repo that will store Gatekeeper Policies and Cluster Wide configurations.

### 1.9 (Extra) Deploy notepad app Helm chart with GitOps

## 2 Cleanup 

You can uninstall Flux with:

```
flux delete kustomization microservices-demo-dev
flux delete source git microservices-demo
flux uninstall --namespace=flux-system
```

The above command performs the following operations:

  * deletes Flux components (deployments and services)
  * deletes Flux network policies
  * deletes Flux RBAC (service accounts, cluster roles and cluster role bindings)
  * removes the Kubernetes finalizers from Flux custom resources
  * deletes Flux custom resource definitions and custom resources
  * deletes the namespace where Flux was installed


Delete GKE cluster:

```
gcloud container clusters delete k8s-gitops-lab --region us-central1
```
