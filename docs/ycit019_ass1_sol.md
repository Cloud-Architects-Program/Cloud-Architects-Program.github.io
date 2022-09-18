# 1 Containerize Applications

**Objective:**

  * Review process of containerizing of applications
  * Review creation of Docker Images
  * Review build image process


## Prepare Lab Environment

This lab can be executed in you GCP Cloud Environment using Google Cloud Shell.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")


Once opened, you can use it to run the instructions for this lab.



## 1.1 Overview of the Sample Application

This package contains two application components which will be used
throughout the course to demonstrate features of Kubernetes:

* gowebapp

This directory contains a simple Go-based note-taking web application. It includes a home page, registration page, login page, about page, and note management page.

Configuration for this application is stored in code/config/config.json. This file is used to configure the backing data store for the application, which in this course is MySQL.

Later in the course, we will externalize this configuration in order to take advantage of Kubernetes Secrets and ConfigMaps. This will include some minor modifications to the Go source code. Go programming experience is not required to complete the exercises.

For more details about the internal design and implementation of the Go web application, see code/README.md.

* gowebapp-mysql

This directory contains the schema file used to setup the backing MySQL database for the Go web application.

## 1.1 Build Dockers image for frontend application


!!! result
    Two folders with go app and mysql config has been reviewed.

**Step 2** Write Dockerfile for your frontend application

Create a file named `Dockerfile` in this directory for the frontend Go application. Use vi or any preferred text editor. The template below provides a starting point for defining the contents of this file. Replace TODO comments with the appropriate commands:

```
cd ~/ycit019_2022/Mod5_assignment/gowebapp
```

```
vim Dockerfile
```

```
FROM golang:1.15.11

LABEL maintainer "student@mcgill.ca"
LABEL gowebapp "v1"

EXPOSE 80

ENV GOPATH=/go

COPY /code $GOPATH/src/gowebapp/

WORKDIR $GOPATH/src/gowebapp/

RUN go get && go install

ENTRYPOINT $GOPATH/bin/gowebapp
```

**Step 3**  Build gowebapp Docker image locally

Build the  image locally. Make sure to include "." at the end. Make sure the
build runs to completion without errors. You should get a success message.

```
docker build -t <user-name>/gowebapp:v1 .
```

## 1.2 Build Docker image for backend application

**Step 1** Locate folder with mysql config

```
cd ~/ycit019_2022/Mod5_assignment/gowebapp-mysql
```

**Step 2** Write Dockerfile for your backend application

Create a file named `Dockerfile` in this directory for the backend MySQL database application. Use vi or any preferred text editor. The template below provides a starting point for defining the contents of this file. Replace TODO comments with the appropriate commands:

```
FROM mysql:8.0

LABEL maintainer "student@mcgill.ca"
LABEL gowebapp-sql "v1"

COPY gowebapp.sql /docker-entrypoint-initdb.d/
```

**Step 2** Build gowebapp-mysql Docker image locally

```
docker build -t <user-name>/gowebapp-mysql:v1 .
```

Build the image locally. Make sure to include "." at the end. Make sure the
build runs to completion without errors. You should get a success message.
Run and test Docker images locally


## 1.4 Test application by running with Docker Engine.

Before putting our app in production let's run the Docker images locally, to ensure that the frontend and backend containers run and integrate properly.

**Step 1** Create Docker user-defined network

To facilitate cross-container communication, let's first define a user-defined
network in which to run the frontend and backend containers:


```
docker network create gowebapp \
-d bridge \
--subnet 172.19.0.0/16
```

**Step 2**  Launch backend container

Next, let's launch a frontend and backend container using the Docker CLI.
First, we launch the database container, as it will take a bit longer to
startup, and the frontend container depends on it. Notice how we are injecting
the database password into the MySQL configuration as an environment variable:

```
docker run --net gowebapp --name gowebapp-mysql --hostname gowebapp-mysql \
-d -e MYSQL_ROOT_PASSWORD=rootpasswd <user-name>/gowebapp-mysql:v1
```

docker run --net gowebapp --name gowebapp-mysql --hostname gowebapp-mysql \
-d -e MYSQL_ROOT_PASSWORD=rootpasswd archyufa/gowebapp-mysql:v1


**Step 3**  Launch frontend container

Now launch a frontend container, mapping the container port 80 - where the web application is exposed - to port 8080
 on the host machine:

```
docker run -p 8080:80 --net gowebapp -d --name gowebapp \
--hostname gowebapp <user-name>/gowebapp:v1
```

docker run -p 8080:80 -d --net gowebapp --name gowebapp \
--hostname gowebapp archyufa/gowebapp:v1

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
docker exec -it gowebapp-mysql  bash
```

**Step 6** Once inside the container, connect to MySQL database:

```
mysql -u root -p
password:
```

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

## 1.5 Cleanup running applications and unused networks

```
docker rm -f $(docker ps -q)
```
