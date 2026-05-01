---
title: "📊 Infrastructure Monitoring with Prometheus and Grafana"
description: "A comprehensive guide to setting up Prometheus and Grafana for monitoring your Docker-based infrastructure, services, and applications."
date: 2026-5-2
permalink: /posts/2026/5/PrometheusGrafanaMonitoring/
categories: [Guides, 🛜 Networking]
tags: [Prometheus 📊, Grafana 📈, Monitoring 🔍, Docker 🐳, DevOps ⚙️]
pin: false
published: true
---

# Infrastructure Monitoring with Prometheus and Grafana

## Introduction

Managing multiple services without visibility into their performance is like flying blind. As your infrastructure grows with more Docker services, Kubernetes clusters, and applications, you need a robust monitoring solution. Prometheus and Grafana are industry-standard tools that provide real-time insights into your system's health and performance.

Prometheus and Grafana are industry-standard tools that provide exactly that: insight into system health, performance trends, and potential issues before they become problems.

In this guide, you will set up a complete monitoring stack that enables you to:
- Collect metrics from all your services and hosts
- Store time-series data efficiently
- Create dashboards for real-time visualization
- Set up alerts for critical issues

This complements my previous guide on [Traefik](/posts/2025/2/traefik/), as both are essential components of this infrastructure. Other reverse proxies are of course also compatible with this monitoring stack, but Traefik's built-in metrics and seamless integration make it a great choice for this example.

## Context and considerations

Prometheus and Grafana provide a strong, flexible foundation for monitoring without locking you into a specific vendor or ecosystem. This makes them particularly suitable for IT professionals who want full control over their stack while keeping the option to scale later.

This guide assumes a single Docker host or homelab-style environment, where Traefik is already used for secure external access. The same architecture scales to production environments, but typically involves:

- Multiple nodes
- Dedicated storage for long-term metrics
- More advanced access control and segmentation

To keep things practical, start by monitoring only the most critical signals:

- Host CPU, memory, and disk usage
- Container status and restarts

These metrics provide a quick and reliable indication of system health and are often enough to detect early signs of instability.

## Security Considerations
A monitoring stack is one of the most information-rich components in your environment. It exposes:

- Service names and internal structure
- Network ports and communication paths
- System capacity and usage patterns

If this information is exposed, it becomes significantly easier to map and target your infrastructure.

For that reason, monitoring should be treated as sensitive internal tooling, not a publicly accessible dashboard. At a minimum:

- Restrict access using authentication (e.g., via Traefik forward auth or SSO)
- Avoid exposing exporters directly to the internet
- Use internal networks for metric collection (as shown in this guide)

A well-secured monitoring stack gives you visibility without increasing your attack surface.

### What each container does

- **Prometheus** collects and stores metrics from the services you want to monitor.
- **Grafana** reads those metrics and turns them into dashboards and alerts.
- **Node Exporter** exposes host-level metrics such as CPU, memory, disk, and network usage.
- **cAdvisor** provides container-level metrics so you can see how each Docker container is performing.
- **Docker State Exporter** reports Docker daemon and container state information.

### Storage and retention

Prometheus needs enough disk space to keep the metrics you care about, and retention directly affects how much storage it uses. A longer retention period gives you more history for troubleshooting and trend analysis, but it also means higher disk usage over time. For a small Docker host, start with a modest retention window and increase it only if you know you need the extra history.

## Prerequisites

Before you start, you need:
- Docker and Docker Compose installed
- A Docker network (or you can create one for the monitoring stack)
- A basic understanding of metrics and time-series data
- 2GB+ of free disk space for Prometheus data storage

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│           Monitored Services                        │
│  (Docker containers, Node exporters, etc.)          │
└────────────────┬────────────────────────────────────┘
                 │ (metrics on :9090, :9100, etc.)
