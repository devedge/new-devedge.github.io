---
layout: post
title: "metrics - Netdata/Prometheus/Grafana stack"
description: "Setup, configuration, and containerization/orchestration of a monitoring stack"
date: 2021-01-18
tags: [metrics, netdata, prometheus, grafana, docker, docker-compose]
---

After accumulating a number of servers/networking gear, it got too much for me to navigate to each of their respective dashboards to view their metrics. Then, an idea spawned: I'll set up a monitoring stack, and aggregate all the metrics I want into a single dashboard.

This is the end result, and what I'll be walking through in this post:

![Metrics NPG Dashboard](/assets/images/metrics-npg-dash.png){: .center-image}

---

### Glossary

- [Intro](#intro)
- [Architecture](#architecture)
- [Prometheus]()
- [Netdata (ivymike)]()
- [Graphite (freenas)]()
- [SNMP (EdgeRouter)]()
- [Grafana]()

## Intro

Before diving into the nitty-gritty, there are four overarching questions that need to be answered:

1. Where/what metrics endpoints will be monitored?
2. What will ingest the metrics? Can it support all the endpoints?
3. What will be used to visualize the metrics?
4. Where/how will the monitoring service(s) run?

To answer these questions:

##### 1. Where/what metrics endpoints will be monitored?

This depends on what systems/servers you have. In my case, these are the systems I want to monitor, and what technologies they use to expose metrics:

- **Arch Linux Server (ivymike)** - Netdata installed on the host
- **FreeNAS Server (freenas)** - natively exposes Graphite metrics
- **Ubiquiti EdgeRouter** - natively exposes metrics over SNMP
- **CyberUPS uninterruptible power supply** - connected to Arch Linux server, is included in Netdata metrics

##### 2. What will ingest the metrics? Can it support all the endpoints?

Here, there are 2 major industry-accepted technologies: [InfluxDB](https://www.influxdata.com/) and [Prometheus](https://prometheus.io/). Both are more than adequate to monitor all endpoints, so it comes down to personal choice.

After doing some research, this [StackOverflow](https://stackoverflow.com/a/38406973) post and [the following one](https://stackoverflow.com/a/40767676) swayed my decision towards Prometheus. It seems like InfluxDB has trouble scaling with a massive amount of servers, so Prometheus would be more useful industry experience.

##### 3. What will be used to visualize the metrics?

This field is largely dominated by one tool - [Grafana](https://grafana.com/) - there doesn't seem to be any other serious competition. This is also easy to integrate with either data ingest tools, so it's an easy choice.

##### 4. Where/how will the monitoring service(s) run?

The final question has 2 answers - the entire monitoring stack will be running in Docker containers, and orchestrated using docker-compose. These containers will run on the Arch Linux server, an Intel NUC called `ivymike`.

## Architecture

![Metrics Architecture](/assets/images/metrics-npg-architecture.png){: .center-image}

The architecture is fairly straightforward, aside from the 2 'exporter' containers.

prometheus
graphite-exporter
snmp-exporter
grafana

how to explain??

intro that the architecture will be explained in the following sections
explain that each section will have a 'docker config' part and a 'configs' part?

prometheus
add docker-compose config
introduce yml, explain that it will be used for the next 2 containers

`docker-compose.yml`

```yaml
version: "3"
services:
  prometheus:
    build: prometheus/
    restart: unless-stopped
    command:
      # Default CLI options to start Prometheus, taken from the container. Docker-Compose's
      # `command:` directive overwrites any CLI args, so if we want to use the directive, all
      # the args need to be redefined.
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --web.console.libraries=/usr/share/prometheus/console_libraries
      - --web.console.templates=/usr/share/prometheus/consoles

      # Max metrics retention time set to one year
      - --storage.tsdb.retention.time=1y
    ports:
      - 9090:9090
    volumes:
      - prometheus-config:/fragments/
      - "/mnt/nas-share/Prometheus:/prometheus:rw" # Metrics are stored on NAS
      - "/etc/localtime:/etc/localtime:ro"
# ...
```

`Dockerfile`

```Dockerfile
FROM prom/prometheus:v2.22.0
ADD prometheus.yml /etc/prometheus/prometheus.yml
```

`prometheus.yml`

```yaml
global:
  # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  scrape_interval: 15s
  # Evaluate rules every 15 seconds. The default is every 1 minute.
  evaluation_interval: 15s

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["0.0.0.0:9090"]
# ...
```

netdata
don't cover installing it
show how it's added to the prometheus yml

graphite
freenas already exposes it, simply configure it to point to prometheus IP

snmp
add docker-compose config
...explanation about how to get it working with EdgeRouter?? point to blog.

grafana
log in with admin creds, add prometheus as data source
configure dashboard will all the metrics prometheus collects
