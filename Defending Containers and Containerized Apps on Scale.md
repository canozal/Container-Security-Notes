# Defending Containers and Containerized Apps on Scale

### Image Security
Check if images and packages inside images are free from security vulnerabilities and follows best practices

1. Docker Best Practices (lint)
2. Vulnerability Scanning (clair)

### Three ways to reduce the size
1. Smaller Base Images
2. Distroless Image \
Distroless Docker images contain only your application and its runtime dependencies without the bloat (package managers, shells etc.,) \
• Improves security by reducing security issues \
• Smaller when compared to ubuntu and alpine, e.g., 2MB Debian image \
• Reduces supply chain attacks using cosign - https://github.com/sigstore/cosign 
3. Scratch Image \
Scratch Image is a minimal image in Docker that: \
• Does not contain anything \
• Is used to create other base images or other minimal images \
• Can't be downloaded or pulled, or tagged \
• Is used to create statically compiled images like golang, or c binaries.
```
FROM scratch
ADD file /
CMD ["/file"]
```

### Building Secure Container Images
1. Choose smaller base images like Alpine \
• Reduces unnecessary software \
• Improves performance. 
2.  Lint Dockerfile and follow best practices to catch any issues
3. Big images like Centos, Debian or Ubuntu can't be avoided for special cases like ELK

### Multi Stage Builds
• Multi stage builds uses more than one image to build docker images \
• The first images are have multiple layers with dependencies and other packages \
• The final image is lean because the final images uses the previously build images \

• In multi stage builds: 
1. There are more than one FROM statements
2. The first FROM statements are images with dependencies and other packages
3. The final FROM statement uses the previous image as a base to build a smaller docker image 

• Additional read: https://medium.com/capital-one-tech/multi-stage-builds-and-dockerfile-b5866d9e2f84
```
FROM node: 12.13.0-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx
EXPOSE 3000
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/build /usr/share/nginx/html
```

--- 

Docker-slim helps reduce images sizes up to 30 times smaller images \
• Allows you to reverse engineer Dockerfile \
• Profile an image to review how hefty an image is \
• Build a minified version of an image \
• Does not modify an existing image, creates a minified image with a .slim suffix

---

### Ensuring Dockerfile Best Practices
Hadolint is a Dockerfile linter that helps in checking for Docker Image best practices. 

Like many SAST tools it uses Abstract Syntax Trees (AST). to find common issues. It also uses ShellCheck to lint the Bash code inside RUN instructions. 

Very useful for local, testing and CI/CD use cases. You can easily install hadolint using native, docker and other installation methods. \
``docker run --rm -i hadolint/hadolint < Dockerfile``

https://github.com/hadolint/hadolint

---

### Clair Image Scanner

Clair is an open source project for the static analysis of vulnerabilities in application containers. 

Clair's analysis is broken in to three parts namely Indexing, Matching, and Notifications.

Clair indexes image layers first, and then matches the indexing with a list of known CVEs, finally uses a notifier to notify these vulnerabilities.

Clair can run in several modes. Indexer, matcher, notifier or combo mode. In combo mode, everything runs in a single OS process.

---

### Trivy Image Scanner

Trivy is a simple and comprehensive vulnerability scanner for containers and other artifacts, suitable for Cl.

Trivy detects vulnerabilities of OS packages (Alpine, RHEL, CentOS, etc.) and application dependencies (Bundler,
Composer, npm, yarn, etc.).

Trivy is easy to use. Just install the binary and you're ready to scan.
All you need to do for scanning is to specify a target such as an image name of the container.

---

### Secrets Endemic

Everyone has a secret, including infrastructure. We need secrets to connect to systems, services, to do authentication, authorization, etc.

CI/CD is at the center and hence has secrets for the systems mentioned above and is a go-to target for any determined attacker.

We must keep secrets away from version control systems and docker images.

---

### Two types of git secrets scanners

