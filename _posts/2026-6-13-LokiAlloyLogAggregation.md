---
title: "📜 Log aggregation for Docker with Loki and Grafana Alloy"
description: "A practical guide to centralizing the logs of all your Docker services with Grafana Loki and Alloy, so you can search, filter, and alert on logs in the same Grafana you already use for metrics."
date: 2026-6-13
permalink: /posts/2026/6/LokiAlloy/
categories: [Guides, 📊 Monitoring]
tags: [Loki 📜, Grafana 📈, Alloy 🔭, Docker 🐳, Monitoring 🔍, DevOps ⚙️]
pin: false
published: true
---

# Log aggregation for Docker with Loki and Grafana Alloy

## Introduction

In the [monitoring guide](/posts/2026/5/PrometheusGrafanaMonitoring/) we set up Prometheus and Grafana, so by now you know *that* something is wrong: a container is restarting, error rates are climbing, a disk is filling up. What metrics cannot tell you is *why*. For that you need the logs, and right now they are scattered across `docker logs` on one or more hosts, gone forever when a container is recreated.

The fix is log aggregation: ship every container's logs to one central place, keep them for a configurable period, and search them from the same Grafana dashboards you already have. In the Grafana ecosystem that central place is [Loki](https://grafana.com/oss/loki/), and the agent that ships logs to it is [Grafana Alloy](https://grafana.com/oss/alloy-opentelemetry-collector/).

If you expected Promtail here: Promtail reached end of life on March 2, 2026. Alloy is its official successor, and since we are building this from scratch, we get to skip the migration entirely and start on the right foot. More on this in the Challenges section.

What makes Loki special compared to something like Elasticsearch is what it does *not* do: it does not index the content of your logs, only a small set of labels per log stream (like the container name). That makes it dramatically lighter to run; the whole stack in this guide idles at a few hundred MB of RAM, which is exactly what you want next to everything else on a homelab Docker host.

## How it works

Three components, one of which you already have:

```
┌────────────────────────── Docker host ─────────────────────────────────┐
│                                                                        │
│  🐳 traefik   🐳 authelia   🐳 grafana   🐳 ...                      │
│      │            │             │          │                           │
│      └────────────┴──────┬──────┴──────────┘                           │
│                          │ container logs (Docker API)                 │
│                          ▼                                             │
│                    ┌──────────┐  push   ┌────────┐  query  ┌─────────┐ │
│                    │  Alloy   │ ──────► │  Loki  │ ◄────── │ Grafana │ │
│                    └──────────┘         └────────┘         └─────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

Alloy discovers every running container through the Docker socket, attaches labels (most importantly the container name), and pushes the log lines to Loki. Loki stores them, compacted and with retention applied. Grafana queries Loki with LogQL, a query language that will feel instantly familiar if you have written any PromQL.

## Prerequisites

Before you start, you need:

- A Docker host with Docker Compose and the `monitoring` setup from my [Prometheus and Grafana guide](/posts/2026/5/PrometheusGrafanaMonitoring/), or at least a running Grafana on a shared Docker network
- Basic familiarity with Grafana

## Step-by-Step Setup

### Step 1: Create the Loki configuration

Create a directory for the stack and inside it a `loki-config.yml`:

```yaml
auth_enabled: false # Single tenant; fine on a private network

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1 # Single node
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-04-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 744h # Keep logs for 31 days

# Retention does NOT work without this block - see Challenges & Learnings
compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
```

This is the simplest sane Loki: single node, filesystem storage, modern TSDB index, and 31 days of retention. Note that `auth_enabled: false` means anyone who can reach port 3100 can read and write logs; we will keep that port strictly on the internal Docker network.

### Step 2: Create the Alloy configuration

Alloy is configured as a pipeline of components that feed into each other. Create `config.alloy`:

```alloy
// Discover every running container via the Docker socket
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

// Docker reports names as "/traefik"; strip the slash into a clean label
discovery.relabel "containers" {
  targets = []

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }
}

// Collect the logs of every discovered container
loki.source.docker "containers" {
  host          = "unix:///var/run/docker.sock"
  targets       = discovery.docker.containers.targets
  relabel_rules = discovery.relabel.containers.rules
  forward_to    = [loki.write.local.receiver]
}

// Ship everything to Loki
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

Read it bottom-up and the pipeline is obvious: a writer pointing at Loki, a Docker log source feeding that writer, and a relabel step that turns `/traefik` into a `container="traefik"` label. New containers are picked up automatically; there is nothing to configure when you add services later.

### Step 3: Create the Docker Compose setup

Add both services to a `docker-compose.yml` on the same network as Grafana:

```yaml
services:
  loki:
    image: grafana/loki:3.6.0 # Pin the version; check for the newest 3.x
    container_name: loki
    restart: unless-stopped
    command: -config.file=/etc/loki/config.yml
    volumes:
      - ./loki-config.yml:/etc/loki/config.yml
      - loki-data:/loki # Named volume avoids permission issues (Loki runs as uid 10001)
    networks:
      - monitoring
    # No ports section! Loki stays internal; Grafana reaches it over the Docker network

  alloy:
    image: grafana/alloy:v1.9.2 # Pin the version; check for the newest release
    container_name: alloy
    restart: unless-stopped
    command:
      - run
      - /etc/alloy/config.alloy
      - --storage.path=/var/lib/alloy/data
      - --server.http.listen-addr=0.0.0.0:12345 # Debug UI, internal only
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy
      - /var/run/docker.sock:/var/run/docker.sock:ro # Needed for discovery + log collection
      - alloy-data:/var/lib/alloy/data
    networks:
      - monitoring

volumes:
  loki-data:
  alloy-data:

networks:
  monitoring:
    external: true
```

