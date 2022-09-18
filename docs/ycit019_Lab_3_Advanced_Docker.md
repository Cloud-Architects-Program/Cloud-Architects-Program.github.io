Lab 3 Docker Networking, Persistence, Monitoring and Logging

**Objective:**

* Networks
    * Docker basics
    * User-defined private Networks
* Persistence
    * Data Volumes

## 1 Docker Networking

### 1.1 Docker Networking Basics

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


### 1.2 Default bridge network
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

### 1.3 User-defined Private Networks


So far we’ve learned how Docker networking works with **Docker default bridge
network**. With the introduction of user-defined networking in Docker 1.9, it
is now possible to create **multiple Docker bridges** to allow network
segregation within the same host or **multi-host networking** to allow
communicate Docker containers between hosts.

The commands are available through the Docker Engine CLI are:

```docker network createdocker network connectdocker network lsdocker network rmdocker network disconnectdocker network inspect
```
Let's demonstrate how to create a custom bridge network.


**Step 1** By default, Docker runs containers in the bridge network. You may
want to isolate one or more containers in a separate network. Let’s create a
new network:
```
docker network create my-network \-d bridge \--subnet 172.19.0.0/16
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
NETWORK ID          NAME                DRIVER              SCOPEd428e49e4869        bridge              bridge              local0d1f78528cc5        host                host                local56ef0481820d        my-network          bridge              local4a07cef84617        none                null                local
```

**Step 3** Let’s inspect the new network:
```
docker network inspect my-network
```
```
[    {        "Name": "my-network",        "Id": "56ef0481820d...",        "Scope": "local",        "Driver": "bridge",        "EnableIPv6": false,        "IPAM": {            "Driver": "default",            "Options": {},            "Config": [                {                    "Subnet": "172.19.0.0/16"                }            ]        },        "Internal": false,        "Containers": {},        "Options": {},        "Labels": {}    }]
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
```...    "Containers": {        "93af62cdab64...": {            "Name": "db",            "EndpointID": "b1e8e314cff0...",            "MacAddress": "02:42:ac:12:00:02",            "IPv4Address": "172.19.0.2/16",            "IPv6Address": ""        }    },...
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
root@c3afff20019a:/# ping -c 1 172.17.0.3PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.--- 172.17.0.3 ping statistics ---1 packets transmitted, 0 received, 100% packet loss, time 0ms
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
...    "Containers": {        "62ed4a627356...": {            "Name": "webapp",            "EndpointID": "ae95b0103bbc...",            "MacAddress": "02:42:ac:12:00:03",            "IPv4Address": "172.19.0.3/16",            "IPv6Address": ""        },        "93af62cdab64...": {            "Name": "db",            "EndpointID": "b1e8e314cff0...",            "MacAddress": "02:42:ac:12:00:02",            "IPv4Address": "172.19.0.2/16",            "IPv6Address": ""        }    },...
```

The output shows that two containers are connected to the my-network and the
webapp container has 172.19.0.3 address in that network.

**Step 9** Check that the webapp container is accessible from the db container
using its new IP address:
```
docker exec -it db bash
```

```
root@c3afff20019a:/# ping -c 1 172.19.0.3PING 172.19.0.3 (172.19.0.3) 56(84) bytes of data.64 bytes from 172.19.0.3: icmp_seq=1 ttl=64 time=0.136 ms--- 172.19.0.3 ping statistics ---1 packets transmitted, 1 received, 0% packet loss, time 0msrtt min/avg/max/mdev = 0.136/0.136/0.136/0.000 ms
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

### 1.4 Access containers from outside

External Access to the Containers can be configured via publishing mechanism.

Docker provides 2 options to publish ports:

  * `-P` flag publishes all exposed ports
  * `-p` flag allows you to specify specific ports and interfaces to use when
    mapping ports.

The `-p` flag can take several different forms with the syntax looking like this:

  * Specify the host port and container port:

    `–p <host port>:<container port>`

  * Specify the host interface, host port, and container port:

    `–p <host IP interface>:<host port>:<container port>`

  * Specify the host interface, have Docker choose a random host port, and specify
the container port:

    `–p <host IP interface>::<container port>`

   * Specify only a container port and have Docker use a random host port:

    `–p <container port>`

Let's test exposing containers. For that let's start a new NGINX container and
map port 8080 on the Docker host to port 80 inside of the container. This means
that traffic that hits the Docker host on port 8080 will be passed on to port
80 inside the container.

!!! note
    If you start a new container from the official NGINX image without specifying a
    command to run, the container will run a basic web server on port 80.

**Step 1** Start a new container based off the official NGINX image by running `docker run --name web1 -d -p 8080:80 nginx`.

