![Status](https://img.shields.io/badge/status-active-success)
![Smoke Test](https://img.shields.io/github/actions/workflow/status/ArthurPinheiroMachado/homelab-macbook/observability-smoke-test.yaml?label=smoke%20test)
![Compose Validation](https://img.shields.io/github/actions/workflow/status/ArthurPinheiroMachado/homelab-macbook/compose-validation.yaml?label=compose%20validation)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana)
![Loki](https://img.shields.io/badge/Loki-899CFF?logo=grafana)

# Homelab Monitoring Platform

A production-inspired monitoring platform built to practice Site Reliability Engineering (SRE), DevOps, Infrastructure as Code, CI/CD and Observability concepts.

This project simulates how modern engineering teams manage monitoring infrastructure through version control, automated validation pipelines and GitOps-style deployments.

---

## Table of Contents

- [About](#about)
- [Server Specs](#server-specs)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Service Details](#service-details)
- [Alerting Rules](#alerting-rules)
- [Project Structure](#project-structure)
- [CI/CD](#cicd)
- [Roadmap](#roadmap)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## About

This repository contains the complete configuration for my personal homelab monitoring environment. The primary goal is to monitor the health and performance of a MacBook used as a home server — tracking CPU, memory, disk, and service availability over time.

**What this project demonstrates:**

- **Containerization** — Docker Compose with multi-network isolation (`frontend`/`backend`)
- **Metrics Collection** — Prometheus + Node Exporter for system and service metrics
- **Log Aggregation** — Loki + Promtail for centralized logging from system and Docker containers
- **Alerting** — Prometheus rules evaluated against metric thresholds, routed through Alertmanager to email
- **Visualization** — Grafana dashboards consuming data from both Prometheus and Loki
- **CI/CD** — GitHub Actions validating every config file and running smoke tests on PRs
- **Infrastructure as Code** — All service definitions, configs, and alert rules are version-controlled

---

## Server Specs

| Component | Spec |
|-----------|------|
| **Model** | MacBook Pro 13" Retina (A1502) |
| **CPU** | Intel I5-4278U @ 2.60 GHz |
| **RAM** | 8 GB DDR3 |
| **Storage** | 120 GB SSD |
| **OS** | Ubuntu 24.04.4 LTS |
| **Purpose** | Home server, monitoring, development |

---

## Architecture

```
                        ┌──────────────────────┐
                        │    Node Exporter      │
                        │  (system metrics)     │
                        │  :9100                │
                        └──────────┬───────────┘
                                   │ /metrics
                                   ▼
┌──────────────────────────────────────────────────────┐
│                      Prometheus                       │
│             ┌─────────────────────────────┐           │
│             │  scrape_interval: 15s       │           │
│             │  rule_files: rules/*.yaml   │           │
│             │  :9090                      │           │
│             └──────────┬──────────────────┘           │
└────────────────────────┼──────────────────────────────┘
                         │
          ┌──────────────┼──────────────────┐
          ▼              ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    Grafana       │ │  Alertmanager   │ │      Loki        │
│  (dashboards)    │ │  (notifications) │ │  (log storage)   │
│  :3000           │ │  :9093           │ │  :3100           │
│  sources:        │ │  ────────▶ Email │ │                  │
│  Prometheus, Loki│ │  (Gmail SMTP)   │ │                  │
└─────────────────┘ └─────────────────┘ └────────▲─────────┘
                                                  │
                                                  │ logs
                          ┌───────────────────────┴──────────┐
                          │            Promtail                │
                          │  ┌─────────────┐ ┌──────────────┐ │
                          │  │ /var/log/*  │ │ docker logs  │ │
                          │  │ (system)    │ │ (containers) │ │
                          │  └─────────────┘ └──────────────┘ │
                          │  :9080                             │
                          └────────────────────────────────────┘
```

**Network topology:** Services are split across two external Docker networks:
- `frontend` — Public-facing services (Prometheus, Grafana)
- `backend` — Internal services (Alertmanager, Loki, Promtail, Node Exporter)

---

## Tech Stack

| Service | Role | Image | Port |
|---------|------|-------|------|
| **Prometheus** | Metrics storage & alert evaluation | `prom/prometheus:latest` | `9090` |
| **Grafana** | Metrics & log visualization | `grafana/grafana:latest` | `3000` |
| **Alertmanager** | Alert routing & notification dispatch | `prom/alertmanager:latest` | `9093` |
| **Loki** | Log aggregation engine | `grafana/loki:3.0.0` | `3100` |
| **Promtail** | Log shipping agent | `grafana/promtail:3.0.0` | `9080` |
| **Node Exporter** | Hardware & OS metrics exporter | `prom/node-exporter:latest` | `9100` |

All services run in Docker containers with `restart: unless-stopped`.

---

## Prerequisites

- **Docker Engine** ≥ 24.x
- **Docker Compose** v2 (plugin)
- Two external Docker networks must exist:

```bash
docker network create frontend
docker network create backend
```

- Email credentials (if using Alertmanager with Gmail SMTP)
- Optional: Loki's default config uses in-memory storage — persistent storage can be configured

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/ArthurPinheiroMachado/homelab-macbook.git
cd homelab-macbook

# Create external Docker networks
docker network create frontend
docker network create backend

# Start all services (single command)
docker compose up -d

# Verify the stack
docker ps

# Access Grafana at http://localhost:3000
# Access Prometheus at http://localhost:9090
# Access Alertmanager at http://localhost:9093
```

---

## Service Details

### Prometheus

The central metrics system. Configured with a 15-second scrape interval and four scrape jobs:

- `prometheus` — Self-monitoring at `localhost:9090`
- `node_exporter` — System metrics at `node-exporter:9100`
- `grafana` — Grafana's internal metrics at `grafana:3000`
- `loki` — Loki metrics at `loki:3100`

Alerting rules in `prometheus/config/rules/node-alerts.yaml` are evaluated every 15 seconds. Alerts are sent to Alertmanager for routing.

### Grafana

Visualization layer. Configured to consume data from Prometheus (metrics) and Loki (logs). No default dashboards are bundled — they can be created manually or imported from the Grafana community dashboards:

- **Node Exporter Full** — ID `1860`
- **Loki / Promtail** — ID `13639`

### Alertmanager

Receives alerts from Prometheus and routes them based on label matchers. Currently configured to send email notifications via **Gmail SMTP** with distinct subject lines per alert type.

> **Note:** You must update `alertmanager/config/alertmanager.yaml` with your own SMTP credentials.

### Loki

Log aggregation system. Runs in single-binary mode with:
- In-memory ring storage
- Filesystem-based log chunk storage
- TSDB index store
- Authentication disabled (suitable for internal networks only)

### Promtail

Log shipping agent deployed alongside the host. It tails:
- **System logs** — `/var/log/*.log`
- **Docker logs** — `/var/lib/docker/containers/*/*-json.log`

Logs are labeled by job (`syslog` or `docker`) and host (`homelab`), then pushed to Loki at `http://loki:3100/loki/api/v1/push`.

### Node Exporter

Exposes standard system metrics: CPU, memory, disk, network, and load. Runs with `--path.rootfs=/host` to access the host filesystem via bind mount.

---

## Alerting Rules

| Alert Name | Expression | Threshold | Duration | Action |
|-----------|-----------|-----------|----------|--------|
| `HighCPUUsage` | CPU idle < 20% | > 80% used | 2 min | Email |
| `HighMemoryUsage` | Available RAM < 10% | > 90% used | 2 min | Email |
| `DiskSpaceLow` | Available disk < 20% | > 80% used | 5 min | Email |
| `InstanceDown` | Any service unreachable | `up == 0` | 1 min | Email |
| `LokiDown` | Loki unreachable | `up{job="loki"} == 0` | 1 min | Email |
| `GrafanaDown` | Grafana unreachable | `up{job="grafana"} == 0` | 1 min | Email |
| `PrometheusDown` | Prometheus unreachable | `up{job="prometheus"} == 0` | 1 min | Email |

All alerts are routed through Alertmanager, which groups and sends email notifications with service-specific subject lines.

---

## Project Structure

```
homelab-macbook/
├── .github/workflows/       # GitHub Actions CI pipelines
│   ├── compose-validation.yaml
│   ├── prometheus-validation.yaml
│   ├── alertmanager-validation.yaml
│   ├── loki-validation.yaml
│   ├── promtail-validation.yaml
│   └── observability-smoke-test.yaml
├── alertmanager/
│   ├── docker-compose.yaml       # Alertmanager service definition
│   └── config/
│       └── alertmanager.yaml     # Notification routing & receivers
├── grafana/
│   └── docker-compose.yaml       # Grafana service definition
├── loki/
│   ├── docker-compose.yaml       # Loki service definition
│   └── config/
│       └── loki-config.yaml      # Log storage & retention config
├── node_exporter/
│   └── docker-compose.yaml       # Node Exporter service definition
├── prometheus/
│   ├── docker-compose.yaml       # Prometheus service definition
│   └── config/
│       ├── prometheus.yaml       # Scrape jobs & global config
│       └── rules/
│           └── node-alerts.yaml  # Alerting rule definitions
├── promtail/
│   ├── docker-compose.yaml       # Promtail service definition
│   └── config/
│       └── promtail-config.yaml  # Log scrape targets & labels
├── docker-compose.yaml          # Root orchestrator for all services
├── .gitignore
├── LICENSE
└── README.md
```

---

## CI/CD

All GitHub Actions workflows are triggered on **pull requests** targeting the main branch:

| Workflow | Purpose |
|----------|---------|
| **Compose Validation** | Validates all Docker Compose files parse correctly |
| **Prometheus Validation** | Uses `promtool check config` to validate Prometheus config and rules |
| **Alertmanager Validation** | Uses `amtool check-config` to validate Alertmanager config |
| **Loki Validation** | Uses `loki -verify-config` to validate Loki config |
| **Promtail Validation** | Uses `promtail -verify-config` to validate Promtail config |
| **Smoke Test** | Brings up the full stack and verifies all endpoints respond correctly |

This ensures that every change is validated before merging, preventing broken configs from reaching production.

---

## Roadmap

- [x] **Docker Compose orchestrator** — Single root `docker-compose.yaml` to manage the entire stack
- [ ] **Dashboards as Code** — Version-controlled Grafana dashboard JSON files
- [ ] **Container orchestration** — Migrate to Docker Swarm or single-node K3s
- [ ] **TLS/SSL** — Reverse proxy with Let's Encrypt (Traefik or Caddy)
- [ ] **Infrastructure as Code** — Terraform or Ansible for reproducible deployment
- [ ] **Metrics retention** — Long-term Prometheus storage with Thanos or Mimir

---

## License

Distributed under the **MIT License**. See [`LICENSE`](LICENSE) for more information.

---

## Acknowledgments

This project was inspired by [Christian Lempa](https://github.com/christianlempa)'s homelab repository. His work provided a solid foundation for understanding how to structure a professional-grade monitoring stack at home.

---

<p align="center">
  <a href="README.pt-BR.md">🇧🇷 Leia em Português</a>
</p>
