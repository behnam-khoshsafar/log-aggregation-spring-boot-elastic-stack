* Docker containers emit logs to the stdout and stderr output streams. Because containers are stateless, the logs are
  stored on the Docker host in JSON files by default.
* The container’s logging driver can access these streams and send the logs to a file, a log collector running on the
  host, or a log management service endpoint.
* Docker host is the machine that docker engine is installed.
* Unlike non-containerized applications that write logs into files, containers write their logs to the standard output
  and standard error stream.

## JSON File logging driver

* By default, Docker captures the standard output (and standard error) of all your containers, and writes them in files
  using the JSON format. The JSON format annotates each line with its origin (stdout or stderr) and its timestamp. Each
  log file contains information about only one container.

```
{"log":"Log line is here\n","stream":"stdout","time":"2019-01-01T11:11:11.111111111Z"}
```

* You can find these JSON log files in the `/var/lib/docker/containers/` directory on a Linux Docker host. Here’s how
  you can access them:

```
/var/lib/docker/containers/<container id>/<container id>-json.log
```

* That’s where logging comes into play. You can collect the logs with a log aggregator and store them in a place where
  they’ll be available forever. It’s dangerous to keep logs on the Docker host because they can build up over time and
  eat into your disk space. That’s why you should use a central location for your logs and enable log rotation for your
  Docker containers.

## two type of logs

* logs from the Dockerized application inside the container
* logs from the host servers, which consist of the system logs, as well as the Docker Daemon logs which are usually
  located in /var/log or a subdirectory within this directory.

## Docker Logging Strategies and Best Practices

### Logging via Application

* This technique means that the application inside the containers handles its own logging using a logging framework. For
  example, a Java app could use a Log4j2 to format and send the logs from the app to a remote centralized location
  skipping both Docker and the OS.
* On the plus side, this approach gives developers the most control over the logging event. However, it creates extra
  load on the application process. If the logging framework is limited to the container itself, considering the
  transient nature of containers, any logs stored in the container’s filesystem will be wiped out if the container is
  terminated or shut down.
* To keep your data, you’ll have to either configure persistent storage or forward logs to a remote destination like a
  log management solution such as Elastic Stack or Sematext Cloud. Furthermore, application-based logging becomes
  difficult when deploying multiple identical containers, since you would need to find a way to tell which log belongs
  to which container.

### Logging Using Data Volumes

* As we’ve mentioned above, one way to work around containers being stateless when logging is to use data volumes.
* With this approach you create a directory inside your container that links to a directory on the host machine where
  long-term or commonly-shared data will be stored regardless of what happens to your container. Now, you can make
  copies, perform backups, and access logs from other containers.
* You can also share volume across multiple containers. But on the downside, using data volumes make it difficult to
  move the containers to different hosts without any loss of data.

### Logging Using the Docker Logging Driver

* Another option to logging when working with Docker, is to use logging drivers. Unlike data volumes, the Docker logging
  driver reads data directly from the container’s stdout and stderr output. The default configuration writes logs to a
  file on the host machine, but changing the logging driver will allow you to forward events to syslog, gelf, journald,
  and other endpoints.
* Since containers will no longer need to write to and read from log files, you’ll likely notice improvements in terms
  of performance. However, there are a few disadvantages of using this approach as well: Docker log commands work only
  with the json-file log driver; the log driver has limited functionality, allowing only log shipping without parsing;
  and containers shut down when the TCP server becomes unreachable.

### Logging Using a Dedicated Logging Container

* Another solution is to have a container dedicated solely to logging and collecting logs, which makes it a better fit
  for the microservices architecture. The main advantage of this approach is that it doesn’t depend on a host machine.
  Instead, the dedicated logging container allows you to manage log files within the Docker environment. It will
  automatically aggregate logs from other containers, monitor, analyze, and store or forward them to a central location.

### Logging Using the Sidecar Approach

* For larger and more complex deployments, using a sidecar is among the most popular approaches to logging microservices
  architectures.
* Similarly to the dedicated container solution, it uses logging containers. The difference is that this time, each
  application container has its own dedicated container, allowing you to customize each app’s logging solution. The
  first container saves log files to a volume which are then tagged and shipped by the logging container to a
  third-party log management solution.

## Get Started with Docker Container Logs

* When you’re using Docker, you work with two different types of logs: daemon logs and container logs.

### What Are Docker Container Logs?

* Docker container logs are generated by the Docker containers. They need to be collected directly from the containers.
  Any messages that a container sends to stdout or stderr is logged then passed on to a logging driver that forwards
  them to a remote destination of your choosing.
* Here are a few basic Docker commands to help you get started with Docker logs and metrics:
    * Show container logs: `docker logs containerName`
    * Show only new logs: `docker logs -f containerName`
    * Show CPU and memory usage: `docker stats`
    * Show CPU and memory usage for specific containers: `docker stats containerName1 containerName2`
    * Show running processes in a container: `docker top containerName`
    * Show Docker events: `docker events`
    * Show storage usage: `docker system df`
* Watching logs in the console is nice for development and debugging, however in production you want to store the logs
  in a central location for search, analysis, troubleshooting and alerting.

#### What Is a Logging Driver?

