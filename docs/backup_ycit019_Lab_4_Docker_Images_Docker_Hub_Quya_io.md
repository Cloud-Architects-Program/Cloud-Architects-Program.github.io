Lab 4 Managing Docker Images

**Objective:**


* Learn to build docker images using Dockerfiles.
* Store images in Docker Hub
* Learn alternative registry solutions (Quya.io)
* Automate image build process with Docker Cloud

# 1 Building Docker Images

In the previous modules, we learned how to use Docker images
to run Docker containers. Docker images that we used have been downloaded from
the Docker Hub, a registry of Docker images. In this section we will create a
simple web application from scratch. We will use Flask
([http://flask.pocoo.org/](http://flask.pocoo.org/)), a microframework for
Python. Our application for each request will display a random picture from the
defined set.

In the next session we will create all necessary files for our application,
build docker image and then push to Docker Hub and Quay.

The code for this application is also available in GitHub:
```
git clone https://github.com/archyufa/k8scanada
```
## 1.1 Create DOCKERFILE

**Step 1** Clone git repo on you laptop:

```
git clone https://github.com/archyufa/k8scanada
cd k8scanada/Module4/flask-app/
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

```pythonfrom flask import Flask, render_templateimport randomapp = Flask(__name__)# list of cat imagesimages = [    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr05/15/9/anigif_enhanced-buzz-26388-1381844103-11.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr01/15/9/anigif_enhanced-buzz-31540-1381844535-8.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr05/15/9/anigif_enhanced-buzz-26390-1381844163-18.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr06/15/10/anigif_enhanced-buzz-1376-1381846217-0.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr03/15/9/anigif_enhanced-buzz-3391-1381844336-26.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr06/15/10/anigif_enhanced-buzz-29111-1381845968-0.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr03/15/9/anigif_enhanced-buzz-3409-1381844582-13.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr02/15/9/anigif_enhanced-buzz-19667-1381844937-10.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr05/15/9/anigif_enhanced-buzz-26358-1381845043-13.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr06/15/9/anigif_enhanced-buzz-18774-1381844645-6.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr06/15/9/anigif_enhanced-buzz-25158-1381844793-0.gif",    "http://ak-hdl.buzzfed.com/static/2013-10/enhanced/webdr03/15/10/anigif_enhanced-buzz-11980-1381846269-1.gif"]@app.route('/')def index():    url = random.choice(images)    return render_template('index.html', url=url)if __name__ == "__main__":    app.run(host="0.0.0.0")```**Step 4** Below is the content of requirements.txt file:
```
Flask==0.10.1
```
**Step 5** Under directory templates observe index.html with the following
content:

```html<html>  <head>    <style type="text/css">      body {        background: black;        color: white;      }      div.container {        max-width: 500px;        margin: 100px auto;        border: 20px solid white;        padding: 10px;        text-align: center;      }      h4 {        text-transform: uppercase;      }    </style>  </head>  <body>    <div class="container">      <h4>Cat Gif of the day</h4>      <img src="{{url}}" />    </div>  </body></html>```
**Step 6** Let’s review content of the Dockerfile:

```docker
# our base image
FROM alpine:3.5

# Install python and pip
RUN apk add --update py2-pip

# upgrade pip
RUN pip install --upgrade pip

# install Python modules needed by the Python app
COPY requirements.txt /usr/src/app/
RUN pip install --no-cache-dir -r /usr/src/app/requirements.txt

# copy files required for the app to run
COPY app.py /usr/src/app/
COPY templates/index.html /usr/src/app/templates/

# tell the port number the container should expose
EXPOSE 5000

# run the application
CMD ["python", "/usr/src/app/app.py"]
```
## 1.2 Build a Docker image

**Step 1** Now let’s build our Docker image. In the command below, replace
<user-name> with your user name. This user name should be the same as you
created when you registered on Docker Hub. Because we will publish our build
image in the next step.
```
docker build -t <user-name>/myfirstapp .
```
**Step 2** Where is your built image? It’s in your machine’s local Docker imageregistry, you cancheck that your image exists with command below:

```
docker images
```

**Step 3** Now run a container in a background and expose a standard HTTP port
(80), which is redirected to the container’s port 5000:
```
docker run -dp 80:5000 --name myfirstapp <user-name>/myfirstapp
```

**Step 4** Use your browser to open the address http://<VM or Laptop IP> and
check that the application works.

**Step 5** Stop the container and remove it:
```
docker stop myfirstapp

docker rm myfirstappmyfirstapp
```
### 1.2.1 Share docker images with tar files

Now ideally you want to share you freshly build docker image with someone or
run it in different environment which for some reason don’t have internet access
so images can not be pulled from Online Docker registries. In that case Docker
images can be shared as we share traditionally regular files by creating
**tarballs**  using docker save  command as following

**Step 1** Create a tar file using docker save  command:

```
docker save <user-name>/myfirstapp > myfirstapp.tar
```

Or
```
docker save --output myfirstapp1.tar archyufa/myfirstapp
ls -trh | grep tar
```

**Step 2** Transfer images to another environment using scp command.!!! Hint
    You can also store this images in Object storages, e.g. Swift or
    Amazon S3 using version control.

**Step 3** Now you can restore this images using docker load, that will load a
tarred repository from a file or the standard input stream. It restores both
images and tags.
```
docker load < myfirstapp.tar
```

```
23b9c7b43573: Loading layer [==================================================>]   4.23MB/4.23MB

b3b5c1214f71: Loading layer [==================================================>]  52.87MB/52.87MB

f877d8dd64d3: Loading layer [==================================================>]  8.636MB/8.636MB

bba871f91589: Loading layer [==================================================>]  3.584kB/3.584kB

1c131e92eb5f: Loading layer [==================================================>]  5.053MB/5.053MB

3f6463bcb64c: Loading layer [==================================================>]   5.12kB/5.12kB

47c61110467a: Loading layer [==================================================>]  4.096kB/4.096kB

Loaded image: archyufa/myfirstapp:latest

```

```
docker images
```

### 1.2.2 Publish Docker Image to Docker Hub

However the most popular way to share and work with you images is to push
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
docker push <user-name>/myfirstapp
```

**Step 4** Login to [https://hub.docker.com](https://hub.docker.com) and verify
simage and tags.

!!! result
    Image been pushed and can be observed in Docker Hub, with the **tag** latest.

**Step 5** It is also possible to specify a custom tag for image prior to push
it to the registry

```
docker tag 5b45ce063cea <user-name>/myfirstapp:v1
docker push <user-name>/myfirstapp:v1
```

!!! result
    Image been pushed and can be observed in Docker Hub. You can now
    observe 2 docker image one with the **tag** latest and another with tag v1

**Step 6** You can now **pull** or **run** specified **Docker images** from any
other location where docker engine is installed with following commands:
```
docker pull <user-name>/myfirstapp:latest

docker pull <user-name>/myfirstapp:v1
```
!!! result    Images stored locally

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
docker run <user-name>/myfirstapp:v1
```


### 1.2.3 Automated Builds with Docker Cloud

**Live Demo:**

* Docker Security scanning
* Setting Up Auto-Build in Docker Cloud and notifications to slac
* Automated Tests with PR

### 1.2.4 Push Docker Images to quay.io

**Prerequisite:** Register to  quay.io with your Github user

**Step 1** Login to quay.io from CLI
```
docker login quay.io
```

**Step 2** Build image with quay prefix
```
docker build -t quay.io/archyufa/myfirstapp .
```

**Step 3** Push image to quay registry
```
docker push quay.io/archyufa/myfirstapp
```

### 1.2.5 Demo quay.io

**Live Demo:**

  * Docker Security scanning
  * Setting Up Auto-Build in Docker Cloud / Quai.io
