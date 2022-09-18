# 1 Containerize Applications

**Objective:**

  * Use GCP Cloud Source Repositories to commit code
  * Review process of containerizing of applications
  * Review creation of Docker Images
  * Review build image process
  * Review process to launch containers with docker cli


## Prepare Lab Environment

This lab can be executed in you GCP Cloud Environment using Google Cloud Shell.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")


Once opened, you can use it to run the instructions for this lab.


## 1 Configure Cloud Source Repository

Google Cloud Source Repositories provides Git version control to support collaborative
development of any application or service. In this lab, you will create a local Git
repository that contains a sample file, add a Google Source Repository as a remote, 
and push the contents of the local repository. You will use the source browser included 
in Source Repositories to view your repository files from within the Cloud Console.

!!! note
    Google Cloud Source Repository is similar tool to GitHub, so if you know Github CLI it is same experience, apart from that UI is different than GITHUB.

### 1.1 Create a personal repository within `Google Cloud Source Repository`


**Step 1** Run the following command to create a new Cloud Source Repository named $student_name-notepad, where $student_name - is you mcgill student-ID:


Setup Environment Variable: 

```
export PROJECT_ID=<project_id>
```


Here is how you can find you project_ID:

![Project ID](/images/porject_id_png.png "Locate project_id")


```
export student_name=<write_your_name_here_and_remove_brakets>
```

!!! important
    Replace above with your project_id student_name


```
gcloud config set project $PROJECT_ID
gcloud source repos create $student_name-notepad
```


Press Y

```
API [sourcerepo.googleapis.com] not enabled on project [686694291909]. Would you like to enable and retry (this will take a few minutes)? (y/N)?
```

You can safely ignore any billing warnings for creating repositories.

**Step 3** Clone the contents of your new Cloud Source Repository to a local repo in your Cloud Shell session:

```
gcloud source repos clone $student_name-notepad
```

The `gcloud source repos clone` command adds Cloud Source Repositories as a remote named origin and clones it into a local Git repository.


**Step 3** Go into the local repository you've created:

```
ls
```

Observe that repository has been cloned


!!! result
    You've created a Personal Repository, this is the locaton where you going to submit you assignments going forward


###  1.2 Locate Module 5 Assignment 

**Step 1** Locate directory where Dockerfile and Readme.md are stored.

```
cd ~
git clone https://github.com/Cloud-Architects-Program/ycit019_2022
cd ~/ycit019_2022/Mod5_assignment/
ls
```

!!! result 
    You can see Readme.md where you will document docker commands given in the assignment

```
ls gowebapp-mysql
ls gowebapp
```

!!! result 
    You can see Dockerfiles for gowebapp and  gowebapp-mysql that you will be working with in this Assignment


**Step 2** Go into your personal Google Cloud Source Repository:

```
cd ~
cd ~/$student_name-notepad
```


**Step 3** Copy Mod 5 Assignment `Mod5_assignment` folder to your personal repo:

```
cp -r ~/ycit019_2022/Mod5_assignment .
```

**Step 4** Configure Git Parametres

```
git config --global user.email "you@example.com"   #You GCP Account User
git config --global user.name "Your Name"
```


**Step 5** Commit `deploy` folder using the following Git commands:

```
git status 
git add .
git commit -m "adding Dockerfiles and Readme for Module 5 Assignement"
```

**Step 5** Once you've committed code to the local repository, add its contents to Cloud Source Repositories using the git push command:

```
git push origin master
```

### 1.3 Review Cloud Source Repositories


Use the `Google Cloud Source Repositories` code browser to view repository files. 
You can filter your view to focus on a specific branch, tag, or comment.

**Step 1** Browse the Mod5_assignment files you pushed to the repository by opening the Navigation menu and selecting Source Repositories:

```
Click Menu -> Source Repositories > Source Code.
```

!!! result
    The console shows the files in the master branch at the most recent commit.


**Step 2** View a file in the Google Cloud repository

```
Click $student_name-notepad > gowebapp/Dockerfile to view content of the Dockerfile for gowebapp
```

```
Click $student_name-notepad > gowebapp/Dockerfile to view content of the Dockerfile for gowebapp-mysql
```