* Logging drivers are Docker’s mechanisms for gathering data from running containers and services to make it available
  for analysis. Whenever a new container is created, Docker automatically provides the json-file log driver if no other
  log driver option has been specified. At the same time, it allows you to implement and use logging driver plugins if
  you would like to integrate other logging tools.
* Here’s an example of how to run a container with a custom logging driver, in this case syslog:

```
docker run -–log-driver syslog –-log-opt syslog-address=udp://syslog-server:514 \
alpine echo hello world
```

### How to Configure the Docker Logging Driver?

When it comes to configuring the logging driver, you have two options:

* setup a default logging driver for all containers
* specify a logging driver for each container

* In the first case, the default logging driver is a JSON file, but, as mentioned above, you have many other options
  such as logagent, syslog, fluentd, journald, splunk, etc. You can switch to another logging driver by editing the
  Docker configuration file and changing the log-driver parameter, or using your preferred log shipper.
* Alternatively, you can choose to configure a logging driver on a per-container basis. As Docker provides a default
  logging driver when you start a new container, you need to specify the new driver from the very beginning by using the
  -log-driver and -log-opt parameters.

### Where Are Docker Logs Stored By Default?

The logging driver enables you to choose how and where to ship your data. The default logging driver as I mentioned
above is a JSON file located on the local disk of your Docker host:

```
/var/lib/docker/containers/[container-id]/[container-id]-json.log.
```

Have in mind, though, that when you use another logging driver than json-file or journald you will not find any log
files on your disk. Docker will send the logs over the network without storing any local copies. This is risky if you
ever have to deal with network issues.

In some cases Docker might even stop your container, when the logging driver fails to ship the logs. This issue might
happen depending on what delivery mode you are using.

### Where Are Delivery Modes?

#### Direct/Blocking

* Blocking is Docker’s default mode. It will interrupt the application each time it needs to deliver a message to the
  driver.
* It makes sure all messages are sent to the driver, but can introduce latency in the performance of your application.
  if the logging driver is busy, the container delays the application’s other tasks until it has delivered the message.
* Depending on the logging driver you use, the latency differs. The default json-file driver writes logs very quickly
  since it writes to the local filesystem, so it’s unlikely to block and cause latency. However, log drivers that need
  to open a connection to a remote server can block for longer periods and cause noticeable latency.
* That’s why we suggest you use the json-file driver and blocking mode with a dedicated logging container to get the
  most of your log management setup. Luckily it’s the default log driver setup, so you don’t need to configure anything
  in the /etc/docker/daemon.json file.

#### Non-blocking

* In non-blocking mode, a container first writes its logs to an in-memory ring buffer, where they’re stored until the
  logging driver is available to process them. Even if the driver is busy, the container can immediately hand off
  application output to the ring buffer and resume executing the application. This ensures that a high volume of logging
  activity won’t affect the performance of the application running in the container.
* Non-blocking mode does not guarantee that the logging driver will log all the events. If the buffer runs out of space,
  buffered logs will be deleted before they are sent. You can use the max-buffer-size option to set the amount of RAM
  used by the ring buffer. The default value for max-buffer-size is 1 MB, but if you have more RAM available, increasing
  the buffer size can increase the reliability of your container’s logging.

### Logging Driver Options

* logagent: A general purpose log shipper. The Logagent Docker image is pre-configured for log collection on container
  platforms. Logagent collects not only logs, it also adds meta-data such as image name, container id, container name,
  Swarm service or Kubernetes meta-data to all logs. Plus it handles multiline logs and can parse container logs.
* syslog: Ships log data to a syslog server. This is a popular option for logging applications.
* journald: Sends container logs to the systemd journal.
* fluentd: Sends log messages to the Fluentd collector as structured data.
* elf: Writes container logs to a Graylog Extended Log Format (GELF) endpoint such as Graylog or Logstash.
* awslogs: Sends log messages to AWS CloudWatch Logs.
* splunk: Writes log messages to Splunk using HTTP Event Collector (HEC).
* cplogs: Ships log data to Google Cloud Platform (GCP) Logging.
* logentries: Writes container logs to Rapid7 Logentries.
* etwlogs: Writes log messages as Event Tracing for Windows (ETW) events, thus only available on Windows platforms.

### Use the json-file Log Driver With a Log Shipper Container

The most reliable and convenient way of log collection is to use the json-file driver and set up a log shipper to ship
the logs.

### How to Work With Docker Container Logs Using the docker logs Command?

* `docker logs <container_id>`
* `docker logs <container_id> --timestamps`
* `docker logs <container_id> --since (or --until) YYYY-MM-DD`
* `docker logs <container_id> --tail N`
* `docker logs <container_id> --follow`
* `docker logs <container_id> | grep pattern`
* `docker logs <container_id> | grep -i error`
* `docker-compose logs` -> This will display the logs from all services in the application defined in the Docker Compose
  configuration file.

### What About Docker Daemon Logs

* Docker daemon logs are generated by the Docker platform and located on the host. Depending on the host operating
  system, daemon logs are written to the system’s logging service or to a log file.

On that note, the Docker daemon logs two types of events:

* Events generated by the Docker service itself
* Commands sent to the daemon through Docker’s Remote API
