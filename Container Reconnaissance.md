# Container Reconnaissance

### Attack surface of containers

1. Vulnerable base images or image layers
2. Attacks through exposed container networks
3. Attacks through file systems
4. Attack through vulnerable applications running in the container

###  Keep Containers alive continued
There are three way to keep containers alive:

1. Open a pseudo tty(-t) \
`$ docker run -d -t ubuntu:18.04`
2. Create an Infinite loop using sleep or any command \
`$ docker run -d ubuntu:18.04 sleep infinity` \
`$ docker run -d -it alphine sh -c "while true; do sleep 2; done"`
3. Override entypoint or run an intractive command \
`$ docker run -d ubuntu:18.04 tail -f /dev/null`
4. Pause/Unpause an image \
`docker pause alp1` \
`docker unpause alp1`

### Identfy a container using container id
You can get a container id by using -d option which detaches if from the terminal 
```
$ docker run -d alphine
e26dc2lfx....
```
docker allows us to name containers using the --name argumant \
``$ docker run --name myAlphine alphine`` 

### docker save vs docker export


1. **docker save**

- Used on images
- Saves the image layers, history
- Larger images

2. **docker export**

- Used on running containers
- Doesn't store the history
- Smaller images

## Use Third-party Tools for Image Inspection

1. Install dive
```
Dive is a tool for exploring a docker image, layer contents, and discovering ways to shrink the size of your Docker/OCI image.
```
Let’s pull a Python image to delve deeper into this exploration. \
`` docker pull python:3.4-alpine ``

Let’s download and install dive.
```
wget https://github.com/wagoodman/dive/releases/download/v0.10.0/dive_0.10.0_linux_amd64.deb

dpkg -i dive_0.10.0_linux_amd64.deb

```
Let’s dive. \
``dive python:3.4-alpine``

2. Install whaler
```
Whaler helps to reverse engineer docker images into the Dockerfile. It also helps to search for potential secret files and displays miscellaneous information.
```
Let’s download and install whaler. \
``wget -O /usr/local/bin/whaler https://github.com/P3GLEG/Whaler/releases/download/1.0/Whaler_linux_amd64``

``chmod +x /usr/local/bin/whaler``

Check if whaler command is installed in our system.

``whaler --help``

---

By default docker containers runs at the **root**  user.

---
### Docker Networking ###

Docker has several network drivers that we connect that we can use when creating the containers, inculude following
1. Bridge \
• Default network driver \
• Containers on the same bridged network can speak to each other \
• Containers on a bridged network can't connect to containers on other bridge \
• Access to external network is allowed through NAT 
2. Host \
• Use the host's networking directly \
• Ports exposed by the container are exposed on the external network using the host's IP
3. Macvlan \
• Attach VLAN to the host's network device (e.g. "ethO") \
• Each container on the macvian network will receive its own MAC address \
• Each container has full network
4. None \
• Networking is disabled \
• Containers cannot communicate to each other \
• Containers cannot communicate with external 

---

### Docker Persistance ###

1. Bind mounts \
A container can save the output/result/data using bind mounts. Think of them like a shared folder.
You can share a folder using the -v/-volume option
```
$ docker run -v HOST_DIR:DOCKER_DIR image-name
$ docker run -v ./app:/app django.nv:1.0
$ docker run -v $(pwd):/app
```
2. Volumes \
Unlike bind mounts, volumes are created, managed by Docker and are platform independent.
Manages permissions and other issues with ease. Volumes are the preferred way of sharing the data as its not tied to underlying file system. \
You can share the volumes in a similar way as bind mount.
```
$ docker run -v VOLUME:DOCKER_DIR image-name
$ docker run -v demo-volume:/app django.nv:1.0
$ docker run -v demo-volume:/app django.nv:1.0

# Stored under /var/lib/docker/volumes
```

3. tmpts

---

### cp of docker ###

Docker cp allows you to copy data from container into the host. \
Copy data from a container with docker cp 

``$ docker cp ubuntu1:/opt/tf-output.json .``

---

### Docker Registry

A registry is a place where you can store and retrieve Docker images. Docker supports both public and private registries.

---

### Analysis of The Attack Surface

• List networks and identify the network attached to the containers \
• Identify mount points to explore information that could be exposed through the file system \
• Use docker inspect to gather container information \
• Identify history of an image and its layers \
• Identify exposed ports to analyze any vulnerable applications \
• Review the resource limits of containers

```
# List Networks
docker network ls

# List Volumes
docker volumes ls

#Inspect various docker objects (Displays internal details about a docker object.)
docker inspect containerid 
docker inspect imagename 
docker network inspect

# Docker system information
docker system events 
docker system --help

# Image Layers and Dockerfile instructions
docker history imageid

# Get running processes inside a container
docker top containerid

# Get container resource limits for memory, cpu
# Review current resource consumption of a container.
docker stats
docker stats con-stats

```

---

### Are You Inside a Container?

1. File System \
**Listing the root dir**

Often times there is a file name `.dockerenv` in the root directory of a container

- `ls -al .dockerenv`

2. Cgroups
**Listing the cgroups**

Reviewing the init processes (PID 1) cgroup will show docker mounts

- `cat /proc/1/cgroup`
- `cat /proc/self/cgroup`








