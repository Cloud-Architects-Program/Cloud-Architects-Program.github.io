Lab 4 Managing Docker Images

**Objective:**


* Learn to build docker images using Dockerfiles.
* Store images in Docker Hub
* Learn alternative registry solutions (GCR)

## Prepare Lab Environment

This lab can be executed in you GCP Cloud Environment using Google Cloud Shell.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")

Once opened, you can use it to run the instructions for this lab.



# 1 Distributing Docker images with Container Registry

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
https://github.com/Cloud-Architects-Program/ycit019/tree/main/Module5/flask-app
```
## 1.1 Create DOCKERFILE

**Step 1** Clone git repo on you laptop:

```
git clone https://github.com/Cloud-Architects-Program/ycit019
cd ~/ycit019/Module5/flask-app/
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
$docker_image_tag
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


# 2 Follow Docker Best Practices

## 2.1 Inspecting Dockerfiles with `dockle`

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



## 2.2 Automated Builds with Google Cloud Build

**Live Demo:**

* GCR Image scanning
* Setting Up Docker Image Auto-Build with Google Cloud Build based on Push to Branch
* Auto Deployment of Image to Cloud Run
