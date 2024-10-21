# Security Monitoring of Containers

Vulnerability Management is one of the most crucial tasks in any Information security program. This helps organizations in managing their risks, be compliant with state, national, and international standards and regulations, prove to customers that security is being worked and improved upon.

The effectiveness of a security program can be gauged using the maturity of vulnerability management in the organization. Many compliance activities mandate certain service level agreements for fixing security issues.

### Vulnerability management - Workflow

1. Find \
Use different tools to find security issues

2. Aggregate \
Different tools generate different data, so consolidation is the key

3. Manage \
Issues can then be analyzed for False positives

4. Fix \
File issues into Defect Trackers for fixing

5. Improve \
Create metrics/ dashboards for stakeholders

---

### Docker Events

Docker has an API that listen for real time events like creating a container, deleting an image or a container, etc.,
The events are generated for the following components in
Docker
1. Containers
2. Images
3. Volumes
4. Networking

It only returns 1000 log events however you can use filters to see only the required information.

```
# Listen to docker events in a terminal
$ docker events

# In another terminal window, run an nginx container to review the events
$ docker run -d nginx

# Docker events can be filtered by timestamps
$ docker events --since '2021-12-15T11: 35:42.1683309762'

# Docker events can be filtered for a particular image
$ docker events --filter 'image alpine'

# Docker events can be formatted for json
$ docker events --format '({json .}}'

```

---

### Docker Logs
Docker events allow us to stream information about docker events however it doesn't show you the application logs.

We can use the docker logs command to fetch the application logs and using some agent, sent the logs to a storage, and alerting system like SIEM.

```
# Run an nginx container named nix in the background
$ docker run -d --name ngx nginx

# Review the logs of the ngx container
$ docker logs ngx

# docker-compose also supports logs
$ docker-compose logs
$
# docker-compose logs for a specific service named webapp
docker-compose logs webapp

# docker-compose logs for a specific service name db
$ docker-compose logs db

# Inspect the location of application logs
inspect
$ docker container inspect --format='{{.LogPath}}' ngx
```

```
# Run an alpine container in the background that prints date in a while 100p
$ docker run -dt --name alp1 alpine sh -c "while true; do date; sleep 1; done;"

# Review the logs with docker logs
$ docker logs alpl

# Review the logs with docker logs again
$ docker logs alpl
```

### Docker Logging - LAB
Logs play a significant role for both operational and security teams. These records offer the organization to get all the information (events, error, activity, etc.) about their application and also enables continuous monitoring by integrating logs with centralized log management system or a Security Information and Event Management (SIEM) solution permits continuous monitoring. There are a lot of tools that we can use to ship our logs into centralized monitoring system. You can also configure your monitoring system with alarm or notification if your system detects anomalies activity due to security issues. Enhancing visibility is particularly important for containerized applications.

In this exercise you will learn how to configure Docker with logging functionality.

we need to update our Docker configuration to save the log into a file (JSON format).

Logging is enabled by default in Docker but you can also add log-driver settings, that provided several options for us to save the log output into different mechanism i.e JSON (human-readable format), syslog, fluentd, or you can ship the logs to monitoring system.

In a running production machine, it’s possible to view logs of containerised applications by using the ***docker logs*** command. Additionally, the location of the logs can be checked in the ***/var/lib/docker/containers/*** directory. In this directory, you can detect a JSON log labelled ***-json.log***.

``cat /var/lib/docker/containers/*/*-json.log | jq``

---

### Auditing Docker using AuditD - LAB
Audit is essential security and compliance measures in the environment that we used and we can see any activities like uncover anomalies, track what changes were made, who did the changes etc..

In Docker, the audit feature is only available for the Docker Team or Business subscriptions and can be accessed through the Docker Hub UI. So, how can we audit our docker daemon in the system? We can use a tool named auditd to monitor docker command activity in our system.

auditd is pre-installed in the lab setup, you just need to follow the steps on how to use it with the docker in this exercise.

To verify the running status of the auditd service, please input the following command.

``systemctl status auditd``

Because the auditd is running at the boot time and logging the activities as we can see at /var/log/audit/audit.log i.e,

``cat /var/log/audit/audit.log``

we can use ausearch tool to filter the log instead of using cat and grep command.

``ausearch -m USER_CMD``

The generated output may appear like an encoded string. However, there’s no need to decode these strings individually. we can add -i argument to decode the string.

``ausearch -m USER_CMD -i``

If you want to search the activity of a process with the id 2170, then you’d use the below command.

``ausearch -p 1722``

You can also check the activity of a particular user with the -ua option.
To check the activity of the root user type the below command.

``ausearch -ua root -i``

### Configure Docker Audit Log
We will use ***auditctl*** command to add Docker into our audit rules using watch feature

Execute the following command to add docker into audit log using -w option.
```
auditctl -w /usr/bin/docker -p rwxa -k docker-daemon
```
We’ve set the filter key as docker-daemon, so we can use ausearch easily to check the activity log.

