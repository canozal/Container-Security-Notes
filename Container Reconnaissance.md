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

Docker has several network drivers that we connect that we can use when creating the containers, inculude following
1. Bridge \
• Default network driver \
• Containers on the same bridged network can speak to each other \
• Containers on a bridged network can't connect to containers on other bridge \
• Access to external network is allowed through NAT 
2. Host
3. Macvlan
4. None





