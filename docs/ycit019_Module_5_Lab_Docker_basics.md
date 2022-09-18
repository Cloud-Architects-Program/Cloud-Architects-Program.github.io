# Module 5 Lab Docker basics

**Objective:**

  * Practice to run Docker containers
  * Learn to build docker images using Dockerfiles
  * Learn Docker Networking



## 1 Setup Environment and Install Docker

### 1.1 Create an instance with gcloud

Rather than using the Google Cloud Console to create a virtual machine instance, you can use the command line tool `gcloud`, which is pre-installed in [Google Cloud Shell](https://cloud.google.com/developer-shell/#how_do_i_get_started). Cloud Shell is a Debian-based virtual machine loaded with all the development tools you’ll need (gcloud, git, and others) and offers a persistent 5GB home directory.


!!! note
    If you want to try this on your own machine in the future, read the [gcloud command line](https://cloud.google.com/sdk/gcloud/) tool guide.


Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")

Once opened, you can create a new virtual machine instance from the command line by using `gcloud` (feel free to use another zone closer to you):


**Step 1** Create a new virtual machine instance on GCE

```
gcloud config set project <set_you_project_id>
```

```
gcloud compute instances create docker-nginx --zone us-central1-c
```

**Output:**
```
Created [...docker-nginx].
NAME     ZONE           MACHINE_TYPE  PREEMPTIBLE INTERNAL_IP EXTERNAL_IP    STATUS
docker-nginx  us-central1-c  n1-standard-1             10.240.X.X  X.X.X.X        RUNNING
```


The instance created has these default values:
- The latest Debian 10 (buster) image.
- The n1-standard-1 machine type.
- A root persistent disk with the same name as the instance; the disk is automatically attached to the instance.


!!! note 
    You can set the default region and zones that `gcloud` uses if you are always working within one region/zone and you don’t want to append the --`zone` flag every time. Do this by running these commands:

    ```
    gcloud config set compute/zone ...
    gcloud config set compute/region ...
    ```



**Step 2** SSH into your instance using `gcloud` as well. 


```
gcloud compute ssh docker-nginx --zone us-central1-c
```

**Output:**

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: [/usr/bin/ssh-keygen] will be executed to generate a key.
This tool needs to create the directory 
[/home/gcpstaging306_student/.ssh] before being able to generate SSH 
Keys.
```

Now you’ll type `Y` to continue.

```
Do you want to continue? (Y/n)
```

**Enter** through the passphrase section to leave the passphrase empty. 

```
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase)
```

### 1.2 Install Docker Engine

We need to first set up the Docker repository. To set up the repository:

**Step 1** Update the `apt` package index:


```
sudo apt-get update
```

**Step 2** Then install packages to allow “apt” to use a repository over HTTPS:

```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

**Step 3**  Add Docker’s official GPG key:

```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

**Step 4** Set up the stable repository

```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

**Step 5** Install Docker Engine:

```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

**Step 6** Verify Docker Version has been deployed:

```
docker version
```

Verify what is the latest Docker [release](https://docs.docker.com/engine/release-notes/)

**Step 6** Confirm that your installation was successful by running Hello World! Container.

```
sudo docker run hello-world
```

**Step 7** If it is successful, you will see the following output:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete
Digest: sha256:f2266cbfc127c960fd30e76b7c792dc23b588c0db76233517e1891a4e357d519
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**Step 8**  Exit from `VM`

```
exit
```

**Step 9**  Delete `VM Instance` to avoid extra cost:

```
gcloud compute instances delete docker-nginx --zone us-central1-c
```


Now you’ll type `Y` to continue.

```
Do you want to continue? (Y/n)
```

!!! Summary
    Now you know how to deploy Docker on Linux VM. 

!!! Task
    Find out how to deploy Docker on Mac or Windows?

## 2 Working with Docker CLI

### 2.1 Show running containers

Going forward we going to run Docker commands directly from Google Cloud Console.
This is possible because Docker is installed on Cloud Console VM for you convenience.

Open the Google Cloud Shell by clicking on the icon on the top right of the screen:

![alt text](images/cloud_shell.png "Google Cloud Shell")

Once opened, you can use it to run the instructions for this lab.


**Step 1** Create a docker container `hello-world`:

```
docker run hello-world
```


**Step 2** Run docker ps to show running containers:

```
docker ps
```

!!! result
    The output shows that there are no running containers at the moment.

**Step 2** Use the command docker `ps -a` to list all containers including the ones has
been stopped:

```
docker ps -a
```
**Output:**
```

CONTAINER ID IMAGE       COMMAND  CREATED        STATUS             PORTS  NAMES
6e6db2a24a8e hello-world "/hello" 15 minutes ago Exited (0) 15 min  dreamy_nobel
```

Review the collumns `CONTAINER ID`, `STATUS`, `COMMAND`, `PORTS`, `NAMES`.

In the previous section we started one container and the command docker `ps -a`
shows it as `Exited`.

!!! note
    You can name your own containers with --name when you use docker run. If
    you do not provide a name, Docker will generate a random one like the one
    you have.

!!! question
    Why Docker names are random? How docker containers named?

**Step 3** Let’s run the command `docker images` to show all the images on your
local system:

```
docker images
```

As you see, there is only one image that was downloaded from the Docker Hub.

### 2.3 Specify a container main process

**Step 1** Let’s run our own "hello world" container. For that we will use the
official [Ubuntu image](https://hub.docker.com/_/ubuntu/):

```
docker run ubuntu /bin/echo 'Hello world'
```
**Output:**
```
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
...
Status: Downloaded newer image for ubuntu:latest
Hello world
```
As you see, Docker downloaded the image ubuntu because it was not on the local
machine.

**Step 2** Let’s run the command `docker images` again:

```
docker images
```
**Output:**
```

REPOSITORY          TAG                 IMAGE ID            CREATED        SIZE
ubuntu              latest              42118e3df429        11 days ago  124.8 MB
hello-world         latest              c54a2cc56cbb        4 weeks ago  1.848 kB
```

**Step 3** If you run the same "hello world" container again, Docker will use a
local copy of the image:

```
docker run ubuntu /bin/echo 'Hello world'
```
**Output:**
```
Hello world
```

!!! question
    Compare Ubuntu Docker image with ISO image or with Cloud VM image.

    * Why the size is so different ?


!!! summary
    Pulling docker images from Docker Hub takes sometime. This time depends on:

    * How large is the image?
    * How fast is the network to Internet ?

    However, it is still much faster than booting traditional OS with Ubuntu
    on VM.

    If image already pulled on local host it takes fraction of a second to start
    a container.

    Running application in docker containers considered as a best practice
    for running CI/CD pipelines as it considerably faster than using VMs and
    reduce time for deploying a test environments.


### 2.3 Specify an image version

**Step 1** As you see, Docker has downloaded the ubuntu:latest image. You can
see Ubuntu version by running the following command:

```
docker run ubuntu /bin/cat /etc/issue.net
```

**Output:**
```
Ubuntu 22.04 LTS
```

Let’s say you need a previous Ubuntu LTS release. In this case, you can specify
the version you need:

```
docker run ubuntu:14.04 /bin/cat /etc/issue.net
```
**Output:**
```
Unable to find image 'ubuntu:14.04' locally
14.04: Pulling from library/ubuntu
...
Status: Downloaded newer image for ubuntu:14.04
Ubuntu 14.04.4 LTS
```

**Step 2** The `docker images` command should show that we have 3 Ubuntu
images downloaded locally:

```
docker images

```
**Output:**
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              42118e3df429        11 days ago         124.8 MB
ubuntu              14.04               0ccb13bf1954        11 days ago         188 MB
hello-world         latest              c54a2cc56cbb        4 weeks ago         1.848 kB
```

!!! tip
    Running CI/CD pipeline with Docker using `latest` tag
    considered as a Bad Practice.

    Instead consider using:

    * Versioning
    * `SHA` tagging.


### 2.4 Run an interactive container

**Step 1** Let’s use the `ubuntu` image to run an *interactive* bash session and
inspect what is running inside our docker image.

To achive that we going to use  `-i` and `-t` flags.

The  `-i` is shorthand for `--interactive`, which instructs Docker to keep
`stdin` open so that we can send commands to the sprocess.

The `-t` flag is short for `--tty` and allocates a `pseudo-TTY` or terminal
inside of the session.

```
docker run -it ubuntu /bin/bash
root@17d8bdeda98e:/#
```

!!! result
    We get a  bash shell prompt inside of the container.

!!! note
    Bash prompt is not availabe for all docker images.

**Step 2** Let's print the system information of the latest `Ubuntu` image:
```
root@17d8bdeda98e:/#  uname -a
Linux 17d8bdeda98e 3.19.0-31-generic ...
```

**Step 3** Let's verify what Ubuntu version is run by `latest` image of ubuntu:
```
root@17d8bdeda98e:/#  lsb_release -a
bash: lsb_release: command not found
```

!!! failure
    Why the standard Ubuntu command that checks version of OS is not working as
    expeced ?

**Step 4** Let's verify Ubuntu version using alternative way by checking
`/etc/lsb-release` file.

```
root@8cbcbd0fe8d2:/# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.3 LTS"
```

**Step 5** Let's compare the number of executable binaries availabe inside of
the docker image versus Cloud VM that we running our class environment. First,
run `ls` command on  `/bin` and `/usr/bin` directories inside of the running
ubuntu container as well as `dpkg --list` command that shows total number of
installed packages:

```
root@8cbcbd0fe8d2:/# ls /bin | wc -l
294
root@8cbcbd0fe8d2:/# ls /usr/bin | wc -l
xx
root@eb11cd0b4106:/# dpkg --list | wc -l
xx
```

**Step 6** Use the `exit` command or press `Ctrl-D` to exit the interactive
bash session back to Cloud VM.
```
root@eb11cd0b4106:/# exit
```
**Step 7** Now run `ls` command on `/bin` and `/usr/bin` directories on Cloud VM
that we using as our class environment:

```
cca-user@userx-docker-vm:~$ ls /bin | wc -l
1173
cca-user@userx-docker-vm:~$ ls /usr/bin | wc -l
660
cca-user@userx-docker-vm:~$ dpkg --list | wc -l
463
```

!!! result
    Official Docker container has much less binaries and packages installed vs
    Ubuntu Cloud Image.

!!! summary
    Some of the use cases running docker containers in `interactive mode` are:

    * Troubleshooting containerized applications
    * Deploying and running containerized application on the existing
    production systems without affecting it.

    We've also learned that an official Docker "minimal" ubuntu image, does not
    include `lsb_release` command, as well as many other commands and packages
    that can be found in [Official Ubuntu ISO image](https://www.ubuntu.com/download/server).
    The docker images are ment to contain only required *core system* commands
    and functions to make Images as light as possible.
    That say you can still install required packages using `apt-get install`,
    however this may increase size of docker image considerably.


!!! hint
    While Docker `Ubuntu` image we used so far or Docker `Centos` image are
    very familiar to users and can be good starting point for learning docker
    containers. Using them in production or development considered as a Bad
    Practice.

    This is due those images still considered as `heavy` and potentially
    contain a lot more valnurabilities compare to specialized images.

    To reduce image pull time from docker hub and follow the best secuirity
    practices consider using specialized images that works well with you
    underlining code (Node image for NodeJS applications and etc.).
    Examples of specialized images are:

      * Alpine Linux
      * Node
      * Atomic

    In fact, not so long ago all the [official Docker Images](https://github.com/docker-library/official-images/tree/master/library)
    in Docker-Hub [has been moved](https://www.brianchristner.io/docker-is-moving-to-alpine-linux/)
    to use [Alpine Image](https://www.alpinelinux.org/).


**Step 8** Finally let’s check that when the shell process has finished, the
container stops:
```
docker ps
```

### 2.5 Run a container in a background

Now we know how to connect to running container and execute commands in it.
However in most cases you just want run a container in a *background* so it can
do a specific action.

**Step 1** Run a container in a background using the `-d` command line argument:

```
docker run -d ubuntu /bin/sh -c "while true; do date; echo hello world; sleep 1; done"
```
!!! result
    Command should return the container ID.

**Step 2** Let’s use the `docker ps` command to see running containers:
```
docker ps

```
```
 CONTAINER ID IMAGE  COMMAND                  CREATED        STATUS  PORTS NAMES
ac231579e57f ubuntu "/bin/sh -c 'while tr"   1 minute ago   Up 11 minute  evil_golick
```

!!! note
    Container id is going to  be different in your case

!!! hint
    Instead of using full `container-id` when building commands, it is possible
    simply type first few characters of container-id, to make things nice and
    easy.

**Step 3** Let’s use `container-id` to show the container standard output:
```
docker logs <container-id>

```
```
Thu Jan 26 00:23:45 UTC 2017
hello world
Thu Jan 26 00:23:46 UTC 2017
hello world
Thu Jan 26 00:23:47 UTC 2017
hello world
...
```

As you can see, in the `docker ps` command output, the auto generated container
name is `evil_golick` (your container can have a different name).

**Step 4** Now, instead of using docker `contaier-id` use container name to
show the container standard output:
```
docker logs <name>

```
```
Thu Jan 26 00:23:51 UTC 2017
hello world
Thu Jan 26 00:23:52 UTC 2017
hello world
Thu Jan 26 00:23:53 UTC 2017
hello world
...
```
**Step 5** Finally, let’s stop our container:
```
docker stop <name>

```
**Step 6** Check, that there are no running containers:
```
docker ps
```

!!! summary
    `docker logs` is a very usefull command to troubleshoot containers,
    and going to be used very often both for Docker and Kubernertes
    troubleshooting.


### 2.6 Accessing Containers from the Internet

**Step 1** Let’s run a simple web application. We will use the existing image
training/webapp, which contains a Python Flask application:

```
docker run -d -P training/webapp python app.py
```
```
...
Status: Downloaded newer image for training/webapp:latest
6e88f42d3d853762edcbfe1fe73fdc5c48865275bc6df759b83b0939d5bd2456
```

In the command above we specified the main process (python app.py), the `-d`
command line argument, which tells Docker to run the container in the
background. The `-P` command line argument tells Docker to map any required
network ports inside our container to our host. This allows us to access the web
application in the container.

**Step 2** Use the `docker ps` command to list running containers:

```
docker ps
```
```
CONTAINER ID IMAGE           COMMAND         CREATED       STATUS       PORTS                   NAMES
6e88f42d3d85 training/webapp "python app.py" 3 minutes ago Up 3 minutes 0.0.0.0:32768->5000/tcp determined_torvalds
```

The PORTS column contains the mapped ports. In our case, Docker has exposed port
5000 (the default Python Flask port) on port 32768 (can be different in your
case).

**Step 3** The `docker port` command shows the exposed port. We will use the
container name (determined_torvalds in the example above, it can be different in
your case):

```
docker port <name> 5000
0.0.0.0:32768
```

**Step 4** Let’s check that we can access the web application exposed port:
```
curl http://localhost:<port>/
```

!!! result
    `Hello world!`

**Step 5** Let’s stop our web application for now:
```
docker stop <name>
```

**Step 6** We want to manually specify the local port to expose (-p argument).
Let’s use the standard HTTP port 8080. We also want to specify the container name
(--name argument):

```
docker run -d -p 8080:5000 --name webapp training/webapp python app.py
```

**Step 7** Let’s check that the port 8080 is exposed:

```
docker ps
```

```

CONTAINER ID IMAGE           COMMAND         CREATED       STATUS       PORTS                NAMES
249476631f7d training/webapp "python app.py" 1 minute  ago Up 1 minute  0.0.0.0:80->5000/tcp webapp
```


```

curl http://localhost/
```
!!! result
    `Hello world!`


**Step 4** Now that we've launched the application containers, let's try to test the web
application locally.

You should be able to access the application at Google Cloud `Web Preview` Console:

![alt text](images/web_preview.png "Web Preview")

!!! note
    Web Preview using port `8080` by default. If you application using other port, you can edit this as needed.

!!! result
    Our web-app can be accessed from Internet!


### 2.7 Restart a container

**Step 1** Let’s stop the container with web application:
```
docker stop webapp
```
The main process inside of the container will receive SIGTERM, and after a grace
period, SIGKILL.

**Step 2** You can start the container later using the `docker start` command:

```
docker start webapp
```

**Step 3** Check that the web application works:

![alt text](images/web_preview.png "Web Preview")


**Step 4** You also can restart the running container using the docker restart
command.
```
docker restart webapp
```

**Step 4**  Run `docker ps` command and check  `STATUS` field:

```
docker ps
```
```
CONTAINER ID    IMAGE             COMMAND           CREATED           STATUS              
6e400179070f    training/webapp   "python app.py"   25 minutes ago    Up 3 seconds
```

### 2.8 Ensuring Container Uptime

Docker considers any containers to exit with a non-zero exit code to have
crashed. By default a crashed container will remain stopped.

**Step 1** Start the container that outputs a message and then exits with code 1
to simulate a crash.
```
docker run -d --name restart-default scrapbook/docker-restart-example
```
```
docker ps -a | grep restart-default
```
```

CONTAINER ID  IMAGE                             CREATED       STATUS               NAMES
c854289d2f39  scrapbook/docker-restart-example  5 seconds ago Exited 3 sec ago    restart-default
$ docker logs restart-default
Sun Sep 17 20:34:55 UTC 2017 Booting up...
```

!!! result
    Container crushed and exited.

However, there are several ways to ensure that you container up and running even
if it’s restarts.

**Step 2** The option `--restart=on-failure`: allows you to say how many times
Docker should try again:

```
docker run -d --name restart-3 --restart=on-failure:3 scrapbook/docker-restart-example
```

```
docker logs restart-3
```

```
Thu Apr 20 14:01:27 UTC 2017 Booting up...
Thu Apr 20 14:01:28 UTC 2017 Booting up...
Thu Apr 20 14:01:29 UTC 2017 Booting up...
Thu Apr 20 14:01:31 UTC 2017 Booting up...
```

**Step 3** Finally, Docker can always restart a failed container. In this case,
Docker will keep trying until the container is explicitly told to stop.
```
docker run -d --name restart-always --restart=always scrapbook/docker-restart-example
docker logs restart-always
```

**Step 4**  After sometime stop running docker container, as it will be keep
failing and starting again:
```
docker stop restart-always
```

### 2.9 Inspect a container

**Step 1** You can use the `docker inspect` command to see the configuration and
status information for the specified container:

```
docker inspect webapp
```
```
[
    {
        "Id": "249476631f7d...",
        "Created": "2016-08-02T23:42:56.932135327Z",
        "Path": "python",
        "Args": [
            "app.py"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 16055,
            "ExitCode": 0,
            "Error": "",
            ...
```

**Step 2** You can specify a filter (-f command line argument) to show only
specific elements. For example:

```
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webapp
```
```

172.17.0.2
```
The command returns the IP address of the container.

### 2.10 Interacting with containers
In some cases using `docker log` is not enough to undertand issues and you
want to login inside of running VM. Also sometimes you package you applicaiton
and in order to run it you need to login inside of container and execute and
leave it running in background. Below provded few ways to interacting with
containers that can help to achive descrined use cases.


### 2.10.1 Detach from Interactive container
In Module, `1.4 Run an interactive container` we run an `Ubuntu` container with
`-it` flag and able directly login inside of the container to interact with it,
however after we exited contianer using `Ctrl-D` or `exit` command container
stopped. However you can exit from `Interactive mode` without stoping
a container. Let's demonstrate how this works:

**Step 1** Start Ubunu container in interactive  mode:
```
docker run -it ubuntu /bin/bash
```

**Step 2** Run `watch date` command inside running container in order to exit
`date` command every 2 seconds.
```
root@1d688a9f4ed4:/# watch date
```

**Step 3** Detach from a container and leave it running using the
`CTRL-p` `CTRL-q` key sequence.


**Step 4** Verify that Ubuntu container is still running:
```
docker ps
```
```
CONTAINER ID  IMAGE   COMMAND      CREATED        STATUS        NAMES
1d688a9f4ed4  ubuntu  "/bin/bash"  1 minutes ago  Up 1 minutes  admiring_lovelace
```

!!! result
    Great you were able to detach from Docker container without stopping it,
    while it is executing a process in it.
    What about attaching back to container ?

!!! important
    `CTRL-p` `CTRL-q`  sequence key only works if docker contaienr started
    with `-it` command!

### 2.11.2 Attach to a container

Now let's get back and `attach` to our running Ubuntu image. For that
docker provides `docker attach` command.

```
docker attach <container name>
```
```
Every 2.0s: date                                                                                                                    Mon Sep 18 00:08:57 2017

```

!!! summary
    `docker attach` attaches your contairs terminal’s standard input, output,
    and error (or any combination of the 3) to a running container. This allows
    you to view its ongoing output or to control it interactively, as though
    the commands were running directly in your terminal.

### 2.11.3 Execute a process in a container

**Step 1** Let verify if webapp container is still running

```
docker ps
```
```
CONTAINER ID IMAGE           COMMAND         CREATED       STATUS       PORTS                NAMES
249476631f7d training/webapp "python app.py" 1 minute  ago Up 1 minute  0.0.0.0:80->5000/tcp webapp

```

If not running start it with following command: `$ docker run -d -p 80:5000 --name webapp training/webapp python app.py`
other wise skip to **next step**.


**Step 2** Use the docker exec command to execute a command in the running
container. For example:
```
docker exec webapp ps aux

```
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.2  0.0  52320 17384 ?        Ss   00:11   0:00 python app.py
root        26  0.0  0.0  15572  2104 ?        Rs   00:12   0:00 ps aux
```
The same command with the `-it` command line argument can be used to run an
interactive session in the container:
```
docker exec -it webapp bash
root@249476631f7d:/opt/webapp# ps auxw
ps auxw

```
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  52320 17384 ?        Ss   00:11   0:00 python app.py
root        32  0.0  0.0  18144  3064 ?        Ss   00:14   0:00 bash
root        47  0.0  0.0  15572  2076 ?        R+   00:16   0:00 ps auxw
```
**Step 2** Use the `exit` command or press `Ctrl-D` to exit the interactive bash
session:
```
root@249476631f7d:/opt/webapp# exit
```

!!! Summary
    `docker exec` is one of the most usefull docker commands used for
    troubleshooting containers.

### 2.12 Copy files to/from container

The `docker cp` command allows you to copy files from the container to the local
machine or from the local file system to the container. This command works for
a running or stopped container.

**Step 1** Let’s copy the container’s app.py file to the local machine:
```
docker cp webapp:/opt/webapp/app.py .
```
**Step 2** Edit the local app.py file. For example, change the line return
'Hello '+provider+'!' to return 'Hello '+provider+'!!!'. Copy the modified file
back and restart the container:
```
docker cp app.py webapp:/opt/webapp/

docker restart webapp
```
**Step 3** Check that the modified web application works::
```
curl http://localhost/

```

!!! result
    `Hello world!!!``



### 2.12 Remove containers
Now let's clean up the environment and at the same time learn how delete
containers.

Step 1 First list running containers:
```
docker ps
```
```
CONTAINER ID IMAGE           COMMAND         CREATED       STATUS       PORTS                NAMES
81c4c66baaf9 training/webapp "python app.py" 1 minute  ago Up 1 minute  0.0.0.0:80->5000/tcp webapp
```

Step 2 Than try to delete running container using `docker rm <container_id>`
```
docker rm $container_id
```
```
Error response from daemon: You cannot remove a running container 81c4c66baaf9. Stop the container before attempting removal or force remove.
```

!!! failure
    Docker containers needs to be first stopped or deleted using `--force` flag.

```
docker rm $container_id -f
```

Alternatively, you can run `stop` and `rm` in sequence:
```
docker stop 81c4c66baaf9
docker rm 81c4c66baaf9
```

## 3 Building Images with DockerFile

In the previous modules, we learned how to use Docker images
to run Docker containers. Docker images that we used have been downloaded from
the Docker Hub, a Docker image registry maintained by Docker Inc. In this section we will create a
simple web application from scratch. We will use Flask
([http://flask.pocoo.org/](http://flask.pocoo.org/)), a microframework for
Python. Our application for each request will display a random picture from the
defined set.

The code for this application is also available in GitHub:
```
https://github.com/Cloud-Architects-Program/ycit019_2022/tree/main/Module5/flask-app
```
### 3.1 Overview DOCKERFILE Creation

**Step 1** Clone git repo on you laptop:

```
git clone https://github.com/Cloud-Architects-Program/ycit019_2022
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

### 3.2 Build a Docker image

**Step 1** Now let’s build our Docker image. In the command below, replace
<user-name> with your user name. This user name should be the same as you
created when you registered on Docker Hub.
```
docker build -t <user-name>/myfirstapp .
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
docker run -dp 8080:5000 --name myfirstapp <user-name>/myfirstapp
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


!!! summary
    We've learned a lot of docker commands which are very handy to know both
    when using Docker and Kubernetes. Next we will learn how to create Docker networks.


## 4 Docker Networking

### 4.1 Docker Networking Basics

**Step 1:** The Docker Network Command
The `docker network` command is the main command for configuring and managing container networks. Run the `docker network` command from the first terminal.

```.term1
docker network
```
```
Usage:	docker network COMMAND

Manage networks

Options:
      --help   Print usage

Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks

Run 'docker network COMMAND --help' for more information on a command.
```

The command output shows how to use the command as well as all of the `docker
network` sub-commands. As you can see from the output, the `docker network`
command allows you to create new networks, list existing networks, inspect
networks, and remove networks. It also allows you to connect and disconnect
containers from networks.

**Step 2** Run a `docker network ls` command to view existing container networks
on the current Docker host.

```.term1
docker network ls
```
```
NETWORK ID          NAME                DRIVER              SCOPE
3430ad6f20bf        bridge              bridge              local
a7449465c379        host                host                local
06c349b9cc77        none                null                local
```
The output above shows the container networks that are created as part of a
standard installation of Docker.

New networks that you create will also show up in the output of the
`docker network ls` command.

You can see that each network gets a unique `ID` and `NAME`. Each network is
also associated with a single driver. Notice that the "bridge" network and the
"host" network have the same name as their respective drivers.

**Step 3:** The `docker network inspect` command is used to view network
configuration details. These details include; name, ID, driver, IPAM driver,
subnet info, connected containers, and more.

Use `docker network inspect <network>` to view configuration details of the
container networks on your Docker host. The command below shows the details of
the network called `bridge`.

```.term1
docker network inspect bridge
```
```
[
    {
        "Name": "bridge",
        "Id": "3430ad6f20bf1486df2e5f64ddc93cc4ff95d81f59b6baea8a510ad500df2e57",
        "Created": "2017-04-03T16:49:58.6536278Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

!!! note
    The syntax of the `docker network inspect` command is `docker network
    inspect <network>`, where `<network>` can be either network name or
    network ID. In the example above we are showing the configuration details
    for the network called "bridge". Do not confuse this with the "bridge"
    driver.


**Step 4** Now, list Docker supported network driver plugins. For that run
`docker info` command, that  shows a lot of interesting information about a
Docker installation.

Run the `docker info` command and locate the list of network plugins.

```.term1
docker info
```
```
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 17.03.1-ee-3
Storage Driver: aufs
<Snip>
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
Swarm: inactive
Runtimes: runc
<Snip>
```

The output above shows the **bridge**, **host**,**macvlan**, **null**, and
**overlay** drivers.

!!! summary
    We've quickly reviewed available docker networking commands as well as
    found what drivers current docker setup supports.


### 4.2 Default bridge network
Every clean installation of Docker comes with a pre-built network called
**Default bridge network**. Let's explore in more details how it works.

**Step 1** Verify this with the `docker network ls`.

```.term1
docker network ls
```
```
NETWORK ID          NAME                DRIVER              SCOPE
3430ad6f20bf        bridge              bridge              local
a7449465c379        host                host                local
06c349b9cc77        none                null                local
```

!!! result
    The output above shows that the **bridge** network is associated with the
    *bridge* driver. It's important to note that the network and the driver are
    connected, but they are not the same. In this example the network and the
    driver have the same name - but they are not the same thing!

    The output above also shows that the **bridge** network is scoped locally.
    This means that the network only exists on this Docker host. This is true
    of all networks using the *bridge* driver - the *bridge* driver provides
    single-host networking.

All networks created with the *bridge* driver are based on a Linux bridge
(a.k.a. a virtual switch).


**Step 5** Start webapp in Default bridge network

```
docker run -d -p 80:5000 --name webapp training/webapp python app.py
```

**Step 6** Check that the webapp and db containers are running:

**Command:**

```.term1
docker ps
```

### 4.3 User-defined Private Networks


So far we’ve learned how Docker networking works with **Docker default bridge
network**. With the introduction of user-defined networking in Docker 1.9, it
is now possible to create **multiple Docker bridges** to allow network
segregation within the same host or **multi-host networking** to allow
communicate Docker containers between hosts.

The commands are available through the Docker Engine CLI are:

```
docker network create
docker network connect
docker network ls
docker network rm
docker network disconnect
docker network inspect
```
Let's demonstrate how to create a custom bridge network.


**Step 1** By default, Docker runs containers in the bridge network. You may
want to isolate one or more containers in a separate network. Let’s create a
new network:
```
docker network create my-network \
-d bridge \
--subnet 172.19.0.0/16
```

The `-d` bridge command line argument specifies the bridge network driver and
the `--subnet` command line argument specifies the network segment in CIDR
format. If you do not specify a subnet when creating a network, then Docker
assigns a subnet automatically, so it is a good idea to specify a subnet to
avoid potential conflicts with the existing networks.

Below are some other options that are available with the bridge Driver:

  * com.docker.network.bridge.enable_ip_masquerade: This instructs the Docker
  host to hide or masquerade all containers in this network behind the Docker
  host's interfaces if the container attempts to route off the local host .

  * com.docker.network.bridge.name: This is the name you wish to give to the
  bridge.

  * com.docker.network.bridge.enable_icc: This turns on or off Inter-Container
  Connectivity (ICC) mode for the bridge.

  * com.docker.network.bridge.host_binding_ipv4: This defines the host interface
  that should be used for port binding.

  * com.docker.network.driver.mtu: This sets MTU for containers attached to this
  bridge.

**Step 2** To check that the new network is created, execute docker network ls:
```
docker network ls
```
```

NETWORK ID          NAME                DRIVER              SCOPE
d428e49e4869        bridge              bridge              local
0d1f78528cc5        host                host                local
56ef0481820d        my-network          bridge              local
4a07cef84617        none                null                local
```

**Step 3** Let’s inspect the new network:
```
docker network inspect my-network

```
```
[
    {
        "Name": "my-network",
        "Id": "56ef0481820d...",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

**Step 4** As expected, there are no containers connected to the my-network.
Let’s recreate the db container in the my-network:
```
docker rm -f db

```

```

docker run -d --network=my-network --name db training/postgres
```

**Step 5** Inspect the `my-network `again:
```
docker network inspect my-network

```
**Output:**
```...
    "Containers": {
        "93af62cdab64...": {
            "Name": "db",
            "EndpointID": "b1e8e314cff0...",
            "MacAddress": "02:42:ac:12:00:02",
            "IPv4Address": "172.19.0.2/16",
            "IPv6Address": ""
        }
    },
...
```
As you see, the `db` container is connected to the my-network and has 172.19.0.2
address.

**Step 6** Let’s start an interactive session in the db container and ping the
IP address of the webapp again:

!!! note
    Quick reminder how to locate webapp ip:

     `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webapp`

```
docker exec -it db bash

```
Once inside of container run:
```
root@c3afff20019a:/# ping -c 1 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.

--- 172.17.0.3 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

As expected, the webapp container is no longer accessible from the db container,
because they are connected to different networks.

!!! summary
    Using `Multi-host networking` provides network isolation within a Docker
    host via network namepsaces. This is can be used if you want to deploy
    different applications on same host for isolation or resource duplicate
    prevention.

**Step 7** Let’s connect the webapp container to the my-network:
```
docker network connect my-network webapp
```

**Step 8** Check that the webapp container now is connected to the my-network:
```
docker network inspect my-network

```
**Output:**

```
...
    "Containers": {
        "62ed4a627356...": {
            "Name": "webapp",
            "EndpointID": "ae95b0103bbc...",
            "MacAddress": "02:42:ac:12:00:03",
            "IPv4Address": "172.19.0.3/16",
            "IPv6Address": ""
        },
        "93af62cdab64...": {
            "Name": "db",
            "EndpointID": "b1e8e314cff0...",
            "MacAddress": "02:42:ac:12:00:02",
            "IPv4Address": "172.19.0.2/16",
            "IPv6Address": ""
        }
    },
...
```

The output shows that two containers are connected to the my-network and the
webapp container has 172.19.0.3 address in that network.

**Step 9** Check that the webapp container is accessible from the db container
using its new IP address:
```
docker exec -it db bash
```

```
root@c3afff20019a:/# ping -c 1 172.19.0.3
PING 172.19.0.3 (172.19.0.3) 56(84) bytes of data.
64 bytes from 172.19.0.3: icmp_seq=1 ttl=64 time=0.136 ms

--- 172.19.0.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.136/0.136/0.136/0.000 ms
```


!!! success
    As expected containers can communicate with each other.



**Step 10** You can now remove the existing container. You should stop the
container before removing it. Alternatively you can use the -f command line
argument:

```
docker rm -f webapp
docker rm -f db
docker network rm  my-network
```


!!! hint
    Use below command to delete running containers in **bulk**:

    ```
    docker rm -f $(docker ps -q)
    ```


!!! summary
    It is recommended to use user-defined bridge networks to control which
    containers can communicate with each other, and also to enable automatic
    DNS resolution of container names to IP addresses