```
Click $student_name-notepad > Readme.md to view content of the Readme
```



### 1.4 Grant viewing permissions for a repository to Instructors/Teachers

Reference [document](https://cloud.google.com/source-repositories/docs/granting-users-access)

**Step 1** This step will grant view access for Instructor to check you assignments

In your Cloud Terminal:

```
gcloud projects add-iam-policy-binding $PROJECT_ID --member='user:ayrat.khayretdinov@gmail.com' --role=roles/viewer
```

!!! result
    Your instructor will be able to review you code and grade it.


### 1.5 Browse and edit files in `Cloud Shell Editor`


**Step 1**  Browse files in the `Google Cloud Source repository` 

```
edit ~/$student_name-notepad
```

!!! result
    Editor opens and you can easily modify you code and save it as you go.


## 2 Build and Deploy gowebapp application 

### 2.1 Overview of the Sample Application

This package contains two application components which will be used
throughout the course to demonstrate features of Kubernetes:

* gowebapp

This directory contains a simple Go-based note-taking web application. It
includes a home page, registration page, login page, about page, and note
management page.

Configuration for this application is stored in code/config/config.json. This
file is used to configure the backing data store for the application, which in
this course is MySQL.

Later in the course, we will externalize this configuration in order to take
advantage of Kubernetes Secrets and ConfigMaps. This will include some minor
modifications to the Go source code. Go programming experience is not required
to complete the exercises.

For more details about the internal design and implementation of the Go web
application, see code/README.md.

* gowebapp-mysql

This directory contains the schema file used to setup the backing MySQL database
for the Go web application.

### 2.3 Build Dockers image for frontend application

**Step 1** Locate and review the go source code:

```
cd ~/$student_name-notepad/Mod5_assignment/
```

!!! result
    Two folders with go app and mysql config has been reviewed.


**Step 2** Write `Dockerfile` for your frontend application

```
cd ~/$student_name-notepad/Mod5_assignment/gowebapp
```

Modify a file named `Dockerfile` in this directory for the frontend Go app.
Use Cloud Editor or editor of you choice

```
edit Dockerfile
```

The template below provides a starting point for defining the contents of this
file. Replace TODO comments with the appropriate commands:

```
#TODO --- Define this image to inherit from the "golang" base image. Use version `1.15.11` or lower for `golang`
#https://hub.docker.com/_/golang/
#https://docs.docker.com/engine/reference/builder/#from


#TODO Set a label corresponding to the MAINTAINER field you could use, so that it wil be visible from docker inspect with the other labels.
#MAINTAINER should be you student e-mail.

#TODO --- Define a version label for this image
#https://docs.docker.com/engine/reference/builder/#label

EXPOSE 80

ENV GOPATH=/go

#TODO --- Copy source code in the local /code directory into $GOPATH/src/gowebapp
#https://docs.docker.com/engine/reference/builder/#copy

WORKDIR $GOPATH/src/gowebapp/

RUN go get && go install

#TODO --- Define an entrypoint for this image which executes the compiled application in $GOPATH/bin/gowebapp when the container starts
#https://docs.docker.com/engine/reference/builder/#entrypoint
```

**Step 4**  Build gowebapp Docker image locally

Build the  image locally. Make sure to include "." at the end. Make sure the
build runs to completion without errors. You should get a success message.

```
#TODO  Build image `<your-github-user>/gowebapp:v1
```

### 2.4 Build Docker image for backend application

**Step 1** Locate folder with mysql config

```
cd ~/$student_name-notepad/Mod5_assignment/gowebapp-mysql
```

**Step 2** Write Dockerfile for your backend application

Create a file named `Dockerfile` in this directory for the backend MySQL database
application. Use Cloud Editor or editor of you choice.

```
edit Dockerfile
```

The template below provides a starting point for defining the contents of this
file. Replace TODO comments with the appropriate commands:

```
#TODO --- Define this image to inherit from the "mysql" version 8.0 base image
#https://hub.docker.com/_/mysql/
#https://docs.docker.com/engine/reference/builder/#from

#TODO Set a label corresponding to the MAINTAINER field you could use, so that it wil be visible from docker inspect with the other labels.
#MAINTAINER should be you student e-mail.

LABEL gowebapp-mysql "v1"

#TODO --- Investigate the "Initializing a Fresh Instance" instructions for the mysql parent image, and copy the local gowebapp.sql file to the proper container directory to be automatically executed when the container starts up
#https://hub.docker.com/_/mysql/
#https://docs.docker.com/engine/reference/builder/#copy
```

**Step 2** Build gowebapp-mysql Docker image locally

```
#TODO  Build image  <your-github-user>/gowebapp-mysql:v1
```

Build the  image locally. Make sure to include "." at the end. Make sure the
build runs to completion without errors. You should get a success message.
Run and test Docker images locally



### 2.5 Test application by running with Docker Engine.

Before putting our app in production let's run the Docker images locally, to ensure that the frontend and backend containers run and integrate properly.


**Step 1** Create Docker user-defined network

To facilitate cross-container communication, let's first define a user-defined
network in which to run the frontend and backend containers:

```
docker network create gowebapp \
-d bridge \
--subnet 172.19.0.0/16
```


!!! note
    Default bridge only allows connecting container by IP addresses which is not viable solution as IP address of docker change at startup.
    There for we creating a user defined bridge network `gowebapp`

**Step 2**  Launch `backend` container

Next, let's launch a frontend and backend container using the Docker CLI.
First, we launch the database container, as it will take a bit longer to
startup, and the frontend container depends on it. Notice how we are injecting
the database password into the MySQL configuration as an environment variable:

```
#TODO Launch `backend` container in background
#TODO Container needs to run on network: `gowebapp`
#TODO Use this settings: `--name gowebapp-mysql` `--hostname gowebapp-mysql` 
#TODO Include following Env Variable in the command: `MYSQL_ROOT_PASSWORD=rootpasswd`
```

**Step 3**  Launch `frontend` container

Now launch a frontend container, mapping the container port 80 - where the web application is exposed - to port 8080 on the host machine:

```
#TODO Launch `frontend` container in background
#TODO Container needs to run on network: `gowebapp`
#TODO Use this settings: `--name gowebapp` `--hostname gowebapp` 
#TODO Map the container port 80  - to port 8080 on the host machine
```

**Step 4**  Test the application locally

Now that we've launched the application containers, let's try to test the web
application locally.

You should be able to access the application at Google Cloud `Web Preview` Console:

![alt text](images/web_preview.png "Web Preview")

!!! note
    Web Preview using port `8080` by default. If you application using other port, you can edit this as needed.


Once you can see the application loaded. Create an account and login.
Write something on your Notepad and save it. This will verify that the application is working and properly integrates with
the backend database container.



!!! task
    Take a screenshot of running application. 

**Step 5** Inspect the MySQL database

Let's connect to the backend MySQL database container and run some queries to
ensure that application persistence is working properly:

```
#TODO docker xxx
```

**Step 6** Once inside the container, connect to MySQL database:

```
mysql -u root -p
password:
```

!!! note
    Use password that has beed used in `MYSQL_ROOT_PASSWORD` env variable.

**Step 7** Once connected, run some simple SQL commands to inspect the database tables
and persistence:
```
#Simple SQL to navigate
SHOW DATABASES;
USE gowebapp;
SHOW TABLES;
SELECT * FROM <table_name>;
exit;
```


### 2.6 Cleanup running applications

```
### TODO docker xxx
```


## 3 Submit Assignment to Instructor
 
### 3.1 Commit `DOCKERFILEs` and `README.md` to repository and share it with Instructor/Teacher


**Step 1** Edit Readme.md files with command you've created a working `gowebapp` application

```
edit  ~/$student_name-notepad/Mod5_assignment/Readme.md
```

**Step 2** Commit gowebapp and gowebapp-mysql folders and Readme.md using the following Git commands:


```
cd  ~/$student_name-notepad/
```

```
git add .
git commit -m "adding DOCKREFILEs and Readme"

```

**Step 2** Push commit to the Cloud Source Repositories:

```
git push origin master
```


**Step 3** Submit link to your Cloud Source Repository to LMS, replace with you values

```
https://source.cloud.google.com/${PROJECT_ID}/$MY_REPO
```

e.g: https://source.cloud.google.com/ycit019-project/ayratk-notepad