1. Regex based scanner \
Regular expressions based scanners look for known secrets patterns in the code 
- Great for catching known
- Can give lots of FPs
- Can create custom regex
- FPs can be reduced \
e.g., git-secrets, gitrob

2. Entropy based scanner \
Entropy looks for data which is random (lacks order) or predictability
- Can catch unknown
- Needs to be random
- Can't create custom rules
- Not always possible \
e.g., Trufflehog, repo-scanner

---

- Linting, Image Scanning, and Secret Scanning can all be in a single pipeline, or they can be parallized.
- Linting catches low hanging fruits while writing a dockerfile.
- Image scanning finds the insecure libraries/binaries.
- Linting takes 1-2 minutes. However, image scanners can take up to 4-5 minutes.
- Secret scanners could be added during build process in development

---

### Static Analysis using Hadolint
Let’s install the Hadolint tool on the system to perform static analysis of Dockerfiles. \
``wget https://github.com/hadolint/hadolint/releases/download/v2.12.0/hadolint-Linux-x86_64``

Let’s rename this file to make it easy for us to type it. \
``mv hadolint-Linux-x86_64 /usr/local/bin/hadolint``

To run Hadolint \
``hadolint Dockerfile``

---

### Scanning Docker for Vulnerabilities with Trivy

Let’s install trivy tool \
``wget https://github.com/aquasecurity/trivy/releases/download/v0.48.1/trivy_0.48.1_Linux-64bit.deb && dpkg -i trivy_*.deb``

Let’s execute the scanner to scan the python:3.4-alpine image \
``trivy image python:3.4-alpine``

To scan filesysystem \
``trivy filesystem .``

**TASKS**

1. Scan the python:3.6-alpine image and label the issues with a severity of CRITICAL as False Positives (FP). Additionally, save the output in JSON format at /trivy-output.json \
```
cat > .trivyignore << EOL
CVE-2022-37434
CVE-2022-22822
CVE-2022-22823
CVE-2022-22824
CVE-2022-23852
CVE-2022-25235
CVE-2022-25236
CVE-2022-25315
EOL
```
``trivy image -f json -o /trivy-output.json python:3.6-alpine``

2. Configure trivy so that it only returns a non-zero exit code when vulnerabilities are found. \
``trivy image -f json --exit-code 1 -o /trivy-output.json python:3.6-alpine``

---

### Embedding Trivy Scanning in GitLab CI

```
image: docker:20.10

services:
  - docker:dind

stages:
  - build
  - test
  - release
  - preprod
  - integration
  - prod

container_scanning:
  stage: build
  before_script:
   - echo $CI_REGISTRY_PASS | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
   - DOCKER_BUILDKIT=0 docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .   # Build the application into Docker image
   - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA         # Push the image into registry
   - docker run --rm -v /root/.cache/:/root/.cache/ -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/src aquasec/trivy image --exit-code 1 -f json -o /src/trivy-output.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    paths: [trivy-output.json]
    when: always

test:
  stage: test
  image: python:3.6
  before_script:
   - pip3 install --upgrade virtualenv
  script:
   - virtualenv env
   - source env/bin/activate
   - pip install -r requirements.txt
   - python manage.py test taskManager

integration:
  stage: integration
  script:
    - echo "This is an integration step"
    - exit 1
  allow_failure: true # Even if the job fails, continue to the next stages

prod:
  stage: prod
  script:
    - echo "This is a deploy step."
  when: manual # Continuous Delivery

```

We do not want to fail the builds/jobs/scan in DevSecOps Maturity Levels 1 and 2, as security tools spit a significant amount of false positives.

You can use the ``allow_failure`` tag to not fail the build even though the tool found issues

