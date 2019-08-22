# Centralized Container Logging: Elastic stack (ELK) + Filebeat on Docker


This repository, modified from the [original repository][original-repo], is about creating a **centralized logging platform** for your Docker containers, using **ELK stack + Filebeat**, which are also running on Docker.

"ELK" is the acronym for three open source projects: Elasticsearch, Logstash, and Kibana. **Elasticsearch** is a search and analytics engine. **Logstash** is a server‑side data processing pipeline that ingests data from multiple sources simultaneously, transforms it, and then sends it to a "stash" like Elasticsearch. **Kibana** lets users visualize data with charts and graphs in Elasticsearch.

On top the ELK stack is **Filebeat**, a log shipper belonging to the Beats family — a group of lightweight shippers installed on hosts for shipping different kinds of data into the ELK Stack for analysis.

![ELK + Filebeat Image][elk-fb]

Based on the official Docker images from Elastic:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/master/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/master/docker)
* [Kibana](https://github.com/elastic/kibana/tree/master/src/dev/build/tasks/os_packages/docker_generator)
* [Filebeat](https://github.com/elastic/beats/tree/master/filebeat)

## Contents

1. [Requirements](#requirements)
   * [Host setup](#host-setup)
   * [SELinux](#selinux)
   * [Docker for Desktop](#docker-for-desktop)
     * [Windows](#windows)
     * [macOS](#macos)
2. [Usage](#usage)
   * [Bringing up the stack](#bringing-up-the-stack)
   * [Bringing up the beat](#bringing-up-the-beat) 
   * [Initial setup](#initial-setup)
     * [Setting up user authentication](#setting-up-user-authentication)
     * [Log in Kibana](#log-in-kibana)
     * [Default Kibana index pattern creation](#default-kibana-index-pattern-creation)
3. [Configuration](#configuration)
   * [How to configure Elasticsearch](#how-to-configure-elasticsearch)
   * [How to configure Kibana](#how-to-configure-kibana)
   * [How to configure Logstash](#how-to-configure-logstash)
   * [How to configure Filebeat](#how-to-configure-filebeat)
   * [How to disable paid features](#how-to-disable-paid-features)
   * [How to scale out the Elasticsearch cluster](#how-to-scale-out-the-elasticsearch-cluster)
4. [Storage](#storage)
   * [How to persist Elasticsearch data](#how-to-persist-elasticsearch-data)
5. [JVM tuning](#jvm-tuning)
   * [How to specify the amount of memory used by a service](#how-to-specify-the-amount-of-memory-used-by-a-service)
   * [How to enable a remote JMX connection to a service](#how-to-enable-a-remote-jmx-connection-to-a-service)
6. [Going further](#going-further)
   * [Using a newer stack version](#using-a-newer-stack-version)
   * [Swarm mode](#swarm-mode)
7. [Additional references](#additional-references)

## Requirements

### Host setup

* [Docker](https://www.docker.com/community-edition#/download) version **17.05+**
* [Docker Compose](https://docs.docker.com/compose/install/) version **1.6.0+**
* 1.5 GB of RAM

By default, the stack exposes the following ports:
* 5044: Logstash TCP input
* 9600: Logstash TCP input
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

> :information_source: Elasticsearch's [bootstrap checks][booststap-checks] were purposely disabled to facilitate the
> setup of the Elastic stack in development environments. For production setups, we recommend users to set up their host
> according to the instructions from the Elasticsearch documentation: [Important System Configuration][es-sys-config].

### SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux
into Permissive mode in order for docker-elk to start properly. For example on Redhat and CentOS, the following will
apply the proper context:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

### Docker for Desktop

#### Windows

Ensure the [Shared Drives][win-shareddrives] feature is enabled for the `C:` drive.

#### macOS

The default Docker for Mac configuration allows mounting files from `/Users/`, `/Volumes/`, `/private/`, and `/tmp`
exclusively. Make sure the repository is cloned in one of those locations or follow the instructions from the
[documentation][mac-mounts] to add more locations.

## Usage

### Bringing up the stack

Clone this repository to the server on which the stack will be running.

To start the **ELK stack** using Docker Compose:

```console
$ cd elk_stack
$ docker-compose up
```

### Bringing up the beat

Clone this repository to the server from which you like to ship logs to ELK stack.

First, in ```filebeat/filebeat.yml``` file, you **MUST** replace `<logstash server IP>` with the IP address your ELK stack is running on.

Then, run the Filebeat as docker:

```console
$ cd filebeat
$ docker-compose up
```

By default, all container logs of the server on which the Filbeat container is running will be shipped to Logstash container for processing.

You can also run all services in the background (detached mode) by adding the `-d` flag to the above command.

> :information_source: You must run `docker-compose build` first whenever you switch branch or update a base image.

If you are starting the stack for the very first time, please read the section below attentively.

### Initial setup

#### Setting up user authentication

> :information_source: Refer to [How to disable paid features](#how-to-disable-paid-features) to disable authentication.

The stack is pre-configured with the following **privileged** bootstrap user:

* user: *elastic*
* password: *changeme*

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security. Passwords for these users must be initialized:

```console
$ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```

Passwords for all 6 built-in users will be randomly generated. Take note of them and replace the `elastic` username with
`kibana` and `logstash_system` inside the Kibana and Logstash configuration files respectively. See the
[Configuration](#configuration) section below.

> :information_source: Do not use the `logstash_system` user inside the Logstash *pipeline* file, it does not have
> sufficient permissions to create indices. Follow the instructions at [Configuring Security in Logstash][ls-security]
> to create a user with suitable roles.

Restart Kibana and Logstash to apply the passwords you just wrote to the configuration files.

```console
$ docker-compose restart kibana logstash
```

> :information_source: Learn more about the security of the Elastic stack at [Tutorial: Getting started with
> security][sec-tutorial].

#### Log in Kibana

Give Kibana about a minute to initialize, then access the Kibana web UI by hitting
[http://localhost:5601](http://localhost:5601) or http://\<docker host IP\>:5601 with a web browser and use the following default credentials to log in:

* user: *elastic*
* password: *\<your generated elastic password>*


#### Default Kibana index pattern creation

When Kibana launches for the first time, it is not configured with any index pattern.

##### Via the Kibana web UI

> :information_source: You need to inject data into Logstash before being able to configure a Logstash index pattern via
the Kibana web UI. Then all you have to do is hit the *Create* button.

Refer to [Connect Kibana with Elasticsearch][connect-kibana] for detailed instructions about the index pattern
configuration.

##### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.2.1' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

The created pattern will automatically be marked as the default index pattern as soon as the Kibana UI is opened for the first time.


## Configuration

> :information_source: Configuration is not dynamically reloaded, you will need to restart individual components after
any configuration change.

### How to configure Elasticsearch

The Elasticsearch configuration is stored in [`elk_stack/elasticsearch/config/elasticsearch.yml`][config-es].

You can also specify the options you want to override by setting environment variables inside the Compose file:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Please refer to the following documentation page for more details about how to configure Elasticsearch inside Docker
containers: [Install Elasticsearch with Docker][es-docker].

### How to configure Kibana

The Kibana default configuration is stored in [`elk_stack/kibana/config/kibana.yml`][config-kbn].

It is also possible to map the entire `config` directory instead of a single file.

Please refer to the following documentation page for more details about how to configure Kibana inside Docker
containers: [Running Kibana on Docker][kbn-docker].

### How to configure Logstash

The Logstash configuration is stored in [`elk_stack/logstash/config/logstash.yml`][config-ls].

It is also possible to map the entire `config` directory instead of a single file, however you must be aware that
Logstash will be expecting a [`log4j2.properties`][log4j-props] file for its own logging.

Please refer to the following documentation page for more details about how to configure Logstash inside Docker
containers: [Configuring Logstash for Docker][ls-docker].

### How to configure Filebeat

The Filebeat configuration is stored in [`filebeat/config/filebeat.yml`][config-fb]

By default, it takes all the containers' logs on its host as the input: 

```yml
filebeat.inputs:
   - type: container
     paths:
        - '/var/lib/docker/containers/*/*.log'
```

And output to Logstash:

```yml
output.logstash:
   hosts: ["<logstash server IP>:5044"]
```

Please refer to the following documentation page for more details about how to configure Filebeat to take container logs as the input: [Container Input][fb-container-input], or how to run Filebeat as a container: [Running Filebeat on Docker][fb-docker].

### How to disable paid features

Switch the value of Elasticsearch's `xpack.license.self_generated.type` option from `trial` to `basic` (see [License
settings][trial-license]).

### How to scale out the Elasticsearch cluster

Follow the instructions from the Wiki: [Scaling out Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

## Storage

### How to persist Elasticsearch data

The data stored in Elasticsearch will be persisted after container reboot but not after container removal.

In order to persist Elasticsearch data even after removing the Elasticsearch container, you'll have to mount a volume on
your Docker host. Update the `elasticsearch` service declaration to:

```yml
elasticsearch:

  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

This will store Elasticsearch data inside `/path/to/storage`.

> :information_source: (Linux users) Beware that the Elasticsearch process runs as the [unprivileged `elasticsearch`
user][esuser] is used within the Elasticsearch image, therefore the mounted data directory must be writable by the uid
`1000`.


## JVM tuning

### How to specify the amount of memory used by a service

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If you want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### How to enable a remote JMX connection to a service

As for the Java Heap memory (see above), you can specify JVM options to enable JMX and map the JMX port on the Docker
host.

Update the `{ES,LS}_JAVA_OPTS` environment variable with the following content (I've mapped the JMX service on the port
18080, you can change that). Do not forget to update the `-Djava.rmi.server.hostname` option with the IP address of your
Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Going further

### Using a newer stack version

To use a different Elastic Stack version than the one currently available in the repository, simply change the version
number inside the `.env` file, and rebuild the stack with:

```console
$ docker-compose build
$ docker-compose up
```

> :information_source: Always pay attention to the [upgrade instructions][upgrade] for each individual component before
performing a stack upgrade.

### Swarm mode

Experimental support for Docker [Swarm mode][swarm-mode] is provided in the form of a `docker-stack.yml` file, which can
be deployed in an existing Swarm cluster using the following command:

```console
$ docker stack deploy -c docker-stack.yml elk
```

If all components get deployed without any error, the following command will show 3 running services:

```console
$ docker stack services elk
```

> :information_source: To scale Elasticsearch in Swarm mode, configure *zen* to use the DNS name `tasks.elasticsearch`
instead of `elasticsearch`.

## Additional references
* [ELK-on-Docker tutorial blog](https://logz.io/blog/docker-logging/)
* [Filebeat tutorial](https://logz.io/blog/filebeat-tutorial/)
* [Discussion: Collecting Logfiles of Docker containers with Filebeat running as docker container](https://discuss.elastic.co/t/collecting-logfiles-of-docker-containers-with-filebeat-running-as-docker-container/77548)


[original-repo]: https://github.com/deviantony/docker-elk
[elk-fb]: https://logz.io/wp-content/uploads/2017/06/relationship-between-filebeat-and-logstash.png

[elk-stack]: https://www.elastic.co/elk-stack
[stack-features]: https://www.elastic.co/products/stack
[paid-features]: https://www.elastic.co/subscriptions
[trial-license]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html

[booststap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

[win-shareddrives]: https://docs.docker.com/docker-for-windows/#shared-drives
[mac-mounts]: https://docs.docker.com/docker-for-mac/osxfs/

[builtin-users]: https://www.elastic.co/guide/en/x-pack/current/setting-up-authentication.html#built-in-users
[ls-security]: https://www.elastic.co/guide/en/logstash/current/ls-security.html
[sec-tutorial]: https://www.elastic.co/guide/en/elastic-stack-overview/current/security-getting-started.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html

[config-es]: ./elk_stack/elasticsearch/config/elasticsearch.yml
[config-kbn]: ./elk_stack/kibana/config/kibana.yml
[config-ls]: ./elk_stack/logstash/config/logstash.yml
[config-fb]: ./filebeat/config/filebeat.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html
[fb-docker]: https://www.elastic.co/guide/en/beats/filebeat/current/running-on-docker.html
[fb-container-input]: https://www.elastic.co/guide/en/beats/filebeat/master/filebeat-input-container.html

[log4j-props]: https://github.com/elastic/logstash/tree/7.3/docker/data/logstash/config
[esuser]: https://github.com/elastic/elasticsearch/blob/7.3/distribution/docker/src/docker/Dockerfile#L18-L19

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html

[swarm-mode]: https://docs.docker.com/engine/swarm/
