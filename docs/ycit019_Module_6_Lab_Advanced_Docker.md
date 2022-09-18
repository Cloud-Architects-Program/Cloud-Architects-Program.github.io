Lab 4 Managing Docker Images

**Objective:**

* Docker Storage
* Learn to build docker images using Dockerfiles.
* Store images in Docker Hub
* Learn alternative registry solutions (GCR)

## Prepare Lab Environment

This lab can be executed in you GCP Cloud Environment using Google Cloud Shell.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")

Once opened, you can use it to run the instructions for this lab.


## 1 Persistant Volumes

### 1.1 Storage driver
We've discussed several Storage drivers (graphdrivers) during the class.
Let's find out what graphdriver is running in our Lab environment.

```
docker info | grep  Storage
```
```
Storage Driver: overlay2
```

!!! result
    Our Classroom is running `auoverlay2fs` storage driver.

!!! summary
    Systems runnng Ubuntu or Debian ,going to run `overlay2` storage driver by
    default. Prior to that it was  `aufs` storage driver

### 1.2 Persisting Data Using Volumes

Docker Volumes are created and assigned when containers are started. Data
Volumes allow you to map a host directory to a container for sharing data.

This mapping is bi-directional. It allows data stored on the host to be accessed
from within the container. It also means data saved by the process inside the
container is persisted on the host.

#### 1.2.1  Create and manage volumes

**Step 1** Create a volume:
```
docker volume create --name my-vol
```
**Step 2** List volumes:
```
docker volume ls
```
```
Output:
local               my-vol
```
**Step 3** Inspect a volume:

```
docker volume inspect my-vol
```
```
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

**Step 3** Add some data to the `Mountpoint` of the volume:
```
sudo touch  /var/lib/docker/volumes/my-vol/_data/test_vol
sudo ls  /var/lib/docker/volumes/my-vol/_data/
```

**Step 4** Create a container `busybox` alpine image and attach created `my-vol`
volume in to it:

```
docker run -it -v my-vol:/world busybox
```
```
/ # ls /world
test_vol
/ #
```

!!! result
    Volume is mounted and `test_vol` file is under `/world` folder as expected

**Step 5**  Try to delete the volume:

```
docker volume rm  my-vol
```
```
Error response from daemon: unable to remove volume: remove my-vol: volume is in use - [6ef3055b516b306847150af8fcea796c02cd90578967802ac29c39d3a2c90102]
```

!!! failure
    Deleting container that is attached is not permited. However you can
    delete with `-f` option

**Step 5**  Busybox container stopped, howerver it is not deleted.
Let's locate stopped `busybox` container and delete it:

```
docker ps -a | grep busybox
```

```
docker rm $docker_id
```

**Step 6** You can now delete `my-vol`
!!! note
    Volume is still avaiable if needed to be reattached any time

```
docker volume ls
docker volume rm my-vol
docker volume ls
```

!!! summary
    Volumes can be craeted and managed separately from containers.


#### 1.2.2 Start a container with a volume
If you start a container with a volume that does not yet exist, Docker creates
the volume for you.

**Step 1** Add a data volume to a container:
```
docker run -d -P --name webapp -v /webapp training/webapp python app.py
```

!!! result
    Command started a new container and created a new volume inside the
    container at /webapp.

**Step 2** Locate the volume on the host using the docker inspect command:

```
 docker inspect webapp | grep -A9 Mounts
 ```
 **Output:**
 ```       "Mounts": [
            {
                "Type": "volume",
                "Name": "39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d",
                "Source": "/var/lib/docker/volumes/39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d/_data",
                "Destination": "/webapp",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""

```

**Step 3** List container
```
docker volume ls
```
**Output:**
```
DRIVER              VOLUME NAME
local               39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d
```


**Step 5** Alternatively, you can specify a host directory you want to use as a data
volume:
```
mkdir db

docker run -d --name db -v ~/db:/db training/postgres
```

**Step 2** Start an interactive session in the db container and create a new
file in the /db directory:
```
docker exec -it db bash
```

Type inside docker containers console:
```

root@9a7a4fbcc929:/# cd /db

root@9a7a4fbcc929:/db# touch hello_from_db_container

root@9a7a4fbcc929:/db# exit
```

**Step 4** Check that the local db directory contains the new file:
```
ls db
hello_from_db_container
```

**Step 5** Check that the data volume is persistent. Remove the db container:
```
docker rm -f db
```

**Step 6** Create the db container again:
```
docker run -d --name db -v ~/db:/db training/postgres
```

**Step 7** Check that its /db directory contains the hello_from_db_container
file:

```
docker exec -it db bash


```
Run commands inside container:
```
root@47a60c01590e:/# ls /db
hello_from_db_container
root@47a60c01590e:/# exit
```


# 2 Distributing Docker images with Container Registry

In the previous modules, we learned how to use Docker images
to run Docker containers. Docker images that we used have been downloaded from
the Docker Hub, a Docker image registry maintained by Docker Inc. In this section we will create a
simple web application from scratch. We will use Flask
([http://flask.pocoo.org/](http://flask.pocoo.org/)), a microframework for
Python. Our application for each request will display a random picture from the
defined set.

In the next session we will create all necessary files for our application,
build docker image and then push to Docker Hub and Quay.

The code for this application is also available in GitHub:
```
https://github.com/Cloud-Architects-Program/ycit019_2022/tree/main/Module5/flask-app
```
## 1.1 Create DOCKERFILE

**Step 1** Clone git repo on you laptop:

```
git clone https://github.com/Cloud-Architects-Program/ycit019_2022
cd ~/ycit019_2022/Module5/flask-app/
git pull
```

**Step 2** In this directory, we see following files:
```
flask-app/
    Dockerfile
    app.py
    requirements.txt
    templates/
        index.html
```
**Step 3** Let’s review file app.py with the following content:

```python
from flask import Flask, render_template
import random

app = Flask(__name__)

# list of cat images
images = [
    "https://media.giphy.com/media/mlvseq9yvZhba/giphy.gif",
    "https://media.giphy.com/media/13CoXDiaCcCoyk/giphy.gif",
    "https://media.giphy.com/media/LtVXu5s7KwlK8/giphy.gif",
    "https://media.giphy.com/media/PekRU0CYIpXS8/giphy.gif",
    "https://media.giphy.com/media/11quO2C07Sh2oM/giphy.gif",
    "https://media.giphy.com/media/12HZukMBlutpoQ/giphy.gif",
    "https://media.giphy.com/media/1HKaikaFqDt7i/giphy.gif",
    "https://media.giphy.com/media/v6aOjy0Qo1fIA/giphy.gif",
    "https://media.giphy.com/media/12bjQ7uASAaCKk/giphy.gif",
    "https://media.giphy.com/media/HFcl9uhuCqzGU/giphy.gif"
]

@app.route('/')
def index():
    url = random.choice(images)
    return render_template('index.html', url=url)

if __name__ == "__main__":
    app.run(host="0.0.0.0")
```
**Step 4** Below is the content of requirements.txt file:
```
Flask==2.0.0
```
**Step 5** Under directory templates observe index.html with the following
content:

```html
<html>
  <head>
    <style type="text/css">
      body {
        background: black;
        color: white;
      }
      div.container {
        max-width: 500px;
        margin: 100px auto;
        border: 20px solid white;
        padding: 10px;
        text-align: center;
      }
      h4 {
        text-transform: uppercase;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h4>Cat Gif of the day</h4>
      <img src="{{url}}" />
    </div>
  </body>
</html>
```
**Step 6** Let’s review content of the Dockerfile:

```docker
# Official Python Alpine Base image using Simple Tags
# Image contains Python 3 and pip pre-installed, so no need to install them

FROM python:3.9.5-alpine3.12

# Specify Working directory
WORKDIR /usr/src/app

# COPY requirements.txt /usr/src/app/
COPY requirements.txt ./

# Install Python Flask used by the Python app
RUN pip install --no-cache-dir -r requirements.txt

# Copy files required for the app to run
COPY app.py ./
COPY templates/index.html ./templates/

# Make a record that the port number the container should be expose is:
EXPOSE 5000

# run the application
CMD ["python", "./app.py"]
```

## 1.2 Build a Docker image

**Step 1** Now let’s build our Docker image. In the command below, replace
<Docker-hub-user-name> with your user name. This user name should be the same as you
created when you registered on Docker Hub. Because we will publish our build
image in the next step to your own Docker Hub.

```
docker build -t <Docker-hub-user-name>/myfirstapp .
```

!!! result
    Image has been buit 

**Step 2** Where is your built image? It’s in your machine’s local Docker image
registry, you can check that your image exists with command below:

```

docker images

```

**Step 3** Now run a container in a background and expose a standard HTTP port
(80), which is redirected to the container’s port 5000:
```
docker run -dp 8080:5000 --name myfirstapp <Docker-hub-user-name>/myfirstapp
```

**Step 4** Now that we've launched the application containers, let's try to test the web
application locally.

You should be able to access the application at Google Cloud `Web Preview` Console:

![alt text](images/web_preview.png "Web Preview")

!!! note
    Web Preview using port `8080` by default. If you application using other port, you can edit this as needed.

**Step 5** Stop the container and remove it:

```
docker rm -f myfirstapp
```

### 1.2.2 Publish Docker Image to Docker Hub

One of the most popular way to share and work with you images is to push
them to the Docker Hub.

**Docker Hub** is a [registry](https://hub.docker.com/explore/)
of Docker images. You can think of the registry as a directory of all available
Docker images.

**Step 1 (Optional)** If you don’t have a Docker account, sign up for one
[here](https://hub.docker.com/). Make a note of your username and password.

**Step 2** Log in to your local machine.

```
docker login
```

**Step 3** Now, publish your image to docker Hub.
```
docker push <Docker-hub-user-name>/myfirstapp
```

**Step 4** Login to [https://hub.docker.com](https://hub.docker.com) and verify
simage and tags.

!!! result
    Image been pushed and can be observed in Docker Hub, with the **tag** latest.


**Step 5** It is also possible to specify a custom tag for image prior to push
it to the registry

Note Image Tag of the created `myfirstapp`:

```
docker images
```

Modify `$docker_image_tag` with `myfirstapp:v1` image tag value:

```
docker tag $docker_image_tag <Docker-hub-user-name>/myfirstapp:v1
docker push <Docker-hub-user-name>/myfirstapp:v1
```

!!! result
    Image been pushed and can be observed in Docker Hub. You can now
    observe 2 docker image one with the **tag** latest and another with tag v1

**Step 6** You can now **pull** or **run** specified **Docker images** from any
other location where docker engine is installed with following commands:

```
docker pull <Docker-hub-user-name>/myfirstapp:latest

docker pull <Docker-hub-user-name>/myfirstapp:v1
```

!!! result
    Images stored locally


```

docker images


```
**Output:**
```
myfirstapp       v1       f50f9524513f    1 hour ago   22 MB

myfirstapp       latest   f50f9524513f    1 hour ago   22 MB
```


Finally run images with specific tag:

```
docker run <Docker-hub-user-name>/myfirstapp:v1
```

### 1.2.3 Pushing images to gcr.io

In a similar manner we need to tag the image to prepare it to be pushed to gcr.io. We just need to change the registry, which is for gcr.io formatted as gcr.io/PROJECT_ID.

**Step 1** Get the Project ID:

```
PROJECT_ID=$(gcloud config get-value project)
```

**Step 2** Enable the required APIs:

```
gcloud services enable containerregistry.googleapis.com
```

**Step 3** Tag the image:

Modify `$docker_image_tag` with `myfirstapp:v1` image tag value:


```
docker tag $docker_image_tag gcr.io/${PROJECT_ID}/myfirstapp:v1
```


Push the image to gcr.io:

```
docker push gcr.io/${PROJECT_ID}/myfirstapp:v1
```

**Step 4**  Login to GCP console -> Container Registry -> Images

!!! result
    Docker images has been pushed to GCR registry


### 1.2.3 Pushing images to Local Repository

First, we need to spin up a local docker registry. This could be a use case if you want to deploy basic registry On-Prem.
This registry will luck security features such as Authentication, SSL, scanning.
If you interested to use Enterprise ready solution On-Prem consider: Jfrog Artifactory, RedHa's Clair, Docker Enterprise or
open source CNCF project Harbor.

**Step 1** Deploy local registry

```
docker run -d -p 5000:5000 --name registry registry:2.7.1
```

**Step 2** In order to upload an image to a registry, we need to tag it properly

Modify `$docker_image_tag` with `myfirstapp:v1` image tag value:



```
docker tag $docker_image_tag localhost:5000/myfirstapp:v1
```


**Step 3** Now that we have an image tagged correctly, we can push it to our local registry

```
docker push localhost:5000/myfirstapp:v1
```

**Step 4** Let’s now delete the local image, and pull it again from the local registry

To delete the image, we need to first remove the container that depends on that image.
Run `docker ps` and get the Container_ID for the container that uses  myfirstapp:v1
Kill and delete that container by running the following command, but make sure to replace CONTAINER_ID, with the actual ID.

```
docker rm CONTAINER_ID
```

Result: The command will print back the container ID, which is an indication it was successful.

**Step 5** Run docker images to validate

```
docker images
```

**Step 6** Now we can delete the docker image

```
docker rmi localhost:5000/myfirstapp:v1
```

**Step 7** Although the image is deleted locally, it is still in the registry and we can pull it back, or use it to deploy containers.

```
docker run -dp 8080:5000 --name myfirstapp localhost:5000/myfirstapp:v1
```

Run `docker images` again to check how the image is available locally again.

```
docker images
```

**Step 8 Cleanup:**

```
docker rm -f myfirstapp
```


# 3 Follow Docker Best Practices

## 3.1 Inspecting Dockerfiles with `dockle`

`Dockle` - Container Image Linter for Security, Helping build the Best-Practice Docker Image, Easy to start


`Dockle` helps you:

* Build Best Practice Docker images
* Build secure Docker images
  * Checkpoints includes CIS Benchmarks


**Step 1**  Install Dockle

```
$ VERSION=$(
 curl --silent "https://api.github.com/repos/goodwithtech/dockle/releases/latest" | \
 grep '"tag_name":' | \
 sed -E 's/.*"v([^"]+)".*/\1/' \
) && curl -L -o dockle.deb https://github.com/goodwithtech/dockle/releases/download/v${VERSION}/dockle_${VERSION}_Linux-64bit.deb
$ sudo dpkg -i dockle.deb && rm dockle.deb
```


**Step 2** Experiment with existing applications we've created in the class:

```
$ dockle [YOUR_IMAGE_NAME]
```


e.g.

```
dockle archy/myfirstapp
```

output:

```
WARN	- CIS-DI-0001: Create a user for the container
	* Last user should not be root
WARN	- DKL-DI-0006: Avoid latest tag
	* Avoid 'latest' tag
INFO	- CIS-DI-0005: Enable Content trust for Docker
	* export DOCKER_CONTENT_TRUST=1 before docker pull/build
INFO	- CIS-DI-0006: Add HEALTHCHECK instruction to the container image
	* not found HEALTHCHECK statement
INFO	- DKL-LI-0003: Only put necessary files
	* Suspicious directory : tmp
```




## 3.2 Scan images with [Trivy](https://aquasecurity.github.io/trivy/v0.18.0/)

Trivy (tri pronounced like trigger, vy pronounced like envy) is a simple and comprehensive vulnerability scanner for containers and other artifacts. A software vulnerability is a glitch, flaw, or weakness present in the software or in an Operating System. Trivy detects vulnerabilities of OS packages (Alpine, RHEL, CentOS, etc.) and application dependencies (Bundler, Composer, npm, yarn, etc.). Trivy is easy to use. Just install the binary and you're ready to scan. All you need to do for scanning is to specify a target such as an image name of the container.


**Step 1**  Install Trivy

```
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```



**Step 2** Specify an image name (and a tag).

```
$ trivy image [YOUR_IMAGE_NAME]
```

For example:

```
$ trivy image python:3.4-alpine
```

```
2019-05-16T01:20:43.180+0900    INFO    Updating vulnerability database...
2019-05-16T01:20:53.029+0900    INFO    Detecting Alpine vulnerabilities...

python:3.4-alpine3.9 (alpine 3.9.2)
===================================
Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 1, HIGH: 0, CRITICAL: 0)

+---------+------------------+----------+-------------------+---------------+--------------------------------+
| LIBRARY | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |             TITLE              |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
| openssl | CVE-2019-1543    | MEDIUM   | 1.1.1a-r1         | 1.1.1b-r1     | openssl: ChaCha20-Poly1305     |
|         |                  |          |                   |               | with long nonces               |
+---------+------------------+----------+-------------------+---------------+--------------------------------+
```

**Step 3**  Explore local images in your environment. 



# 4 Docker Compose

In this module, will guide you through the process of building a multi-container
application using docker compose. The application code is available at GitHub: [https://github.com/Cloud-Architects-Program/ycit019_2022](https://github.com/Cloud-Architects-Program/ycit019_2022)


## 4.1 Deploy Guestbook app with Compose

Let’s build another application. This time we going to create famous Guestbook
application.

Guestbook consists of three services. A redis-master node, a set of redis-slave
that can be scaled and find the redis-master via its DNS name. And a PHP frontend
that exposes itself on port 80. The resulting application allows you to leave
short messages which are stored in the redis cluster.


**Step 1** Change directory to the guestbook

```
cd ~/ycit019_2022/Module6/guestbook/
ls
```

**Step 2** Let’s review the docker-guestbook.yml file

```
version: "2"

services:
 redis-master:
   image: gcr.io/google_containers/redis:e2e
   ports:
     - "6379"
 redis-slave:
   image: gcr.io/google_samples/gb-redisslave:v1
   ports:
     - "6379"
   environment:
     - GET_HOSTS_FROM=dns
 frontend:
   image: gcr.io/google-samples/gb-frontend:v4
   ports:
     - "80:80"
   environment:
     - GET_HOSTS_FROM=dns
```

**Step 3** Let’s run docker-guestbook.yml with compose

```
export LD_LIBRARY_PATH=/usr/local/lib
docker-compose -f docker-guestbook.yml up -d
```
```

Creating network "examples_default" with the default driver
Creating examples_redis-slave_1
Creating examples_frontend_1
Creating examples_redis-master_1
```

!!! note
    `-d` -  Detached mode: Run containers in the background, print new container names.

    `-f` -  Specify an alternate compose file (default: docker-compose.yml)

**Step 4** Check that all containers are running:

```
docker ps
```
```
CONTAINER ID        IMAGE                                    COMMAND
d1006d1beee5        gcr.io/google-samples/gb-frontend:v4     "apache2-foreground"
fb3a15fde23f        gcr.io/google_containers/redis:e2e       "redis-server /etc..."
326b94d4cdd7        gcr.io/google_samples/gb-redisslave:v1   "/entrypoint.sh /b..."
```

**Step 5**  Test the application locally

Now that we've launched the application containers, let's try to test the web
application locally.

You should be able to access the application at Google Cloud `Web Preview` Console:

![alt text](images/web_preview.png "Web Preview")

!!! note
    Web Preview using port `8080` by default. If you application using other port, you can edit this as needed.

!!! success
    Nice you now have compose stuck up and running!


**Step 6** Cleanup environment:

```
docker-compose -f docker-guestbook.yml down
```
```
Stopping guestbook_frontend_1 ... done
Stopping guestbook_redis-master_1 ... done
Stopping guestbook_redis-slave_1 ... done
Removing guestbook_frontend_1 ... done
Removing guestbook_redis-master_1 ... done
Removing guestbook_redis-slave_1 ... done
Removing network guestbook_default
```


!!! Congratulations
    You are now docker expert!
    We were able to start microservices application with docker compose.


!!! Summary
    We've learned how to use docker-compose v2.

    In the assignement you will be using docker-compose v3

    Read the Docker-Compose [documentation](https://docs.docker.com/compose/compose-file/) on new syntax.