``ausearch -k docker-daemon -i``

---

### Sysdig Falco – Runtime Protection and Monitoring
Runtime security refers to the process of securing applications and systems while they are running, as opposed to securing them at build time or during deployment.
```
Runtime security involves actively monitoring the applications and systems for vulnerabilities, anomalies, and attacks, and taking immediate action to mitigate and contain the threats as they arise.
```
Runtime security can be seen as the last line of defense in an organization’s security strategy, as it is focused on protecting against threats that may have evaded other security measures such as firewalls, intrusion detection and prevention systems, and vulnerability scanners. Some benefits of runtime security in a container environment include:

- Enhanced visibility into container activity
- Real-time threat detection
- Faster and automated response to security incidents
- Process isolation and control
- Compliance and governance enforcement

### Install Falco
```
Falco is an open-source project created by Sysdig for security monitoring, detecting anomalous activity in your container workloads such as Docker and Kubernetes. It has its own security policy to detect malicious activity in real time, and can be integrated with SIEMs such as Splunk, ELK, Grafana, and more. This tool is similar to auditd but more powerful, using kernel modules like eBPF, easy to configure, with policy customization based on scenarios, and many more features.
```
To install Falco.
```
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
bash -c 'cat << EOF > /etc/apt/sources.list.d/falcosecurity.list
deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main
EOF'
apt update -y
```
``apt install -y dkms make linux-headers-$(uname -r) dialog`` \
``apt install -y falco=0.38.0`` \
We have installed Falco in our system, let’s start the service. \
``systemctl status falco-modern-bpf``


### Configuring Falco Logs to Use JSON Format
By default, Falco stores logs in the syslog file. However, we will configure it to store logs in JSON format for easier parsing with tools if necessary.

Let’s update the configuration file at /etc/falco/falco.yaml to make the logging format as JSON.

```
cat > /etc/falco/falco.yaml<<EOF
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/rules.d

plugins:
  - name: json
    library_path: libjson.so

load_plugins: []

watch_config_files: true

time_format_iso_8601: false

json_output: true
json_include_output_property: false
json_include_tags_property: false

log_stderr: true
log_syslog: false
log_level: info

libs_logger:
  enabled: false
  severity: debug

priority: warning
buffered_outputs: false

syscall_event_drops:
  threshold: .1
  actions:
    - log
    - alert
  rate: .03333
  max_burst: 1

syscall_event_timeouts:
  max_consecutives: 1000

syscall_buf_size_preset: 4
output_timeout: 2000
outputs:
  rate: 0
  max_burst: 1000

syslog_output:
  enabled: false

file_output:
  enabled: true
  filename: /var/log/falco.json

stdout_output:
  enabled: true

webserver:
  enabled: true
  threadiness: 0
  listen_port: 8765
  k8s_healthz_endpoint: /healthz
  ssl_enabled: false
  ssl_certificate: /etc/falco/falco.pem

program_output:
  enabled: false

http_output:
  enabled: false

grpc:
  enabled: false

grpc_output:
  enabled: false

metadata_download:
  max_mb: 100
  chunk_wait_us: 1000
  watch_freq_sec: 1
EOF
```
After updating the configuration, we need to restart the service. \
``systemctl restart falco-modern-bpf``

Now, let’s use the previous command we tried to generate some malicious activity. \
``docker run -it --rm falcosecurity/event-generator run syscall``

we will check the log from Falco to see the results. \
``cat /var/log/falco.json``

---

### Tracee - Runtime Security

```
Tracee is a security and visibility tool that operates in real time, aiding you in comprehending your system’s and applications’ functioning. By utilizing eBPF technology, Tracee gains access to your system and unveils such information in the form of events for your consumption. These events extend from actual system activity occurrences to complex security events identifying potentially questionable behavioral trends.
```
To install Tracee by pulling the Tracee docker image using the following command. \
``docker pull aquasec/tracee:0.19.0``

To run Tracee to see the options provided by Tracee.
```
docker run --name tracee -it --rm \
  --pid=host --cgroupns=host --privileged \
  -v /etc/os-release:/etc/os-release-host:ro \
  -v /var/run:/var/run:ro \
  aquasec/tracee:0.19.0 --help
```
Tracee needs a configuration file to set up runtime monitoring.

Let’s create a file named /etc/tracee.yaml to instruct tracee that we want the output to be stored in a JSON format, at /tmp/tracee-output.json.

```
cat > /etc/tracee.yaml <<EOF
output:
    json:
        files:
            - "/tmp/tracee-output.json"
EOF
```
After creating the configuration file, we need to run the Tracee container in the background.

```
docker run --name tracee -dit \
    --pid=host --cgroupns=host --privileged \
    -v /etc/os-release:/etc/os-release-host:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    -v /etc/tracee.yaml:/etc/tracee.yaml \
    -v /tmp:/tmp aquasec/tracee:0.19.0 --config /etc/tracee.yaml --scope container
```

 