# Docker Host and Container Monitoring

A monitoring solution for Docker hosts and containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor), [NodeExporter](https://github.com/prometheus/node_exporter) and alerting with [AlertManager](https://github.com/prometheus/alertmanager).
I forked this project to bring some interesting advantages to the monitoring solution from [Stefan Prodans dockprom](https://github.com/stefanprodan/dockprom).
An other great solution is the monitoring and logging suite from [Wilhelm Uschtrin](https://github.com/uschtwill/docker_monitoring_logging_alerting).
He provides a very good tool set that is very well structured.

I also was inspired by other docker monitor projects and tutorials:
- [A docker-compose stack for Prometheus monitoring](https://github.com/vegasbrianc/prometheus/)
- [Monitoring in Docker Stacks - It’s that easy with Prometheus!](https://medium.com/@soumyadipde/monitoring-in-docker-stacks-its-that-easy-with-prometheus-5d71c1042443)
- [Prometheus Monitoring](https://kjanshair.github.io/2018/02/20/prometheus-monitoring/)

Docker logs are collected by [fluentd](https://www.fluentd.org/) and provided with [elasticsearch](https://www.elastic.co/de/).
For this I build the container image with the elasticsearch plugin. You can visit the [project site on Github](https://github.com/andz-dev/fluentdelasticsearch).

Because this is a huge docker sevice I recommend to split the services into smaller ones which are runs on servers as a service

***If you're looking for the Docker Swarm version please go to [stefanprodan/swarmprom](https://github.com/stefanprodan/swarmprom)***

## Prerequisites

- Docker Engine >= 1.13
- Docker Compose >= 1.11

## Install

Clone this repository on your Docker host, go into the `dockmon` directory and run `docker-compose up -d`:

```bash
git clone https://github.com/andz-dev/dockmon.git
cd dockmon

docker-compose up -d
```

## Containers

- Prometheus (metrics database) `http://<host-ip>:9090`
- Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
- AlertManager (alerts management) `http://<host-ip>:9093`
- Grafana (visualize metrics) `http://<host-ip>:3000`
- NodeExporter (host metrics collector)
- cAdvisor (containers metrics collector)
- Traeffik (reverse proxy and basic auth provider for prometheus and alertmanager)
- fluentd with elasticsearch plugin for the Docker logs
- Elasticsearch for the fluentd collected logs

## Networks

## Volumes

## Grafana

[Grafana](https://grafana.com/) is the open platform for beautiful analytics and monitoring and supports various services for time series analytics.
In this scenario we use Grafana to display all necessary stats about our host, the Docker container and other services.
Also the logging from elasticsearch with fluentd will be displayed.

### Setup

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables on compose up.
Instead of using the environments inside the compose file create a new `config` file.
The config file can be added directly in grafana part like this

```yml
grafana:
  image: grafana/grafana:5.2.4
  env_file:
    - config

```

and the config file format should have this content

```yml
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
# Set to true if you want to allow new users to register
GF_USERS_ALLOW_SIGN_UP=false
```

If you want to change the password, you have to remove this entry, otherwise the change will not take effect

```yml
- grafana_data:/var/lib/grafana
```

### Settings

Grafana is preconfigured with some dashboards and configs for Prometheus, Elasticsearch and so on as the default data source. The directory [datasources](./grafana/provisioning/datasources) contains all in this project available datasources.

Because we use the new grafana version provisioning is provided for our docker container also. This means that it is enough to volume mount the provisioning directory in the compose file.

```yaml
volumes:
  - ./grafana/provisioning/:/etc/grafana/provisioning/
```

The JSON files for the dashboards and datasources should be "grafana compatible" - so export your dashboard directly from grafana UI.

Using the provisioning doesn't need an extra `setup` script to load the data via the API.
If you want to use this way, have a look to grafana's [HTTP API Reference](http://docs.grafana.org/http_api/) documentation.

#### Data Sources

To show all data sources open your browser and enter [http://{IP}:3000/api/datasources](http://{IP}:3000/api/datasources).

You can export all datasources with the following command:

```bash
mkdir data_sources && curl -s "http://{IP}:3000/api/datasources" -u admin:admin|jq -c -M '.[]'|split -l 1 - data_sources/
```

For more information visit the [Simple export/import of Data Sources in Grafana](https://rmoff.net/2017/08/08/simple-export-import-of-data-sources-in-grafana/)

![Grafana data sources API](./screens/Grafana_Datasources_API.png)

#### Dashboards

To show all dashboards with inside the `general` directory in grafana you can use the following query. [http://{IP}:3000/api/search?folderIds=0&query=&starred=false](http://{IP}:3000/api/search?folderIds=0&query=&starred=false)

![Grafana dashboard query](./screens/Grafana_Dashboard_Query.png)

To get detail information for a dashboard use the uid, for example:
[http://{IP}:3000/api/dashboards/uid/ceETkLLmk (Docker Container Dashboard)](http://{IP}:3000/api/dashboards/uid/ceETkLLmk)

To export the dashboards open it inside Grafana and click on the `Share` button.

![Share Button](./screens/Grafana_Share_Button.png)

On the new window switch to the `Export` tab and click on `Save to file`.

![Grafana Dashboard Export](./screens/Grafana_Dashboard_Export.png)

Repeat the steps for each dashboard you want to export. [Read the "Export and Import" manual in the official docs for more information](http://docs.grafana.org/reference/export_import/).

### Dashboard Views

***Docker Host Dashboard***

![Host](./screens/Grafana_Docker_Host.png)

The Docker Host Dashboard shows key metrics for monitoring the resource usage of your server:

- Server uptime, CPU idle percent, number of CPU cores, available memory, swap and storage
- System load average graph, running and blocked by IO processes graph, interrupts graph
- CPU usage graph by mode (guest, idle, iowait, irq, nice, softirq, steal, system, user)
- Memory usage graph by distribution (used, free, buffers, cached)
- IO usage graph (read Bps, read Bps and IO time)
- Network usage graph by device (inbound Bps, Outbound Bps)
- Swap usage and activity graphs

For storage and particularly Free Storage graph, you have to specify the `fstype` in grafana graph request.
You can find it in `grafana/dashboards/docker_host.json`, at line 480 :

```json
"expr": "sum(node_filesystem_free_bytes{fstype=\"btrfs\"})",
```

I work on BTRFS, so i need to change `aufs` to `btrfs`.

You can find right value for your system in Prometheus `http://{IP}:9090` launching this request:

      node_filesystem_free_bytes

***Docker Containers Dashboard***

![Containers](./screens/Grafana_Docker_Containers.png)

The Docker Containers Dashboard shows key metrics for monitoring running containers:

- Total containers CPU load, memory and storage usage
- Running containers graph, system load graph, IO usage graph
- Container CPU usage graph
- Container memory usage graph
- Container cached memory usage graph
- Container network inbound usage graph
- Container network outbound usage graph

Note that this dashboard doesn't show the containers that are part of the monitoring stack.

***Monitor Services Dashboard***

![Monitor Services](./screens/Grafana_Prometheus.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

- Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
- Container CPU usage graph
- Container memory usage graph
- Prometheus chunks to persist and persistence urgency graphs
- Prometheus chunks ops and checkpoint duration graphs
- Prometheus samples ingested rate, target scrapes and scrape duration graphs
- Prometheus HTTP requests graph
- Prometheus alerts graph

## Define alerts

I've setup three alerts configuration files:

- Monitoring services alerts [targets.rules](./prometheus/targets.rules)
- Docker Host alerts [host.rules](./prometheus/host.rules)
- Docker Containers alerts [containers.rules](./prometheus/containers.rules)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```bash
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
ALERT monitor_service_down
  IF up == 0
  FOR 30s
  LABELS { severity = "critical" }
  ANNOTATIONS {
      summary = "Monitor service non-operational",
      description = "{{ $labels.instance }} service is down.",
  }
```

***Docker Host alerts***

Trigger an alert if the Docker host CPU is under high load for more than 30 seconds:

```yaml
ALERT high_cpu_load
  IF node_load1 > 1.5
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Server under high load",
      description = "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.",
  }
```

Modify the load threshold based on your CPU cores.

Trigger an alert if the Docker host memory is almost full:

```yaml
ALERT high_memory_load
  IF (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) ) / sum(node_memory_MemTotal_bytes) * 100 > 85
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Server memory is almost full",
      description = "Docker host memory usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.",
  }
```

Trigger an alert if the Docker host storage is almost full:

```yaml
ALERT hight_storage_load
  IF (node_filesystem_size_bytes{fstype="aufs"} - node_filesystem_free_bytes{fstype="aufs"}) / node_filesystem_size_bytes{fstype="aufs"}  * 100 > 85
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Server storage is almost full",
      description = "Docker host storage usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.",
  }
```

***Docker Containers alerts (Jenkins example)***

Trigger an alert if a container is down for more than 30 seconds:

```yaml
ALERT jenkins_down
  IF absent(container_memory_usage_bytes{name="jenkins"})
  FOR 30s
  LABELS { severity = "critical" }
  ANNOTATIONS {
    summary= "Jenkins down",
    description= "Jenkins container is down for more than 30 seconds."
  }
```

Trigger an alert if a container is using more than 10% of total CPU cores for more than 30 seconds:

```yaml
 ALERT jenkins_high_cpu
  IF sum(rate(container_cpu_usage_seconds_total{name="jenkins"}[1m])) / count(node_cpu_seconds_total{mode="system"}) * 100 > 10
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary= "Jenkins high CPU usage",
    description= "Jenkins CPU usage is {{ humanize $value}}%."
  }
```

Trigger an alert if a container is using more than 1,2GB of RAM for more than 30 seconds:

```yaml
ALERT jenkins_high_memory
  IF sum(container_memory_usage_bytes{name="jenkins"}) > 1200000000
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Jenkins high memory usage",
      description = "Jenkins memory consumption is at {{ humanize $value}}.",
  }
```

## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://{IP}:9093`.

The notification receivers can be configured in [alertmanager/config.yml](./alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](./screens/Slack_Notifications.png)

## Sending metrics to the Pushgateway

The [pushgateway](https://github.com/prometheus/pushgateway) is used to collect data from batch jobs or from services.

To push data, simply execute:

```bash
echo "some_metric 3.14" | curl --data-binary @- http://user:password@localhost:9091/metrics/job/some_job
```

Please replace the `user:password` part with your user and password set in the initial configuration (default: `admin:admin`).

## Updating Grafana to v5.2.2

[In Grafana versions >= 5.1 the id of the grafana user has been changed](http://docs.grafana.org/installation/docker/#migration-from-a-previous-version-of-the-docker-container-to-5-1-or-later). Unfortunately this means that files created prior to 5.1 won’t have the correct permissions for later versions.

| Version |   User  | User ID |
|:-------:|:-------:|:-------:|
|  < 5.1  | grafana |   104   |
|  \>= 5.1 | grafana |   472   |

There are two possible solutions to this problem.
- Change ownership from 104 to 472
- Start the upgraded container as user 104

### Specifying a user in docker-compose.yml

To change ownership of the files run your grafana container as `root` and modify the permissions.

First perform a `docker-compose down` then modify your `docker-compose.yml` to include the `user: root` option:

```yaml
  grafana:
    image: grafana/grafana:5.3.4
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana/
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    user: root
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: on-failure
    expose:
      - 3000
    networks:
      - monitor-backend
```

Perform a `docker-compose up -d` and then issue the following commands:

```bash
docker exec -it --user root grafana bash

# in the container you just started:
chown -R root:root /etc/grafana && \
chmod -R a+r /etc/grafana && \
chown -R grafana:grafana /var/lib/grafana && \
chown -R grafana:grafana /usr/share/grafana
```

To run the grafana container as `user: 104` change your `docker-compose.yml` like such:

```yaml
  grafana:
    image: grafana/grafana:5.3.4
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana/
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    user: "104"
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: on-failure
    expose:
      - 3000
    networks:
      - monitor-backend
```

## Logging

To extend this monitoring stack to collect and show container logs you can use fluentd with elasticsearch and bind each service to the logging stack.
Catching the container logs from this stack needs the `logging` parameter in your compose file if you want to use this way.

***Example***

```yaml
depends_on:
    - fluentd
  logging:
    driver: fluentd
    options:
      fluentd-address: "{IP}:24224"
      tag: "docker.{{.ID}}"
```

Have a look inside the [fluentd elasticsearch project]() for more information.

## Reverse Proxy

## Architecture