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
Dive is a tool for exploring a docker image, layer contents,
and discovering ways to shrink the size of your Docker/OCI image.
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

Other commands
- `ps aux | grep dockerd`
- `ps aux | grep containerd`
- `cat /proc/self/mountinfo`

---

### Container uses these three kernel features

1. Namespaces \
Namespaces allow docker to create a set of segregated resources like process (pid), networking (net), file system mounts (mnt), communication (ipc), user (user) and kernel identifiers. \
It's an isolation mechanism to provide separation. Using this feature, every container believes it's the only container in the system.

-> Important Commands 
1) nsenter \
Run a program in different namespaces. In layman terms, enter into an existing namespace and execute a command \
``nsenter --target PID -NAMESPACE COMMAND`` \
``nsenter --target 897 --net ip addr`` \
``nsenter --target 897 --mount Is /root/``
2) unshare \
Unshares the indicated namespaces from the parent process and then executes the specified program. In layman terms, create new namespace and execute a command in that namespace \
``unshare -Ur -u sh # create a user namespace with root privileges and uts namespace``
2. Cgroups \
Control groups are responsible for resource meeting and limiting as the name control suggests. \
It controls the following resources/groups
    1) Memory
    2) CPU
    3) IO
    4) Network 

It also handles device node(/dev/*) access control \
Many of these restrictions are not enabled by default while running containers.

3. Capabilities \
capabilities allow docker to further constraint the root privileges and add some root privileges to non-root users.

Docker by default enables the following 14 capabilities

```
CHOWN, DAC_OVERRIDE, FSETID, FOWNER,KILL, MKNOD, NET_RAW, SETGID, SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT, AUDIT_ WRITE
```

**In short** \
• Cgroups - limits on resources (CPU, RAM etc.,) \
• Namespaces - process separation and isolation. \
• Capabilities - Restrict root permissions \
• SecComp - Avoid dangerous system calls

---

To display only the network information of a container and reduce the output size, you can use the following command: \
``docker inspect -f "{{json .NetworkSettings.Networks }}" <container_name>``
``docker inspect alpine1 -f "{{json .NetworkSettings.Networks }}" | jq``

---

**Exploring Docker File Mounts**

``docker volume ls`` \
``docker inspect alpine3 -f "{{json .Mounts}}" | jq``

---

** Exploring the Layers of a Docker Image **

``docker inspect c06adcf62f6e<image_id>`` \
``docker history c06adcf62f6e``

---

**Exploring System Information and Events of Docker**

``docker system info``

You can also monitor events as they happen in the docker system using docker system events.

``docker system events >> events &``

---

**Exploring Container Processes and Resource Limits**

``docker top alpine4``

``docker exec alpine4 'env'``

---

Cgroups and Resource limits

By default docker stats shows stats for running containers. \
``docker stats``

---

### Challenges

1. Task 1 \
Run the Nginx image with the name nginx-limit-memory, using the latest version, and limit the memory to 512 MB \
``docker run -dit --name nginx-limit-memory --memory="512m" nginx:latest``

2. Task 2 \
Verify the changes using the Container-CPU-Memory custom format with docker stats command \
``timeout 1 docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"``

3. Task 3 \
Perform load testing on the Nginx container you have created using Apache Benchmark
``apt update && apt install apache2-utils -y`` \
``docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx-limit-memory`` \
``ab -n 1000 -c 10 http://<YOUR-IP-ADDRESS-HERE>/``

----

### Scanning the Remote Host for Unauthenticated Docker API Access

> In its default configuration, Docker APIs are exposed through port 2375.

Let’s run an nmap scan from port range 1-5000 to see if there is a Docker service running on the prod-47pv40ou machine.

``docker run --rm uzyexe/nmap -p 1-5000 prod-47pv40ou``

``curl http://prod-47pv40ou:2375/version``

**TASKS**

1. List the images on the remote host prod-47pv40ou using curl command \
``curl http://prod-47pv40ou:2375/images/json``

2. List all containers on the remote host prod-47pv40ou using curl command \
``curl http://prod-47pv40ou:2375/images/json``

---

### Create and Restore a Snapshot of the Container for Further Analysis

we can use the following command to check the docker host information \
``docker -H prod-47pv40ou:2375 info``

Additional commands: \
``docker -H prod-47pv40ou:2375 ps``

if the running container is utilizing a private registry inaccessible to us, and only the internal organization has access to it, how can we download the image? One approach is to use the docker save command, which enables us to back up the container as a tar file. \
``docker -H prod-47pv40ou:2375 save -o django.tar hysnsec/django.nv``

In the previous step, we utilized the save option to back up the container, and now we will employ the load option to run the container from a tar file. \
``docker load < django.tar``

we can explore any information about the image, just by running the image. \
``docker run -d --name analysis -it hysnsec/django.nv``

let’s spawn a shell inside the container. \
``docker exec -it analysis bash``

``cat /etc/os-release``
``pwd``

---

### Identify a Container and Extract Sensitive Information

``apk add py3-pip git``

``pip3 install detect-secrets``

``detect-secrets scan .``

***TASKS***

1. Run detect-secret and disable specific plugins, such as KeywordDetector and AWSKeyDetector \
``docker exec analysis detect-secrets scan --disable-plugin KeywordDetector --disable-plugin AWSKeyDetector``
2. Re-run the previous task and save the output as detect-secret.json \
``docker exec analysis detect-secrets scan --disable-plugin KeywordDetector --disable-plugin AWSKeyDetector > detect-secret.json``

---

### Identifying Misconfigurations In Namespace, Capabilities, And Networking

***Capabilities:***
You can add or drop capabilities using appropriate flags when starting a container. For example, the --cap-add and --cap-drop Docker CLI flags helps in adding and dropping capabilities for a container. \
``docker run --cap-add NET_ADMIN nginx`` \
The command starts an nginx container with the NET_ADMIN capability added.


1. ***Identifying Namespace Misconfigurations*** \
Namespaces are critical for creating a distinct, isolated working environment for each container, providing separation of processes, networking, IPC, and other resources.

##### Case 1: Embedding Host Process Namespaces

``docker run -d --pid=host --name alp1 alpine sleep infinity`` \
The command ***docker run -d --pid=host --name alp1 alpine sleep infinity*** is vulnerable because it shares the PID namespace between the container and the host, breaking the isolation that containers are supposed to provide. This allows processes inside the container to view and potentially interact with all host processes, increasing the risk of information leakage and privilege escalation. An attacker with access to the container could manipulate host processes, send signals to them, or gather sensitive data about the system. As a result, this setup poses significant security risks, especially in untrusted or production environments, and should be avoided unless strictly necessary for specific use cases like debugging.

##### Case 2: Embedding Shared Mount Namespace
Creating a Docker container with the host’s mount namespace is a risky measure as it allows the container to share filesystems with the host. \
``docker run -dit --name alp2 -v /:/hostroot:rw alpine``

2. ***Identifying Capabilities Misconfigurations***
Misconfiguration of Linux Capabilities can cause potential security risks, so it’s crucial to ensure the configured capabilities are exactly what a container needs and nothing more.

#### Case 1: Unnecessary Root Level Capability
Unnecessary Root Level Capability refers to granting a container or process elevated privileges that it doesn’t need to perform its functions, similar to giving it root access.

Limit capabilities to the bare minimum required for a container to function.

``docker run -it --rm --cap-add=NET_ADMIN alpine``

> Container runtimes like docker for instance start a container with only required capabilities. The [needed capabilities](https://github.com/moby/moby/blob/master/oci/caps/defaults.go#L6-L19) for running a docker container does not include the NET_ADMIN capability.

#### Case 2: Preventing Unprivileged Root Capabilities
Let’s consider an Alpine Linux container running a service as root. We want to explicitly drop the CAP_DAC_OVERRIDE capability from the root user, effectively preventing the issue from reading certain files within the container. \
``docker run -it --rm --cap-drop CAP_DAC_OVERRIDE alpine``

3. Identifying Networking Misconfigurations

When all containers can communicate with each other without restriction, there is unrestricted network access, which may not be a desired setting in certain environments.

``docker network create -o "com.docker.network.bridge.enable_icc"="false" isolated_network``

> The whole command -o “com.docker.network.bridge.enable_icc”=”false” sets the Inter-Container Communication (ICC) to false in the network created by the docker which means that containers in the same network will not be able to communicate with each other automatically, unless explicitly linked using --link.
