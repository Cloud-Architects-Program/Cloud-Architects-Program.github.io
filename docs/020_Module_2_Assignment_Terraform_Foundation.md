Lab 3 Creating Production GKE Cluster via IaC using Terraform

**Objective:**

  * Learn Terraform Commands
  * Learn GCP Terraform Provider
  * Learn Terraform variables
  * Store TF State in GCS buckets
  * Learn how to create GCP resources with Terraform 


## Working with Git and IaC

Git-based workflows, merge requests and peer reviews create a level of documented transparency that is great for security teams
and audits. They ensure every change is documented as well as the metadata surrounding the change, answering the why, who,
when and what questions.

In this set of exercises you will continue using `notepad` Google Source Repository that already contains you application code,
`dockerfiles` and kubernetes manifests.

We will use so called [Monorepo](https://en.wikipedia.org/wiki/Monorepo) approach, and store our terraform IAC configuration in the same repo as our application code and Kubernetes manifests.


### Add your IaC code to your repository


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

**Step 2** Create `foundation-infrastructure` and `notepad-infrastructure` folders that will contains respective terraform configurations

```
cd ~/$MY_REPO
mkdir foundation-infrastructure
mkdir notepad-infrastructure
```



**Step 3** Create a `README.md` file describing `foundation-infrastructure` folder purpose:  

```
cat <<EOF> foundation-infrastructure/README.md

# Creation of GCP Foundation Layer

This step, is focused on creating GCP Foundation: folders (if any) and service projects creation with their respictive GCS bucket to store terraform state and IAM Config.

EOF
```


**Step 4** Create a `README.md` file describing `notepad-infrastructure` folder purpose:  

```
cat <<EOF> notepad-infrastructure/README.md
# Creation of GCP Services Layer

This step, is focused on creating gcp services such as GKE, VPC and etc. inside of existing project

EOF
```


**Step 5** Create a .gitignore file in your working directory:


```
cat<<EOF>> .gitignore
.terraform.*
.terraform
terraform.tfstate*
EOF
```

!!! note
    Ignore files are used to tell `git` not to track files. You should also always include any files with credentials in a `gitignore` file.


List created folder structure along with `gitignore`:

```
ls -ltra
```

**Step 6** Commit `deploy` folder using the following Git commands:

```
git status 
git add .
git commit -m "Terraform Folder structure for assignement 3"
```

**Step 6** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```



## 1 Build GCP Foundation Layer


In this Lab we going to build GCP Foundation layer. As we learned in the class this includes Org Structure creation with Folders and Projects,
creation IAM Roles and assigning them to users and groups in organization. This Layer usually build be DevOps or Infra team that.

Since we don't have organization registered for each student, we going to skip creation of folders and IAM groups.
We going to start from creation of a Project, deleting `default` VPC and configuring inside that project a `gcs` bucket that will be 
used in the next terraform layer to store a Terraform state.


**Part 1:** Create Terraform Configurations file to create's  GCP Foundation Layer:

Layer 1 From existing  (we going to call it seeding) project that you currently use to store code in Google Cloud Source Repository:

  * Create structure: provider.tf, variable.tf, variables.tfvars, main.tf, output.tf
  * Create a new `notepad-dev` Project
      * Delete Default VPC
  * Create a bucket in this project to store terraform state



**Part 2:** Create Terraform Configurations file that create's  GCP Services Layer:

Layer 2 From `notepad-dev` GCP Project:

  * Enable Google Project Service APIs
  * Create VPC (google_compute_network) and Subnet (google_compute_subnetwork)
  * Create Cloud Nat (google_compute_router) and (google_compute_router_nat)
  * Private GKE
    * Private Nodes with Public API Endpoint (for simplicity)


### 1.1 Installing Terraform


GCP Cloud Shell comes with many common tools pre-installed including `terraform`


Verify and validate the version of Terraform that is installed:

```
terraform --version
```


If you want to use specific version of terraform or want to install terraform in you local machine use following [link](https://www.terraform.io/downloads.html) to download binary.

### 1.2 Configure GCP credentials for Terraform

If you would like to use terraform on GCP you have 2 options:

1). Using you user credentials, great option for testing terraform from you laptop and for learning purposes.

2). Using [service account](https://cloud.google.com/docs/authentication/getting-started#command-line), great option if you going to use terraform with CI/CD system and fully automate Terraform deployment.

In this Lab we going to use Option 1 - using user credentials.


**Step 1:** In order to make requests against the GCP API, you need to authenticate to prove that it's you making the request. 
The preferred method of provisioning resources with Terraform on your workstation is to use the Google Cloud SDK (Option 1) 

```
gcloud auth application-default login
```


### 1.3 Initializing Terraform

Terraform relies on `providers` to interact with remote systems. Every resource type is implemented by a provider; without
`providers`, Terraform can't manage any kind of infrastructure; in order for terraform to install and use a provider it must be
declared.


In this exercise you will declare and configure the Terraform provider(s) that will be used for the rest of the Lab.


#### 1.3.1 Declare Terraform Providers


The `required_providers` block defines the providers terraform will use to provision resources and their source.

  * `version`: The version argument is optional and is used to tell terraform to pick a particular version from the available
versions

  * `source`: The source is the provider registry e.g. hashicorp/gcp is the short for registry.terraform.io/hashicorp/gcp


```
cd ~/$MY_REPO/foundation-infrastructure/
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


#### 1.3.2 Configure the Terraform Provider


Providers often require configuration (like endpoint URLs or cloud regions) before they can be used. These configurations are defined by a `provider` block. Multiple provider configuration blocks can be declared for the same by adding the `alias`
argument


Set you current `PROJECT_ID` value here:

```
export PROJECT_ID=<YOUR_PROJECT_ID>
```

```
cd ~/$MY_REPO/foundation-infrastructure/
cat << EOF>> main.tf
provider "google" {
  alias   = "gcp-provider"
  project = "$PROJECT_ID"
  region  = "us-central1"
}
EOF
```

!!! result
    We configured `provider`, so that it can provision resource in specified gcp project in `us-central1` region.

#### 1.3.3 Initialize Terraform

Now that you have declared and configured the GCP provider for terraform, initialize terraform:

```
cd ~/$MY_REPO/foundation-infrastructure/
terraform init
```

Explore your directory. What has changed?

```
ls -ltra
```

!!! result
    We can see that new directory `.terraform` and `.terraform.lock.hcl` file.


**Extra!**

Investigate available providers in the [Terraform Provider Registry](https://registry.terraform.io/)

Select another provider from the list, add it to your required providers, and to your main.tf

Run terraform init to load your new provider!


 
### 1.4 Terraform Variables

Input variables are used to increase your Terraform configuration's flexibility by defining values that can be assigned to customize the configuration. They provide a consistent interface to change how a given configuration behaves. Input variables blocks have a defined format:


Input variables blocks have a defined format:


```
variable "variable_name" {
  type = "The variable type eg. string , list , number"
  description = "A description to understand what the variable will be used for"
  default = "A default value can be provided, terraform will use this if no other value is provided at terraform apply"
}
```

Terraform CLI defines the following optional arguments for variable declarations:

  * default - A default value which then makes the variable optional.
  * type - This argument specifies what value types are accepted for the variable.
  * description - This specifies the input variable's documentation.
  * validation - A block to define validation rules, usually in addition to type constraints.
  * sensitive - Limits Terraform UI output when the variable is used in configuration.


In our above example for `main.tf` we can actually declare `project` and `region` as variables, so let's do it.


#### 1.4.1 Declare Input Variables

Variables are declared in a `variables.tf` file inside your terraform working directory.

The label after the variable keyword is a name for the variable, which must be unique among all variables in the same module.  You can choose variable name based on you preference, or in some cases based on agreed in company naming convention. In our case we declaring variables names: `gcp_region` and `gcp_project_id`

We are going to declare a type as a  `string` variable for `gcp_project_id` and `gcp_region`.

It is always good idea to specify a clear `description` for the variable for documentation purpose as well as code reusability and readability.

Finally, let's configured `default` argument. In our case we going to use "us-central1" as a `default` for `gcp_region` and we going to keep `default` value empty for `gcp_project_id`.


```
cat <<EOF> variables.tf
variable "gcp_region" {
  type        = string
  description = "The GCP Region"
  default     = "us-central1"
}

variable "gcp_project_id" {
  type        = string
  description = "The GCP Seeding project ID"
  default     = ""
}
EOF
```

!!! note
    Explore other [type](https://www.terraform.io/docs/language/values/variables.html#type-constraints) of variables such as `number`, `bool` and type constructors such as: list(<TYPE>), set(<TYPE>), map(<TYPE>)



#### 1.4.2 Using Variables

Now that you have created your input variables, let's re-create `main.tf` that currently has the terraform configuration for GCP Provider.

```
rm main.tf
cat << EOF>> main.tf
provider "google" {
  alias   = "gcp-provider"
  project = var.gcp_project_id
  region  = var.gcp_region
}
EOF
```

Test out your terraform configuration:

```
terraform plan
```

#### 1.4.3 Working with Variables files.

Create a `terraform.tfvars` file to hold the values for your variables:


```
export gcp_project_id=<YOUR_PROJECT_ID>
```


```
cat <<EOF >> terraform.tfvars
gcp_project_id = "$PROJECT_ID"
gcp_region = "us-central1"
EOF
```

!!! hint
    using different `env-name.tfvars` files you can create different set of terraform configuration for the same code. (e.g. code for dev, staging, prod)


#### 1.4.4 Validate configuration and code syntax


Let's Validate configuration and code syntax that we've added so far.


There are several tools that can analyze your Terraform code without running it, including:


Terraform has build in command that you can use to check your Terraform syntax and types (a bit like a compiler):

```
terraform validate
```

!!! result
    Seems the code is legit so far


Is your terraform easy to read and follow? Terraform has a built-in function to lint your configuration manifests
for readability and best practice spacing:

```
terraform fmt --recursive
```

The `--recursive` flag asks the `fmt` command to traverse all of your terraform directories and format the .tf files it finds. It
will report the files it changed as part of the return information of the command


!!! Hint
    Use the git diff command to see what was changed.

!!! result
    We can see that `terraform.tfvars` file had some spacing that been fixed for better code readability.


!!! extra
    Another cool tool that you can use along you terraform development is [tflint](https://github.com/terraform-linters/tflint/blob/master/README.md) - framework and each feature is provided by plugins, the key features are as follows:

    * Find possible errors (like illegal instance types) for Major Cloud providers (AWS/Azure/GCP).
    * Warn about deprecated syntax, unused declarations.
    * Enforce best practices, naming conventions.



Finally let's run `terraform plan`:

```
terraform plan
```

**Output:**
```
No changes. Your infrastructure matches the configuration.
```


!!! result
    We don't have any errors, however we don't have any resources created so far. Let's create a GCP project resource to start with!


### 1.5 Create GCP project using Terraform 

#### 1.5.1 Configure `google_project` resource

`Resources` describe the infrastructure objects you want terraform to create. A resource block can be used to create any object such as virtual private cloud,
security groups, DNS records etc.

Let's create our first Terraform resource: GCP project.

In order to accomplish this we going to use `google_project` resource, documented [here](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project)


In order to create a new Project we need to define following arguments:
  * name - (Required) The display name of the project.
  * project_id - (Required) The project ID. Changing this forces a new project to be created.
  * billing_account - The alphanumeric ID of the billing account this project belongs to.


First, Let's declare actual values for `name`, `project_id` and `billing_account` with variables:

```
cat <<EOF >> variables.tf
variable "billing_account" {
  description = "The billing account ID for this project"
}

variable "project_name" {
  description = "The human readable project name (min 4 letters)"
}

variable "project_id" {
  description = "The GCP project ID"
}
EOF
```


Then, update `terraform.tfvars` variables files, with actual values for  `name`, `project_id` and `billing_account`:

We going to use same naming structure for `PROJECT_ID` and `PROJECT_NAME` as in previous Assignment, in order to define your new GCP project:

```
export ORG=<student-name>
```
```
export ORG=ayrat
export PRODUCT=notepad
export ENV=dev
export PROJECT_PREFIX=4  
export PROJECT_NAME=$ORG-$PRODUCT-$ENV
export PROJECT_ID=$ORG-$PRODUCT-$ENV-$PROJECT_PREFIX  # Project ID has to be unique
echo $PROJECT_NAME
echo $PROJECT_ID
```

In order to get Billing account run following command:

```
ACCOUNT_ID=$(gcloud alpha billing accounts list | grep Education | grep True |awk '{ print $1 }')
echo $ACCOUNT_ID
```


```
cat <<EOF >> terraform.tfvars
billing_account = "$ACCOUNT_ID"
project_name    = "$PROJECT_NAME"
project_id      = "$PROJECT_ID"
EOF
```


Verify if everything looks good, and correct values has been set:

```
cat terraform.tfvars
```


Finally, let's define `google_project` resource in `project.tf` file, we will replace actual values for `name` and `billing_account` with variables:

```
cat <<EOF >> project.tf
resource "google_project" "project" {
  name                = var.project_name
  billing_account     = var.billing_account
  project_id          = var.project_id
}
EOF
```


Run plan command:

```
terraform plan -var-file terraform.tfvars
```

!!! result
    `Plan: 1 to add, 0 to change, 0 to destroy.`

!!! note
    Take notice of plan command and see how some values that you declared in `*.tvfars` are visible and some values will `known after apply`


```
terraform apply -var-file terraform.tfvars
```


!!! success
    We've created our first resource with terrraform

#### 1.5.2 Making Project resource Immutable


Very often when you developing IaC, you need to destroy and recreate your resorces, e.g. for troubleshooting or creating a new resources using same config.
Let's `destroy` our project and try to recreate it again.


```
terraform destroy -var-file terraform.tfvars
```

```
terraform apply -var-file terraform.tfvars
```


!!! failed
    What happened? Did you project been able to create? If not why ?


If we want to make our infrastructure to be Immutable and fully automated, we need to make sure that we can destroy our service and recreate it any time the same way.
In our case we can't do that because  `Project ID` always has to be unique.
To tackle this problem we need to randomize our `Project ID`  creation within the terraform.


#### 1.5.3 Create a Project using Terraform Random Provider


In order to create a GCP Project with Random name, we can use Terraform's [random_integer resource](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/integer) with following arguments:

* max (Number) The maximum inclusive value of the range - 100
* min (Number) The minimum inclusive value of the range - 999


```
cat <<EOF >> random.tf
resource "random_integer" "id" {
  min = 100
  max = 999
}
EOF
```

`random_integer` resource doesn't belongs to Google Provider, it requires [Hashicorp Random Provider](https://registry.terraform.io/providers/hashicorp/random/latest) to be initialized. Since it's native hashicorp provider we can skip the step of defining and configuring that provider, as it will be automatically initialized. 

```
terraform init
```

!!! result
    - Installing hashicorp/random v3.1.0...
    - Installed hashicorp/random v3.1.0 (signed by HashiCorp)

#### 1.5.3 Create a Project_ID name using Local Values


`Local` Values allow you to assign a name to an expression, so the expression can be used multiple times, without repeating it.

We going to define value of `project_id` as local. And it will be a combination of `project_name` `-` `random_number`.

Let's add local value for `project_id` and replace value `var.project_id` with `local.project_id`.


```
edit project.tf
```

Update code snippet of `project.tf` as following and save:

```
locals {
  project_id     = "${var.project_name}-${random_integer.id.result}"
}

resource "google_project" "project" {
  name                = var.project_name
  billing_account     = var.billing_account
  project_id          = local.project_id
}
```


We can now remove `project_id` value from `terraform.tfvars`, as we going to randomly generate it using `locals` expression with `random_integer`

```
edit terraform.tfvars
```

Remove `project_id` line and save.

We can now also remove `project_id` variable from `variables.tf`:

```
rm variables.tf
cat <<EOF >> variables.tf
variable "gcp_region" {
  type        = string
  description = "The GCP Region"
  default     = "us-central1"
}

variable "gcp_project_id" {
  type        = string
  description = "The GCP Seeding project ID"
  default     = ""
}
variable "billing_account" {
  description = "The billing account ID for this project"
}

variable "project_name" {
  description = "The human readable project name (min 4 letters)"
}
EOF
```

```
terraform plan -var-file terraform.tfvars
```

**Output:**
```
  # random_integer.id will be created
  + resource "random_integer" "id" {
      + id     = (known after apply)
      + max    = 999
      + min    = 100
      + result = (known after apply)
    }
```

```
terraform apply -var-file terraform.tfvars
```

!!! result
    Project has been created with Random project_id

#### 1.5.4 Configure Terraform Output for GCP Project

`Outputs` provide return values of a Terraform state at any given time. So you can get for example values of what GCP resources like project, VMs, GKE cluster been created and their parametres.

`Outputs` can be used used to export structured data about resources. This data can be used to configure other parts of your infrastructure, or as a data source for another Terraform workspace. Outputs are also necessary to share data from a child module to your root module.


Outputs follow a similar structure to variables:

```
output "output_name" {
  description = "A description to understand what information is provided by the output"
  value = "An expression and/or resource_name.attribute"
  sensitive = "Optional argument, marking an output sensitive will supress the value from plan/apply phases"
}
```

Let's declare GCP Project `output` values for `project_id` and `project_number`.

You can find available `output` in each respective `resource` documents, under `[Attributes Reference](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project#attributes-reference).

For example for GCP project available outputs are:

```
cat <<EOF >> outputs.tf
output "id" {
  value = google_project.project.project_id
  description = "GCP project ID"
}

output "number" {
  value = google_project.project.number
  description = "GCP project number"
  sensitive = true
}
EOF
```

```
terraform plan -var-file terraform.tfvars
```

We can see that we have new changes to output:

**Output:**
```
Changes to Outputs:
  + id     = "ayrat-notepad-dev-631"
  + number = (sensitive value)
```


Let's apply this changes:

```
terraform apply -var-file terraform.tfvars
```

**Output:**
```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

Let's now run `terraform output` command:

```
terraform output
```

!!! result
    We can see value of project `id`, however we don't see project number as it's been marked as sensitive.

!!! note
    `Outputs` can be useful when you want to provide results of terraform resource creation to CI/CD or next automation tool like `helm` to deploy application.


#### 1.5.5 Recreate GCP Project without Default VPC

One of the requirements for our solution is to create GCP project with `custom` VPC. However, when terraform creates GCP Project it creates `DEFAULT` vpc by default.

```
gcloud compute networks list --project ayrat-notepad-dev-631
```

**Output:**
```
NAME     SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
default  AUTO         REGIONAL
```

In order to remove automatically created `default` VPC, specify special attribute during `google_project` resource creation. Check documentation [here](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project) and find this argument.


**Task N1:** Find Attribute to remove `default` VPC during  `google_project` resource creation. Define in it `project.tf` and set it's value in `variables.tf` as `true` by default.


```
edit project.tf
TODO
```

```
edit variables.tf
TODO
```

After changes you need to re-create a project, as vpc get's deleted during project creation only.

```
terraform destroy -var-file terraform.tfvars
```

```
terraform plan -var-file terraform.tfvars
terraform apply -var-file terraform.tfvars
```

!!! notice
    How long project is now getting created. This is due to after project creation, `default` vpc is being removed.

Verify that `default` VPC has been deleted:

```
gcloud compute networks list --project ayrat-notepad-dev-631
```

**Output:**
```
Listed 0 items.
```

!!! success
    We now able to create a GCP project without `Default` VPC.

### 1.6 Create GCP Storage bucket

#### 1.6.1 Create GCP Storage bucket in New GCP Project

Using reference doc for [google_storage_bucket
](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/storage_bucket), let's create `google_storage_bucket` resource that will create MULTI_REGIONAL GCS bucket in newly created project. We going to give bucket `name`: `$ORG-notepad-dev-tfstate`, and we going to use this bucket to store Terraform state for GCP Service Layer.

```
cat <<EOF >>  bucket.tf
resource "google_storage_bucket" "state" {
  name          = var.bucket_name
  project       = local.project_id
  storage_class = var.storage_class

  force_destroy = "true"
}
EOF
```

```
cat <<EOF >> variables.tf
variable "bucket_name" {
  description = "The name of the bucket."
}

variable "storage_class" {
  description = 
  default = "MULTI_REGIONAL"
}
EOF
```


Set variable that will be used to build name for a bucket:
```
export ORG=<student-name>
```

```
cat <<EOF >> terraform.tfvars
bucket_name   = "$ORG-notepad-dev-tfstate"
EOF
```

```
cat <<EOF >> outputs.tf
output "bucket_name" {
  value = google_storage_bucket.state.name
}
EOF
```

Let's review the plan:
```
terraform plan -var-file terraform.tfvars
```


And create google_storage_bucket resource:
```
terraform apply -var-file terraform.tfvars
```



Verify created bucket in GCP UI:

```
Storage -> Cloud Storage
```

```
gsutil ls -L -b gs://$ORG-notepad-dev-tfstate
```

!!! result
    GCS bucket for terraform state has been created


#### 1.6.2 Configure `versioning` on GCP Storage bucket.

Check if versioning enabled on the created bucket:

```
gsutil versioning get gs://$ORG-notepad-dev-tfstate
```

!!! result
    Versioning: Suspended

It is highly recommended that if you going to use GCS bucket as [Terraform storage backend](https://www.terraform.io/docs/language/settings/backends/gcs.html) you should enable Object Versioning on the GCS bucket to allow for state recovery in the case of accidental deletions and human error.

**Task N2:** Using reference doc for [google_storage_bucket
](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/storage_bucket) find and configure argument that enables gcs bucket versioning feature.

Edit `bucket.tf` with correct argument.


```
edit bucket.tf
TODO
```

```
terraform plan -var-file terraform.tfvars
```

**Output:**

```
Plan: 0 to add, 1 to change, 0 to destroy.
```

!!! result
    Configuration for Versioning looks correct


```
terraform apply -var-file terraform.tfvars
```

Verify versioning is ON:

```
gsutil versioning get gs://$ORG-notepad-dev-tfstate
```

**Output:**

```
Versioning Enabled
```


!!! result
    We've finished building the Foundation Layer. So far we able to accomplish following:

      * Create structure: provider.tf, variable.tf, variables.tfvars, main.tf, output.tf
      * Create a new `notepad-dev` Project
      * Delete Default VPC
      * Create a bucket in this project to store terraform state



## 2 Build GCP Services Layer


Once we finished building a GCP Foundation layer which is essentially our project, we can start building  GCP Services Layer inside that project.

This second layer will configure following items:

  * Enable Google Project Service APIs
  * Create VPC (google_compute_network) and Subnet (google_compute_subnetwork)
  * Create Cloud Nat (google_compute_router) and (google_compute_router_nat)
  * Private GKE
    * Private Nodes with Public API Endpoint (for simplicity)



### 2.1  Initialize Terraform for GCP Services Layer

#### 2.1.1 Define and configure terraform provider

**Step 1:** Take note of newly created GCP Project_ID and BUCKET_ID:

```
cd ~/$MY_REPO/foundation-infrastructure
```

```
terraform output | grep 'id' |awk '{ print $3}'
terraform output | grep 'bucket_name' |awk '{ print $3}'
```

Set it as variable:

```
export PROJECT_ID=$(terraform output | grep 'id' |awk '{ print $3}')
export BUCKET_ID=$(terraform output | grep 'bucket_name' |awk '{ print $3}')
echo $PROJECT_ID
echo $BUCKET_ID
```


**Step 2:** Declare the Terraform Provider for GCP Services Layer:

We now going to switch to `notepad-infrastructure` where we going to create a new GCP service Layer terraform configuration:

```
cd ~/$MY_REPO/notepad-infrastructure
```

```
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

**Step 4:** Configure the Terraform Provider

```
cat << EOF>> main.tf
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}
EOF
```
**Step 5:** Define variables:

```
cat <<EOF> variables.tf
variable "gcp_region" {
  type        = string
  description = "The GCP Region"
  default     = "us-central1"
}

variable "gcp_project_id" {
  type        = string
  description = "The newly created GCP project ID"
}
EOF
```


**Step 5:** Set variables in `terraform.tfvars`

```
cat <<EOF >> terraform.tfvars
gcp_project_id = $PROJECT_ID
EOF
```

**Step 4:** Now that you have declared and configured the GCP provider for terraform, initialize terraform:

```
terraform init
```


#### 2.1.2 Configure Terraform State backend

Terraform records information about what infrastructure it created in a Terraform state file. This information, or state, is stored by
default in a `local` file named terraform.tfstate. This allows Terraform to compare what's in your configurations with what's in
the state file, and determine what changes need to be applied.


**Local vs Remote backend:**
`Local` state files are the default for terraform. `Remote` state allows for collaboration between members of your team, for example multiple users or systems can deploy terraform configuration to the same environment as state is stored remotely and can be pulled `localy` during the execution. This however may create issues if 2 users running terraform `plan` and `apply` at the same time, however some remote backends including gcs provide `locking` feature that we covered and enabled in previous section `Configure versioning on GCP Storage bucket.`

In addition to that, `Remote` state is more secure storage of sensitive values that might be contained in variables or outputs.

See documents for[remote](https://www.terraform.io/docs/language/settings/backends/remote.html) backend for reference:

**Step 1** Configure remote backend using gcs bucket:

Let's configure a backend for your state, using the `gcs` bucket you previously created in Foundation Layer.

Backends are configured with a nested backend block within the top-level terraform block While a backend can be declared
anywhere, it is recommended to use a `backend.tf`.

Since we running on GCP we going to use [GCS remote backend](https://www.terraform.io/docs/language/settings/backends/gcs.html).
It stores the state as an object in a configurable prefix in a pre-existing bucket on Google Cloud Storage (GCS). This backend also supports state locking. The bucket must exist prior to configuring the backend.

We going to use following arguments:

  * bucket - (Required) The name of the GCS bucket. This name must be globally unique. For more information, see Bucket Naming Guidelines.
  * prefix - (Optional) GCS prefix inside the bucket. Named states for workspaces are stored in an object called <prefix>/<name>.tfstate.

```
cat <<EOF >> backend.tf
terraform {
  backend "gcs" {
    bucket = $BUCKET_ID
    prefix = "state"
  }
}
EOF
```

**Step 2** 
When you change, configure or unconfigure a backend, terraform must be re-initialized:

```
terraform init
```

Verify if `folder` state has been created in our bucket:

```
gsutil ls gs://$ORG-notepad-dev-tfstate
```

!!! summary
    `state` Folder has been created in gcs bucket and terrafrom has been initialized with  `remote` backend



### 2.2 Enable required GCP Services API


Before we continue with creating GCP services like VPC, routers, Cloud Nat and GKE it is required to enable underlining GCP API services.
When we creating a new project most of the services API's are disabled, and requires explicitly to be enabled.
`google_project_service` [resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_service) allows management of a single API service for an existing Google Cloud Platform project.


Let's enable `compute engine API` service with terraform:

```
cat <<EOF >> services.tf
resource "google_project_service" "compute" {
  service = "compute.googleapis.com"
  disable_on_destroy = false
}
EOF
```


```
terraform plan -var-file terraform.tfvars
terraform apply -var-file terraform.tfvars
```


Verify Compute Engine API service has been enabled:

```
gcloud services list
```

!!! result
    Compute Engine API is enabled as it is listed.


### 2.2 Set variables to define standard for Naming Convention

As per terraform best practices, when you creating terraform resources, you need to follow naming convention, that is clear for you organization.

  * Configuration objects should be named using underscores to delimit multiple words.
  * Object's name should be named using dashes

This practice ensures consistency with the naming convention for resource types, data source types, and other predefined values and helps prevent accidental deletion or outages:

Example:
```
# Good
resource "google_compute_instance" "web_server" {
  name = “web-server-$org-$app-$env”
  # ...
}

# Bad
resource “google_compute_instance” “web-server” {
  name =  “web-server”
  # …
```

Create variables to define standard for Naming Convention:

```
cat <<EOF >> variables.tf
variable "org" {
  type = string
}
variable "product" {
  type = string
}
variable "environment" {
  type = string
}
EOF
```


```
export ORG=ayrat
```

```
export PRODUCT=notepad
export ENV=dev
```

```
cat <<EOF >> terraform.tfvars
org            = "$ORG"
product        = "$PRODUCT"
environment    = "$ENV"
EOF
```

Review if created files are correct.

### 2.3 Create a `custom mode` network (VPC) with terraform

Using [google_compute_network](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network) resource create a VPC network with terraform.

**Task N3:** Create `vpc.tf` file and define `custom mode` network (VPC) with following requirements:
  * Description: `VPC that will be used by the GKE private cluster on the related project`
  * `name` - vpc name with following pattern: "vpc-$ORG-$PRODUCT-$ENV"
  * Subnet mode: `custom`
  * Bgp routing mode: `regional`
  * MTUs: `default`

!!! hint
    To create "vpc-$ORG-$PRODUCT-$ENV" name, use [format function](https://www.terraform.io/docs/language/functions/format.html), and use `%s` to convert variables to string values.


Define variables in `variables.tf` and `terraform.tfvars` if required.

Define `Output` for:

  * Generated VPC name: `google_compute_network.vpc_network.name`
  * self_link - The URI of the created resource.


```
cat <<EOF >> vpc.tf
TODO
EOF
```

```
cat <<EOF >> outputs.tf
TODO
EOF
```

Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```


**Output:**
```
  + create

Terraform will perform the following actions:
  # google_compute_network.vpc_network will be created
  + resource "google_compute_network" "vpc_network" {
      + auto_create_subnetworks         = false
      + delete_default_routes_on_create = false
      + description                     = "VPC that will be used by the GKE private cluster on the related project"
      + gateway_ipv4                    = (known after apply)
      + id                              = (known after apply)
      + mtu                             = (known after apply)
      + name                            = "vpc-ayrat-notepad-dev"
      + project                         = (known after apply)
      + routing_mode                    = "REGIONAL"
      + self_link                       = (known after apply)
```

Create VPC:
```
terraform apply -var-file terraform.tfvars
```


Verify that `custom-mode` VPC has been created:

```
gcloud compute networks list
gcloud compute networks describe $(gcloud compute networks list | grep CUSTOM |awk '{ print $1 }')
```

**Output:**
```
autoCreateSubnetworks: false
description: VPC that will be used by the GKE private cluster on the related project
kind: compute#network
name: vpc-student_name-notepad-dev
routingConfig:
  routingMode: REGIONAL
selfLink: https://www.googleapis.com/compute/v1/projects/XXX
x_gcloud_bgp_routing_mode: REGIONAL
x_gcloud_subnet_mode: CUSTOM
```

!!! result
    VPC Network has been created, without auto subnets.

### 2.4 Create a `user-managed` subnet with terraform

Using [google_compute_subnetwork](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_subnetwork) resource create a `user-managed` subnet with terraform.


```
cat <<EOF >> subnet.tf
resource "google_compute_subnetwork" "gke_private_subnet" {
  name                     = format("subnet-%s-%s-%s", var.org, var.product, var.environment)
  network                  = google_compute_network.vpc_network.self_link
  region                   = var.gcp_region
  project                  = var.gcp_project_id
  ip_cidr_range            = var.network_cidr
  secondary_ip_range {
    range_name    = var.pods_cidr_name
    ip_cidr_range = var.pods_cidr
  }
  secondary_ip_range {
    range_name    = var.services_cidr_name
    ip_cidr_range = var.services_cidr
  }
}
EOF
```

!!! note
    Notice power of terraform outputs. Here we link `subnet` with our VPC `network` using `google_compute_network.vpc_network.self_link` `output` value of created `network` in previous step.

Define variables:

```
cat <<EOF >> variables.tf

# variables used to create VPC

variable "network_cidr" {
  type = string
}
variable "pods_cidr" {
  type = string
}
variable "pods_cidr_name" {
  type = string
}
variable "services_cidr" {
  type = string
}
variable "services_cidr_name" {
  type = string
}
EOF
```

Define outputs:

```
cat <<EOF >> outputs.tf
output "subnet_selflink" {
  value = "\${google_compute_subnetwork.gke_private_subnet.self_link}"
}
EOF
```


**Task N4:** Update `terraform.tfvars` file values with following information:

  * Node Range: See column `subnet` in above table for `dev` cluster
  * Secondary service range with name: `services`
  * Secondary Ranges:
    * Service range name: `services`
    * Service range CIDR: See column `srv range` in above table for `dev` cluster
    * Pods range name: `pods`
    * Pods range CIDR: See column `pod range` in above table for `dev` cluster

```
env |   subnet      | pod range        | srv range      | kubectl api range
dev | 10.128.1.0/26 | 172.0.0.0/18    | 172.10.0.0/21   | 172.16.0.0/28
stg | 10.128.2.0/26 | 172.1.0.0/18    | 172.11.0.0/21   | 172.16.0.16/28
prd | 10.128.3.0/26 | 172.2.0.0/18    | 172.12.0.0/21   |  172.16.0.32/28
```

!!! note
    Ranges must be with in Private (RFC1918) [Address Space](https://en.wikipedia.org/wiki/Private_network)


```
cat <<EOF >> terraform.tfvars
#subnet vars
network_cidr       = "TODO"
pods_cidr          = "TODO"
pods_cidr_name     = "TODO"
services_cidr      = "TODO"
services_cidr_name = "TODO"
EOF
```



Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

Create VPC:
```
terraform apply -var-file terraform.tfvars
```

Review created subnet:

```
gcloud compute networks subnets list
gcloud compute networks subnets describe $(gcloud compute networks subnets list | grep us-central1 |awk '{ print $1 }')  --region us-central1
```

**Output:**
```
enableFlowLogs: false
gatewayAddress: 10.128.1.1
ipCidrRange: 10.128.1.0/26
kind: compute#subnetwork
logConfig:
  enable: false
name: subnet-student_name-notepad-dev
privateIpGoogleAccess: false
privateIpv6GoogleAccess: DISABLE_GOOGLE_ACCESS
purpose: PRIVATE
secondaryIpRanges:
- ipCidrRange: 172.0.0.0/18
  rangeName: pods
- ipCidrRange: 172.10.0.0/21
  rangeName: services
stackType: IPV4_ONLY
```

Also check in Google cloud UI:

```
Networking->VPC Networks -> Click VPC network and check `Subnet` tab
```

**Task N5:** Update  `subnet.tf` so that  `google_compute_subnetwork` resource supports following features:

    * Flow Logs
      * Aggregation interval: 15 min
      * Flow sampling: 0.1
      * Metadata: "INCLUDE_ALL_METADATA"
    * Private IP Google Access


```
edit subnet.tf
TODO
```


Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

Create VPC:
```
terraform apply -var-file terraform.tfvars
```

### 2.5 Create a Cloud router

Create Cloud Router for `custom mode` network (VPC), in the same region as the instances that will use Cloud NAT. Cloud NAT is only used to place NAT information onto the VMs. It is not used as part of the actual NAT gateway.



**Task N6:** Define a `google_compute_router` inside `router.tf` that will be able to create a NAT router so the nodes can reach DockerHub and external APIs from private cluster, using following parameters:
  
  * Create router for custom `vpc_network` created above with terraform
  * Same project as VPC
  * Same region as VPC
  * Router name: `gke-net-router`
  * Local BGP Autonomous System Number (ASN): 64514
  

!!! hint
    You can automatically recover vpc name from terraform output like this: `google_compute_network.vpc_network.self_link`.

```
cat <<EOF >> router.tf
TODO
EOF
```


Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

Create Cloud Router:
```
terraform apply -var-file terraform.tfvars
```

Verify created Cloud Router:

CLI:
```
gcloud compute routers list
gcloud compute routers describe $(gcloud compute routers list | grep gke-net-router |awk '{ print $1 }')  --region us-central1

```

**Output:**
```
bgp:
  advertiseMode: DEFAULT
  asn: 64514
  keepaliveInterval: 20
kind: compute#router
name: gke-net-router
```


UI:
```
Networking -> Hybrid Connectivity -> Cloud Routers
```

!!! result
    Router resource has been created for VPC Network

### 2.6 Create a Cloud Nat

Set up a simple Cloud Nat configuration using [google_compute_router_nat](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_router_nat) resource, which will `automatically` allocates the necessary external IP addresses to provide NAT services to a region. 

When you use auto-allocation, Google Cloud reserves IP addresses in your project automatically. 


```
cat <<EOF >> cloudnat.tf
resource "google_compute_router_nat" "gke_cloud_nat" {
  project                = var.gcp_project_id
  name                   = "gke-cloud-nat"
  router                 = google_compute_router.gke_net_router.name
  region                 = var.gcp_region
  nat_ip_allocate_option = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
EOF
```

Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

Create Cloud Router:
```
terraform apply -var-file terraform.tfvars
```

Verify created `Cloud Nat`:

CLI:
```
# List available Cloud Nat Routers
gcloud compute routers nats list --router gke-net-router --router-region us-central1
# Describe Cloud Nat Routers `gke-cloud-nat`:
gcloud compute routers nats describe gke-cloud-nat --router gke-net-router --router-region us-central1
```

**Output:**

```
enableEndpointIndependentMapping: true
icmpIdleTimeoutSec: 30
name: gke-cloud-nat
natIpAllocateOption: AUTO_ONLY
sourceSubnetworkIpRangesToNat: ALL_SUBNETWORKS_ALL_IP_RANGES
tcpEstablishedIdleTimeoutSec: 1200
tcpTransitoryIdleTimeoutSec: 30
udpIdleTimeoutSec: 30
```

UI:
```
Networking -> Network Services -> Cloud NAT
```

!!! result
    A NAT service created in a router

**Task N7:** Additionally turn `ON` logging feature for `ALL` log types of communication for Cloud Nat

```
edit cloudnat.tf
TODO
```

Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

Update Cloud Nat Configuration:

```
terraform apply -var-file terraform.tfvars
```

**Output:**
```
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

```
gcloud compute routers nats describe gke-cloud-nat --router gke-net-router --router-region us-central1
```

!!! result
    Cloud Nat now supports Logging. Cloud NAT logging allows you to log NAT connections and errors. When Cloud NAT logging is enabled, one log entry can be generated for each of the following scenarios:

      * When a network connection using NAT is created.
      * When a packet is dropped because no port was available for NAT.


### 2.7 Create SSH Firewall rules `default-allow-internal` and `default-allow-ssh`

Let's create SSH Firewall rule with name `allow-tcp-ssh-icmp-$ORG-$PRODUCT-$ENV` to allow SSH, ping, using [google_compute_firewall](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_firewall) resource.

Reference: 

  * https://cloud.google.com/kubernetes-engine/docs/concepts/firewall-rules
  * https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_firewall

```
cat <<EOF>> firewall.tf
resource "google_compute_firewall" "ssh-rule" {
  name =   format("allow-tcp-ssh-icmp-%s-%s-%s", var.org, var.product, var.environment)
  network = google_compute_network.vpc_network.self_link
  allow {
    protocol = "tcp"
    ports = ["22"]
  }
  allow {
    protocol = "icmp"
  }
}
EOF
```


Review created firewall rules:

```
gcloud compute firewall-rules list
```

Also check in Google cloud UI:

```
Networking->Firewalls
```


### 2.8 Create a Private GKE Cluster and delete default node pool

#### 2.8.1 Enable GCP Beta Provider

In order to create a GKE cluster with terraform we will be leveraging [google_container_cluster](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_cluster) resource.

Some of  `google_container_cluster` arguments, such VPC-Native networking mode, VPA, Istio, CSI Driver add-ons, requires  `google-beta` Provider.

The `google-beta` provider is distinct from the `google` provider in that it supports GCP products and features that are in beta, while google does not. Fields and resources that are only present in google-beta will be marked as such in the shared provider documentation.


**Task N8:** Configure and Initialize [GCP Beta Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_versions), similar to how we did it for GCP Provider in [1.3.3 Initialize Terraform](#133-initialize-terraform) update `provider.tf` and `main.tf` configuration files.


```
edit provider.tf
TODO
```

```
edit main.tf
TODO
```

Initialize `google-beta` provider plugin:

```
terraform init
```

!!! success
    Terraform has been successfully initialized!


#### 2.8.2 Enable Kubernetes Engine API

[Kubernetes Engine API](https://cloud.google.com/kubernetes-engine/docs/reference/rest) used to build and manages container-based applications, powered by the open source Kubernetes technology. Before starting  GKE cluster creation it is required to enable it.

**Task N9:** Enable `container.googleapis.com` in `services.tf` file similar to what we already did in [2.2 Enable required GCP Services API](#22-enable-required-gcp-services-api).


```
edit services.tf
TODO
```

!!! note
    Adding `disable_on_destroy=false` helps to prevent errors during redeployments of the system.

Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

Update Cloud Nat Configuration:

```
terraform apply -var-file terraform.tfvars
```


#### 2.8.3 Create a Private GKE Cluster and delete default node pool

Using [google_container_cluster](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_cluster) resource create a Regional, Private GKE cluster, with following characteristics:

Cluster Configuration:

  * Cluster name: `gke-$ORG-$PRODUCT-$ENV`
  * GKE Control plane is replicated across three zones of a region: `us-central1`
  * Private cluster with unrestricted access to the public endpoint:
    * Cluster Nodes access: *Private* Node GKE Cluster with *Public API* endpoint
    * Cluster K8s API access: *with unrestricted access to the public endpoint*
  * Cluster Node Communication: `VPC Native`
  * Secondary pod range with name: `pods`
  * Secondary service range with name: `services`
  * GKE master and node version: "1.20.8-gke.700"
  * Terraform Provider: `google-beta`
  * Timeouts: 30M


Node Pool Configuration:

  * VM size: `e2-small`
  * Node count: 1 per zone
  * Node images: Container-Optimized OS
  * The name of a GCE machine (VM) type: e2-small


!!! note
    **Why delete default node pool?** The `default` node pools cause trouble with managing the cluster, when created with terraform as it is not part of the terraform lifecycle. GKE Architecture Best Practice recommends to delete `default` node pool and create a `custom` one instead and manage the node pools explicitly.

!!! note
    **Why define `Timeouts` for gke resource?** Normally GKE creation takes few minutes. However, in our case we creating GKE Cluster, and then system cordon, drain and then destroy default node pool. This process may take 10-20 minutes and we want to make sure terraform will not time out during this time.

**Step 1:** Let's define GKE resource first:

```
cat <<EOF >> gke.tf
resource "google_container_cluster" "primary_cluster" {
  provider = google-beta

  project = var.gcp_project_id

  name               = format("gke-%s-%s-%s", var.org, var.product, var.environment)
  min_master_version = var.kubernetes_version
  network            = google_compute_network.vpc_network.self_link
  subnetwork         = google_compute_subnetwork.gke_private_subnet.self_link

  location                    = var.gcp_region
  logging_service             = var.logging_service
  monitoring_service          = var.monitoring_service

  remove_default_node_pool = true
  initial_node_count       = 1

  private_cluster_config {
    enable_private_nodes   = var.enable_private_nodes
    enable_private_endpoint = var.enable_private_endpoint
    master_ipv4_cidr_block = var.master_ipv4_cidr_block
  }

  network_policy {
    enabled  = var.network_policy
    provider = var.network_policy ? "CALICO" : "PROVIDER_UNSPECIFIED"
  }

  addons_config {
    http_load_balancing {
      disabled = var.disable_http_load_balancing
    }

    network_policy_config {
      disabled = var.network_policy ? false : true
    }
  }

  ip_allocation_policy {
    cluster_secondary_range_name  = var.pods_range_name
    services_secondary_range_name = var.services_range_name
  }

  timeouts {
    create = "30m"
    update = "30m"
    delete = "30m"
  }

  workload_identity_config {
    identity_namespace = "${var.gcp_project_id}.svc.id.goog"
  }
}
EOF
```

**Step 2:** Next define GKE cluster specific variables:

```
cat <<EOF >> gke_variables.tf

variable "kubernetes_version" {
  default     = ""
  type        = string
  description = "The GKE version of Kubernetes"
}

variable "logging_service" {
  description = "The logging service that the cluster should write logs to."
  default     = "logging.googleapis.com/kubernetes"
}

variable "monitoring_service" {
  default     = "monitoring.googleapis.com/kubernetes"
  description = "The GCP monitoring service scope"
}

variable "disable_http_load_balancing" {
  default     = false
  description = "Enable HTTP Load balancing GCP integration"
}

variable "network_policy" {
  description = "Enable network policy addon"
  default     = true
}

variable "pods_range_name" {
  description = "The pre-defined IP Range the Cluster should use to provide IP addresses to pods"
  default     = ""
}

variable "services_range_name" {
  description = "The pre-defined IP Range the Cluster should use to provide IP addresses to services"
  default     = ""
}

variable "enable_private_nodes" {
  default     = false
  description = "Enable Private-IP Only GKE Nodes"
}

variable "enable_private_endpoint" {
  default     = false
  description = "When true, the cluster's private endpoint is used as the cluster endpoint and access through the public endpoint is disabled."
}

variable "master_ipv4_cidr_block" {
  description = "The ipv4 cidr block that the GKE masters use"
}
EOF
```

**Step 3:** Define GKE cluster specific outputs:

```
cat <<EOF >> outputs.tf
output "id" {
  value = "${google_container_cluster.primary_cluster.id}"
}
output "endpoint" {
  value = "${google_container_cluster.primary_cluster.endpoint}"
}
output "master_version" {
  value = "${google_container_cluster.primary_cluster.master_version}"
}
EOF
```


**Task N9:** Complete `terraform.tfvars` with required values to GKE Cluster specified above:


```
cat <<EOF >> terraform.tfvars
//gke specific
enable_private_nodes   = "TODO"
master_ipv4_cidr_block = "TODO"
pods_range_name        = "TODO"
services_range_name    = "TODO"
kubernetes_version     = "TODO"
EOF
```

In the next step, we going to create a `custom` GKE Node Pool.

#### 2.8.4 Create a GKE `custom` Node pool

Using [google_container_node_pool](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_node_pool) resource create a `custom` GKE Node Pool with following characteristics:

Node Pool Configuration:

  * VM size: `e2-small`
  * Node count: 1 per zone
  * Node images: Container-Optimized OS
  * The name of a GCE machine (VM) type: e2-small


**Step 1:** Let's define GKE resource first:

```
cat <<EOF >> gke.tf
#Node Pool Resource
resource "google_container_node_pool" "custom-node_pool" {
  provider = google-beta
  
  name       = "main-pool"
  location = var.gcp_region
  project  = var.gcp_project_id
  cluster    = google_container_cluster.primary_cluster.name
  node_count = var.gke_pool_node_count
  version    = var.kubernetes_version

  node_config {
    image_type   = var.gke_pool_image_type
    disk_size_gb = var.gke_pool_disk_size_gb
    disk_type    = var.gke_pool_disk_type
    machine_type = var.gke_pool_machine_type
  }

  timeouts {
    create = "10m"
    delete = "10m"
  }

  lifecycle {
    ignore_changes = [
      node_count
    ]
  }
}
EOF
```

**Step 2:** Next define GKE cluster specific variables:

```
cat <<EOF >> gke_variables.tf

#Node Pool specific variables
variable "gke_pool_machine_type" {
  type = string
}
variable "gke_pool_node_count" {
  type = number
}
variable "gke_pool_disk_type" {
  type = string
  default = "pd-standard"
}
variable "gke_pool_disk_size_gb" {
  type = string
}
variable "gke_pool_image_type" {
  type = string
}
EOF
```

**Task 9 (Continued):** Complete `terraform.tfvars` with required values to GKE Node Pool values specified above:

```
cat <<EOF >> terraform.tfvars
#pool specific
gke_pool_node_count   = "TODO"
gke_pool_image_type   = "TODO"
gke_pool_disk_size_gb = "TODO"
gke_pool_machine_type = "TODO"
EOF
```

**Step 3:** Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

**Step 4:** Create GKE Cluster and Node Pool:

```
terraform apply -var-file terraform.tfvars
```

**Output:**

```
google_container_cluster.primary_cluster: Creating...
...
google_container_cluster.primary_cluster: Creation complete after 20m9s 
google_container_node_pool.custom-node_pool: Creating...
google_container_node_pool.custom-node_pool: Creation complete after 2m10s
```


#### 2.8.5 Update GKE Node Pool to support Auto Upgrade and Auto Recovery features


!!! note
    GKE Master Nodes are managed by Google and get's upgraded automatically. Users can only specify Maintenance Window if they have preference for that process to occur (e.g. after busy hours). Users can however control Node Pool upgrade lifecycle. The can choose to do it themselves or with Auto Upgrade.

**Task N10:** Using [google_container_node_pool](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_node_pool) resource update node pool to support Auto Upgrade and Auto Recovery features.

```
edit gke.tf
TODO
```

**Step 3:** Review TF Plan:

```
terraform plan -var-file terraform.tfvars
```

No errors.

**Step 4:** Update GKE Cluster Node Pool configuration:

```
terraform apply -var-file terraform.tfvars
```

!!! summary
    Congrats! You've now learned how to deploy production grade GKE clusters.


### 2.9 (Optional) Repeatable Infrastructure

When you doing IaC it is important to insure that you can both create and destroy resources consistently.
This is especially important when doing CI/CD testing.


**Step 3:** Destroy all resources:

```
terraform destroy -var-file terraform.tfvars
```

No errors.

**Step 4:** Recreate all resources:

```
terraform plan -var-file terraform.tfvars
terraform apply -var-file terraform.tfvars
```

# 3. Create Documentation for terraform code

Documentation for your terraform code is an important part of IaC. Make sure all your variables have a good description!

There are community tools that have been developed to make the documentation process smoother, in terms of documenting
Terraform resources and requirements.Its good practice to also include a usage example snippet.

[Terraform-Docs](https://terraform-docs.io/user-guide/introduction/) is a good example of one tool that can generate some documentation based on the description argument of your Input Variables, Output Values, and from your required_providers configurations.


**Step 1** Install the terraform-docs cli to your Google CloudShell environment: 

```
curl -Lo ./terraform-docs.tar.gz https://github.com/terraform-docs/terraform-docs/releases/download/v0.14.1/terraform-docs-v0.14.1-$(uname)-amd64.tar.gz
tar -xzf terraform-docs.tar.gz
rm terraform-docs.tar.gz
chmod +x terraform-docs
sudo mv terraform-docs /usr/local/bin/
terraform-docs
```

Generating terraform documentation with Terraform Docs:

```
cd ~/$MY_REPO
cd foundation-infrastructure
terraform-docs markdown . > README.md
cd ../notepad-infrastructure
terraform-docs markdown . > README.md
```

Verify created documentation:

```
edit README.md
```


# 4. Workaround for Project Quota issue

If you see following error during project creation in foundation layer:

```
Error: Error setting billing account "010BE6-CA1129-195D77" for project "projects/ayrat-notepad-dev-244": googleapi: Error 400: Precondition check failed., failedPrecondition
```

This is due to our Billing account has quota of 5 projects per account.

To solve this issue find all unused accounts:

```
gcloud beta billing projects list --billing-account $ACCOUNT_ID
```

And unlink them, so you have less then 5 projects per account:

```
gcloud beta billing projects unlink $PROJECT_ID
```

# 5 Commit Readme doc to repository and share it with Instructor/Teacher

**Step 1** Commit `docs` folder using the following Git commands:


```
cd ~/$MY_REPO
```

```
git add .
git commit -m "Readme doc for Production GKE Creation using gcloud"
```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```

# 6 Cleanup

We only going to cleanup GCP Service foundation layer, as we going to use GCP project in future.

```
cd ~/$MY_REPO/notepad-infrastructure
terraform destroy -var-file terraform.tfvars
```