```.term1
docker run --name web1 -d -p 8080:80 nginx
```
```
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
6d827a3ef358: Pull complete
b556b18c7952: Pull complete
03558b976e24: Pull complete
9abee7e1ef9d: Pull complete
Digest: sha256:52f84ace6ea43f2f58937e5f9fc562e99ad6876e82b99d171916c1ece587c188
Status: Downloaded newer image for nginx:latest
4e0da45b0f169f18b0e1ee9bf779500cb0f756402c0a0821d55565f162741b3e
```

**Step 2** Review the container status and port mappings by running `docker ps`.

```.term1
docker ps
```
```
CONTAINER ID   IMAGE   COMMAND                  PORTS                           NAMES
4e0da45b0f16   nginx   "nginx -g 'daemon ..."   443/tcp, 0.0.0.0:8080->80/tcp   web1
```

!!! result
    The top line shows the new **web1** container running NGINX. Take note of the
    command the container is running as well as the port mapping
    - `0.0.0.0:8080->80/tcp` maps port 8080 on all host interfaces to port 80
    inside the **web1** container. This port mapping is what effectively makes the
    containers web service accessible from external sources (via the Docker hosts
    IP address on port 8080).

**Step 3** Test connectivity to the NGINX web server, by pasting `<Public_IP:8080>`
of VM to the browser.

!!! note
    In order to locate Public IP see the list of VMs.


Alternatively from inside of VM run `curl 127.0.0.1:8080` command.

```.term1
curl 127.0.0.1:8080
```
```
<!DOCTYPE html>
<html>
<Snip>
<head>
<title>Welcome to nginx!</title>
    <Snip>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

!!! success
    Both CLI and UI method works!

If you try and curl the IP address on a different port number it will fail.

!!! summary
    Docker provides easy way to expose containers outside of the Docker Node.
    This can ber used for connecting containers between each other:

      * Between networks on the same host
      * Between networks on different host
      * Accessing containers from outside (e.g web site)

    However, port mapping is implemented via port address translation (PAT)
    unlike in Kubernetes which we learn soon, exposes applications via
    service IPs and communicates via POD IPs using (NAT)

**Step 4** Cleanup environment
```
docker rm -f $(docker ps -q)
```

## 2 Persistant Volumes

### 2.1 Storage driver
We've discussed several Storage drivers (graphdrivers) during the class.
Let's find out what graphdriver is running in our Lab environment.

```
docker info | grep  Storage
```
```
WARNING: No swap limit support
Storage Driver: aufs
```

!!! result
    Our Classroom is running `aufs` storage driver. Not a suprise as we running
    our Lab on Ubuntu VM.

!!! summary
    Systems runnng Ubuntu or Debian ,going to run `aufs` graphdriver by
    default and will most likely meet the majority of your needs.  
    In future `overlay2` may replace `aufs` stay tunned!

### 2.2 Persisting Data Using Volumes

Docker Volumes are created and assigned when containers are started. Data
Volumes allow you to map a host directory to a container for sharing data.

This mapping is bi-directional. It allows data stored on the host to be accessed
from within the container. It also means data saved by the process inside the
container is persisted on the host.

#### 2.2.1  Create and manage volumes

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


#### 2.2.2 Start a container with a volume
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
 ```       "Mounts": [            {                "Type": "volume",                "Name": "39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d",                "Source": "/var/lib/docker/volumes/39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d/_data",                "Destination": "/webapp",                "Driver": "local",                "Mode": "",                "RW": true,                "Propagation": ""
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
ls dbhello_from_db_container
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
root@47a60c01590e:/# ls /dbhello_from_db_containerroot@47a60c01590e:/# exit
```
#### 2.2.3 Use a read-only volume

**Step 1** Mounting Volumes gives the container **full read and write access** to the
directory. You can specify **read-only permissions** on the directory by adding
the **permissions :ro** to the mount. If the container attempts to modify data
within the directory it will error.

```
docker run -d --name db1 -v ~/db:/db:ro training/postgres
docker exec -it db1 bash
cd db
touch test
```

!!! result

```
touch: cannot touch 'test': Read-only file system
$ exit
```

** Step 2** Clean up containers and volumes:

```
docker rm -f $(docker ps -q)
docker volume ls
```
!!! output
```
DRIVER              VOLUME NAME
local               39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d
```

```
docker volume rm 39885c52758dcf7516513be2d44a17560e42b6da75aba30bc66d4af41df5384d
```


!!! summary
    We've learned how to manage volumes with containers

!!! hint
    If you Docker host has several Storage plugins configured (e.g. ceph,
    gluster) you can specify via `--opt type=btrfs, nfs` or `--driver=glusterfs`
    during docker volume creation.