The stack expects a shared external network called `monitoring` that Grafana is also attached to. Create it once:

```bash
docker network create monitoring
```

Then attach Grafana to it: in the monitoring stack from the [previous guide](/posts/2026/5/PrometheusGrafanaMonitoring/), add the network to the Grafana service (keeping its existing ones) and declare it as external at the bottom of that file too:

```yaml
  grafana:
    # ...existing configuration...
    networks:
      - traefik
      - monitoring_internal
      - monitoring # New: shared network with Loki
```

Recreate Grafana so the change takes effect, then start the new stack:

```bash
docker-compose up -d
```

### Step 4: Add Loki as a Grafana datasource

In Grafana, go to **Connections → Data sources → Add data source → Loki** and set the URL to `http://loki:3100`. Click **Save & test**; you should see a green confirmation.

That is the entire integration. Open **Explore**, select the Loki datasource, and pick a container label. Your logs are there, live, searchable, from every container at once.

### Step 5: Learn just enough LogQL

LogQL works in two stages: select streams with labels, then filter and parse the lines. A few queries that cover 90% of homelab usage:

```
# Everything from one container
{container="traefik"}

# Only the errors
{container="traefik"} |= "error"

# Search across ALL containers at once - the killer feature
{container=~".+"} |= "panic"

# Rate of failed login attempts on the Authelia portal
rate({container="authelia"} |= "Unsuccessful" [1m])
```

That last one is a metric *derived from logs*, which means you can graph it, put it on the dashboard from the monitoring guide, and alert on it. Someone hammering the login portal from the [Authelia guide](/posts/2026/6/AutheliaTraefik/) now shows up as a spike in a panel.

### Step 6: Keep the host disk safe

One thing Loki does not change: Docker itself still writes every container's logs to JSON files on the host, and by default those grow forever. Now that the logs live in Loki anyway, cap the local copies in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Restart the Docker daemon, and note that the limits apply to newly created containers. Recreate your stacks at a convenient moment.

## Challenges & Learnings

⚰️ **Promtail is dead, long live Alloy**: For years, every Loki tutorial started with Promtail. It was deprecated in early 2025 and reached end of life on March 2, 2026, with all development moving to Alloy, Grafana's OpenTelemetry-based collector. If you are following an older guide, you are setting up an unsupported agent. The Alloy configuration language takes an hour to get used to, but the component pipeline model is genuinely nicer, and the same agent can later collect metrics and traces too. If you have an existing Promtail setup, `alloy convert` migrates the config for you.

🏷️ **Labels are not fields**: The single most important Loki concept. Every unique combination of labels creates a separate stream, and every stream costs memory and index space. Container name as a label: perfect, you have a few dozen. Request path or user ID as a label: catastrophic, you have created millions of streams and Loki will let you know. Keep labels low-cardinality and find everything else with line filters and parsers at query time; that is the entire design philosophy.

🧹 **Retention is opt-in, twice**: Setting `retention_period` alone does nothing; deletion is performed by the compactor, which must be explicitly enabled with `retention_enabled: true`, which in turn refuses to start without `delete_request_store`. Miss any of the three and Loki happily keeps every log forever until the disk is full. Ask me how I know.

🐳 **The socket is the easy 90%**: Reading container logs through the Docker socket means zero per-service configuration, but it only captures what containers write to stdout/stderr. Services that log to files inside the container, or things running outside Docker entirely (the host's journal, Proxmox), need additional Alloy components like `local.file_match` or `loki.source.journal`. Start with the socket, add the rest when you actually need it.

## Troubleshooting

If no logs appear in Grafana, open the Alloy debug UI (temporarily expose port 12345 or curl it from another container on the network). It shows every component, its health, and live throughput; an unhealthy `loki.source.docker` almost always means the Docker socket is not mounted or not readable.

If Loki restarts in a loop, it is a configuration problem and the container logs will name the exact key. The classic one on bind mounts is a permission error on `/loki`, because Loki runs as uid 10001; the named volume in this guide sidesteps that, but `chown -R 10001:10001` fixes it if you prefer bind mounts.

If queries return "too many entries" or time out, you are selecting too much and filtering too little. Narrow the time range and put the cheapest filter (`|=` text match) before parsers like `| json`.

If you see "entry too far behind" errors after restoring or replaying old logs, Loki is rejecting out-of-order writes for a stream. For homelab purposes: let it be, the live tail is fine, and Loki accepts slightly old data within its ingestion window.

## Next Steps

- Define [Loki alerting rules](https://grafana.com/docs/loki/latest/alert/) so a flood of errors pages you the same way a metric threshold does
- Collect the host's systemd journal with `loki.source.journal`, and your Proxmox nodes' logs while you are at it
- Switch Traefik's access log to JSON format and build a dashboard of status codes and response times per router, parsed straight from the logs
- Put the Alloy debug UI behind Traefik with the [Authelia middleware](/posts/2026/6/AutheliaTraefik/) instead of leaving it internal-only
- When a second host joins the party, run one Alloy per host, all pushing to the same central Loki

## Conclusion

The monitoring stack from the [previous guide](/posts/2026/5/PrometheusGrafanaMonitoring/) told you that something broke; with Loki and Alloy attached to the same Grafana, you can now also see why, across every container, going back 31 days, with one query. Metrics for the what, logs for the why, and both behind the same [Authelia](/posts/2026/6/AutheliaTraefik/) login.

💡 Want to learn more about observability for self-hosted setups, or how this scales up to production log volumes? Feel free to reach out!