┌────────────────▼────────────────────────────────────┐
│         Prometheus Server (Scraper)                 │
│    Stores time-series data in local storage         │
└────────────────┬────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────┐
│      Grafana (Visualization & Alerting)             │
│       Queries Prometheus, creates dashboards        │
└─────────────────────────────────────────────────────┘
```

## Step-by-Step Setup

### Step 1: Create the Docker Compose Setup

Create a `docker-compose.yml` file for your monitoring stack:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v3
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    networks:
      - traefik # Expose this demo through Traefik
      - monitoring_internal
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
    labels:
      - "traefik.enable=true" # You should not expose Prometheus to the internet in production, but for this demo we will do it through Traefik. Make sure to secure it with authentication and TLS!
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.example.com`)"
      - "traefik.http.routers.prometheus.entrypoints=websecure"
      - "traefik.http.services.prometheus.loadbalancer.server.port=9090"
      - "traefik.http.routers.prometheus.service=prometheus"
      - "traefik.http.routers.prometheus.tls.certresolver=myresolver"

  grafana:
    image: grafana/grafana:13.1
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin # Change me!
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
    networks:
      - traefik # Exposed through Traefik
      - monitoring_internal
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.grafana.rule=Host(`grafana.example.com`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
      - "traefik.http.routers.grafana.service=grafana"
      - "traefik.http.routers.grafana.tls.certresolver=myresolver"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    restart: unless-stopped
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      - monitoring_internal
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --output-document=- http://localhost:9100/metrics | grep -q node_"]
      interval: 30s
      timeout: 10s
      retries: 3

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.55.1
    container_name: cadvisor
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring_internal
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3

  docker-state-exporter:
    image: karugaru/docker_state_exporter
    container_name: docker-state-exporter
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Be aware that this gives the exporter access to the Docker daemon, so only use it if you trust the image and understand the security implications.
    networks:
      - monitoring_internal
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --output-document=- http://localhost:8080/metrics | grep -q docker"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  prometheus_data:
  grafana_data:

networks:
  traefik:
    external: true

  monitoring_internal:
    internal: true # Be aware that this network is not accessible from outside, but also does not have internet access.
```

**Be aware:** All services that you want to expose to the outside via Traefik will need the network attached to the container. I recommend separating this from the internal services like above.

### Step 2: Configure Prometheus

Create a `prometheus.yml` file in the same directory:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'docker-monitor'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'docker-state-exporter'
    scrape_interval: 1m
    static_configs:
      - targets: ['docker-state-exporter:8080']

  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8082']

  # Add your other services here
  # Example for a custom application exporting metrics on port 8000:
  # - job_name: 'my_app'
  #   static_configs:
  #     - targets: ['my_app:8000']
```

### Step 3: Set Up Grafana Provisioning

Create the directory structure:

```bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
```

Create `grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

### Step 4: Start the Stack

```bash
docker-compose up -d
```

Verify all services are running:

```bash
docker-compose ps
```

### Step 5: Access the Services

- **Prometheus**: https://prometheus.example.com (exposed via Traefik)
- **Grafana**: https://grafana.example.com (exposed via Traefik. Username: `admin`, password: `admin`)

### Step 6: Create Your First Dashboard in Grafana

1. Log into Grafana
2. Change the default password (Settings → Admin → Change Password)
3. Add Prometheus as a data source (Configuration → Data Sources)
4. Create a new dashboard (Create → Dashboard)
5. Add panels with queries like:
   - `node_cpu_seconds_total` - CPU usage
   - `node_memory_MemAvailable_bytes` - Available memory
   - `container_memory_usage_bytes` - Container memory
   - `container_cpu_usage_seconds_total` - Container CPU

### Step 7: Monitor Your Docker Services

To add monitoring to your existing Docker services, add labels to their containers:

```yaml
services:
  my_service:
    image: my_image:latest
    ports:
      - "8000:8000"
    expose:
      - 8000
    environment:
      - METRICS_PORT=8000
    networks:
      - traefik
```

Then add a scrape job in `prometheus.yml`:

```yaml
  - job_name: 'my_service'
    static_configs:
      - targets: ['my_service:8000']
```

## Setting Up Alerts

Create alert rules in a new file `prometheus/rules.yml`:

```yaml
groups:
  - name: alerts
    rules:
      - alert: HighCPUUsage
        expr: node_cpu_seconds_total > 0.8
        for: 5m
        annotations:
          summary: "High CPU usage detected"

      - alert: HighMemoryUsage
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.2
        for: 5m
        annotations:
          summary: "Low available memory"
```

Update `prometheus.yml` to include the rules file:

```yaml
rule_files:
  - 'rules.yml'
```

## Challenges & Learnings

🔧 **Storage Planning**: Prometheus can consume significant disk space over time. Plan your retention policy based on your needs. The default is 15 days; adjust with `--storage.tsdb.retention.time=7d` if needed.

📊 **Metric Cardinality**: Be careful with high-cardinality metrics (many unique label combinations). This can impact performance. Use relabeling to drop unnecessary labels in `prometheus.yml`.

🎯 **Alert Fatigue**: Start with a few critical alerts and gradually add more. Too many alerts lead to alert fatigue and important issues get missed.

🔐 **Security**: Put Prometheus and Grafana behind a reverse proxy (like Traefik) and enable authentication. Never expose them directly to the internet without security measures.

## Troubleshooting

If a target is not showing up in Prometheus, first check whether the container is running, attached to the correct network, and listening on the port you configured. The Prometheus targets page is usually the fastest way to see whether the scrape is failing or whether the service itself is unreachable.

If Grafana cannot reach Prometheus, confirm the datasource URL and make sure the Grafana container can resolve the Prometheus service name inside the Docker network. If a dashboard looks empty, the issue is often either a bad query, a missing metric name, or a scrape job that has not been collecting data long enough yet.

When something still looks off, check the container logs and verify the labels, ports, and network names in the Compose file. Small configuration mistakes are common in monitoring stacks, and they are usually easier to spot there than in the dashboards themselves.

## Next Steps

- Integrate with your existing Traefik setup by monitoring it
- Set up alerting through Slack, PagerDuty, or email
- Create custom exporters for application-specific metrics
- Implement long-term storage with Thanos or Cortex
- Explore advanced Grafana features like template variables and annotations

## Conclusion

With Prometheus and Grafana, you now have complete visibility into your infrastructure. This monitoring foundation will help you identify issues before they become critical, optimize resource usage, and maintain reliability across your services.

💡 Want to learn more about monitoring, infrastructure optimization, or need help integrating this with your existing setup? Feel free to reach out!