```
image: docker:20.10

services:
  - docker:dind

stages:
  - build
  - test
  - release
  - preprod
  - integration
  - prod

container_scanning:
  stage: build
  before_script:
   - echo $CI_REGISTRY_PASS | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
   - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .   # Build the application into Docker image
   - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA         # Push the image into registry
   - docker run --rm -v /root/.cache/:/root/.cache/ -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/src aquasec/trivy image --exit-code 1 -f json -o /src/trivy-output.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    paths: [trivy-output.json]
    when: always
  allow_failure: true

test:
  stage: test
  image: python:3.6
  before_script:
   - pip3 install --upgrade virtualenv
  script:
   - virtualenv env
   - source env/bin/activate
   - pip install -r requirements.txt
   - python manage.py test taskManager

integration:
  stage: integration
  script:
    - echo "This is an integration step"
    - exit 1
  allow_failure: true # Even if the job fails, continue to the next stages

prod:
  stage: prod
  script:
    - echo "This is a deploy step."
  when: manual # Continuous Delivery
```

---

### Build a Secure, Miniature Image With Distroless To Minimize Attack Footprint
Distroless is one of the image created by Google with security purposes such as minus the operating system (smaller than alpine, without shell, no package manager).

This build pattern uses two Docker images, the first image serves as a base image for building or compiling the application, and the second image is used to run it. This pattern was introduced in Docker 17.05 CE and is called multi-stage builds. Its purpose is to reduce the size of Docker images and remove unnecessary packages.
```
cat >Dockerfile<<EOF
FROM debian:11-slim AS build

COPY requirements.txt .
RUN apt-get update \
    && apt-get install --no-install-suggests --no-install-recommends --yes python3-venv gcc libpython3-dev \
    && python3 -m venv /venv \
    && /venv/bin/pip install --upgrade pip

FROM build AS build-venv
COPY requirements.txt /requirements.txt
RUN /venv/bin/pip install -r /requirements.txt

FROM gcr.io/distroless/python3-debian11

WORKDIR /app
COPY --from=build-venv /venv /venv
COPY . .
RUN /venv/bin/python3 manage.py migrate \
    && /venv/bin/python3 manage.py loaddata fixtures/*

EXPOSE 8000
ENTRYPOINT ["/venv/bin/python3", "manage.py", "runserver", "0.0.0.0:8000"]
EOF
```

---

### Docker Deamon Security 
Make sure Docker engine and host is configured properly, with special focus on namespaces, groups, capabilities

***Docker Bench Security:*** \
Docker bench script is used to check common best practices around deploying Docker containers in production.

Docker bench is a shell script and can be configured in a container for ease of use.

Due to its nature, you can use it on a local machine, CI/CD or production machine.

You can configure it to check for all the tests or a specific check.

https://github.com/docker/docker-bench-security

```
$ docker run -it --net host --pid host --userns host --cap-add audit_control \
-e DOCKER_ CONTENT_ TRUST=$DOCKER_CONTENT.
_ TRUST \
-v /var/lib:/var/lib \
-v /var/run/docker. sock: /var/run/docker. sock \
-v /usr/lib/systemd:/usr/lib/systemd \
-v / etc: /etc --label docker_ bench_security \ docker/docker-bench-security
```


***Dev-Sec Hardening Framework:*** \
Dev-Sec Project has lots of good compliance checks on configuring various daemons including SSH, Docker etc.,. \
For example, https://github.com/dev-sec/cis-docker-benchmark

***Daemon Security:***\
• Ensure SELinux/AppArmor is enabled by default \
• APl is not exposed \
• If exposed, uses TLS auth

---

### Minimize Docker Security Misconfigurations With CIS Compliance

In this scenario, you will learn how to ensure your Docker daemon configuration follows the security best practices using docker-bench.

The Docker Bench for Security is a script that checks for dozens of common best-practices around deploying Docker containers in production.

``wget https://github.com/aquasecurity/docker-bench/releases/download/v0.5.0/docker-bench_0.5.0_linux_amd64.deb``

``dpkg -i docker-bench_0.5.0_linux_amd64.deb``

``git clone https://github.com/aquasecurity/docker-bench.git``

``cd docker-bench``

After that, execute Docker Bench with the specified benchmark version:
``docker-bench --benchmark cis-1.6.0``

To identify configuration weakness in docker, we can execute the following command. \
``docker-bench``


