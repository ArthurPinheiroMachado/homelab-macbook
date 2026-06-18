![Status](https://img.shields.io/badge/status-active-success)
![Smoke Test](https://img.shields.io/github/actions/workflow/status/ArthurPinheiroMachado/homelab-macbook/observability-smoke-test.yaml?label=smoke%20test)
![Compose Validation](https://img.shields.io/github/actions/workflow/status/ArthurPinheiroMachado/homelab-macbook/compose-validation.yaml?label=compose%20validation)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana)
![Loki](https://img.shields.io/badge/Loki-899CFF?logo=grafana)

# Plataforma de Monitoramento Homelab

Uma plataforma de monitoramento inspirada em produção, construída para praticar conceitos de Site Reliability Engineering (SRE), DevOps, Infraestrutura como Código, CI/CD e Observabilidade.

Este projeto simula como equipes modernas de engenharia gerenciam infraestrutura de monitoramento através de controle de versão, pipelines de validação automatizados e deploys no estilo GitOps.

---

## Sumário

- [Sobre](#sobre)
- [Especificações do Servidor](#especificações-do-servidor)
- [Arquitetura](#arquitetura)
- [Tecnologias](#tecnologias)
- [Pré-requisitos](#pré-requisitos)
- [Início Rápido](#início-rápido)
- [Detalhes dos Serviços](#detalhes-dos-serviços)
- [Regras de Alerta](#regras-de-alerta)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [CI/CD](#cicd)
- [Roadmap](#roadmap)
- [Licença](#licença)
- [Agradecimentos](#agradecimentos)

---

## Sobre

Este repositório contém a configuração completa do meu ambiente de homelab para monitoramento. O objetivo principal é monitorar a saúde e performance de um MacBook usado como servidor doméstico — acompanhando CPU, memória, disco e disponibilidade de serviços ao longo do tempo.

**O que este projeto demonstra:**

- **Conteinerização** — Docker Compose com isolamento de rede multi-camadas (`frontend`/`backend`)
- **Coleta de Métricas** — Prometheus + Node Exporter para métricas do sistema e serviços
- **Agregação de Logs** — Loki + Promtail para centralização de logs do sistema e contêineres Docker
- **Alertas** — Regras do Prometheus avaliadas contra threshold de métricas, roteadas pelo Alertmanager para e-mail
- **Visualização** — Dashboards Grafana consumindo dados do Prometheus e Loki
- **CI/CD** — GitHub Actions validando cada arquivo de configuração e executando smoke tests em PRs
- **Infraestrutura como Código** — Definições de serviços, configurações e regras de alerta versionadas

---

## Especificações do Servidor

| Componente | Especificação |
|------------|---------------|
| **Modelo** | MacBook Pro 13" Retina (A1502) |
| **CPU** | Intel I5-4278U @ 2.60 GHz |
| **RAM** | 8 GB DDR3 |
| **Armazenamento** | 120 GB SSD |
| **SO** | Ubuntu 24.04.4 LTS |
| **Finalidade** | Servidor doméstico, monitoramento, desenvolvimento |

---

## Arquitetura

```
                        ┌──────────────────────┐
                        │    Node Exporter      │
                        │  (métricas do host)   │
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
│  (dashboards)    │ │  (notificações) │ │ (armazenamento   │
│  :3000           │ │  :9093           │ │  de logs)        │
│  fontes:         │ │  ────────▶ Email │ │  :3100           │
│  Prometheus, Loki│ │  (Gmail SMTP)   │ │                  │
└─────────────────┘ └─────────────────┘ └────────▲─────────┘
                                                  │
                                                  │ logs
                          ┌───────────────────────┴──────────┐
                          │            Promtail                │
                          │  ┌─────────────┐ ┌──────────────┐ │
                          │  │ /var/log/*  │ │ logs Docker  │ │
                          │  │ (sistema)   │ │ (contêineres)│ │
                          │  └─────────────┘ └──────────────┘ │
                          │  :9080                             │
                          └────────────────────────────────────┘
```

**Topologia de rede:** Os serviços são divididos em duas redes Docker externas:
- `frontend` — Serviços com interface pública (Prometheus, Grafana)
- `backend` — Serviços internos (Alertmanager, Loki, Promtail, Node Exporter)

---

## Tecnologias

| Serviço | Função | Imagem | Porta |
|---------|--------|--------|-------|
| **Prometheus** | Armazenamento de métricas e avaliação de alertas | `prom/prometheus:latest` | `9090` |
| **Grafana** | Visualização de métricas e logs | `grafana/grafana:latest` | `3000` |
| **Alertmanager** | Roteamento de alertas e notificações | `prom/alertmanager:latest` | `9093` |
| **Loki** | Motor de agregação de logs | `grafana/loki:3.0.0` | `3100` |
| **Promtail** | Agente de coleta de logs | `grafana/promtail:3.0.0` | `9080` |
| **Node Exporter** | Expositor de métricas de hardware e SO | `prom/node-exporter:latest` | `9100` |

Todos os serviços rodam em contêineres Docker com `restart: unless-stopped`.

---

## Pré-requisitos

- **Docker Engine** ≥ 24.x
- **Docker Compose** v2 (plugin)
- Duas redes Docker externas devem existir:

```bash
docker network create frontend
docker network create backend
```

- Credenciais de e-mail (se for usar Alertmanager com Gmail SMTP)
- Opcional: a config padrão do Loki usa armazenamento in-memory — é possível configurar armazenamento persistente

---

## Início Rápido

```bash
# Clone o repositório
git clone https://github.com/ArthurPinheiroMachado/homelab-macbook.git
cd homelab-macbook

# Crie as redes Docker externas
docker network create frontend
docker network create backend

# Inicie todos os serviços (comando único)
docker compose up -d

# Verifique a stack
docker ps

# Acesse o Grafana em http://localhost:3000
# Acesse o Prometheus em http://localhost:9090
# Acesse o Alertmanager em http://localhost:9093
```

> **Nota:** Um `docker-compose.yaml` raiz já está incluído para orquestrar todos os serviços com um único comando.

---

## Detalhes dos Serviços

### Prometheus

Sistema central de métricas. Configurado com intervalo de coleta de 15 segundos e quatro jobs de scrape:

- `prometheus` — Auto-monitoramento em `localhost:9090`
- `node_exporter` — Métricas do sistema em `node-exporter:9100`
- `grafana` — Métricas internas do Grafana em `grafana:3000`
- `loki` — Métricas do Loki em `loki:3100`

As regras de alerta em `prometheus/config/rules/node-alerts.yaml` são avaliadas a cada 15 segundos. Alertas são enviados ao Alertmanager para roteamento.

### Grafana

Camada de visualização. Configurado para consumir dados do Prometheus (métricas) e Loki (logs). Nenhum dashboard padrão está incluído — eles podem ser criados manualmente ou importados da comunidade:

- **Node Exporter Full** — ID `1860`
- **Loki / Promtail** — ID `13639`

### Alertmanager

Recebe alertas do Prometheus e os roteia com base em matchers de labels. Atualmente configurado para enviar notificações por e-mail via **Gmail SMTP** com linhas de assunto distintas por tipo de alerta.

> **Nota:** Você deve atualizar `alertmanager/config/alertmanager.yaml` com suas próprias credenciais SMTP.

### Loki

Sistema de agregação de logs. Roda em modo single-binary com:
- Armazenamento de ring in-memory
- Armazenamento de chunks em filesystem
- Índice TSDB
- Autenticação desabilitada (adequado apenas para redes internas)

### Promtail

Agente de coleta de logs executado junto ao host. Ele monitora:
- **Logs do sistema** — `/var/log/*.log`
- **Logs do Docker** — `/var/lib/docker/containers/*/*-json.log`

Os logs são rotulados por job (`syslog` ou `docker`) e host (`homelab`), e enviados ao Loki em `http://loki:3100/loki/api/v1/push`.

### Node Exporter

Expõe métricas padrão do sistema: CPU, memória, disco, rede e carga. Roda com `--path.rootfs=/host` para acessar o filesystem do host via bind mount.

---

## Regras de Alerta

| Nome do Alerta | Expressão | Threshold | Duração | Ação |
|---------------|-----------|-----------|---------|------|
| `HighCPUUsage` | CPU idle < 20% | > 80% usado | 2 min | E-mail |
| `HighMemoryUsage` | RAM disponível < 10% | > 90% usado | 2 min | E-mail |
| `DiskSpaceLow` | Disco disponível < 20% | > 80% usado | 5 min | E-mail |
| `InstanceDown` | Serviço indisponível | `up == 0` | 1 min | E-mail |
| `LokiDown` | Loki indisponível | `up{job="loki"} == 0` | 1 min | E-mail |
| `GrafanaDown` | Grafana indisponível | `up{job="grafana"} == 0` | 1 min | E-mail |
| `PrometheusDown` | Prometheus indisponível | `up{job="prometheus"} == 0` | 1 min | E-mail |

Todos os alertas são roteados através do Alertmanager, que agrupa e envia notificações por e-mail com linhas de assunto específicas para cada serviço.

---

## Estrutura do Projeto

```
homelab-macbook/
├── .github/workflows/       # Pipelines de CI do GitHub Actions
│   ├── compose-validation.yaml
│   ├── prometheus-validation.yaml
│   ├── alertmanager-validation.yaml
│   ├── loki-validation.yaml
│   ├── promtail-validation.yaml
│   └── observability-smoke-test.yaml
├── alertmanager/
│   ├── docker-compose.yaml       # Definição do serviço Alertmanager
│   └── config/
│       └── alertmanager.yaml     # Roteamento de notificações e receivers
├── grafana/
│   └── docker-compose.yaml       # Definição do serviço Grafana
├── loki/
│   ├── docker-compose.yaml       # Definição do serviço Loki
│   └── config/
│       └── loki-config.yaml      # Configuração de armazenamento e retenção de logs
├── node_exporter/
│   └── docker-compose.yaml       # Definição do serviço Node Exporter
├── prometheus/
│   ├── docker-compose.yaml       # Definição do serviço Prometheus
│   └── config/
│       ├── prometheus.yaml       # Jobs de scrape e configuração global
│       └── rules/
│           └── node-alerts.yaml  # Definições de regras de alerta
├── promtail/
│   ├── docker-compose.yaml       # Definição do serviço Promtail
│   └── config/
│       └── promtail-config.yaml  # Alvos de log e labels
├── docker-compose.yaml          # Orquestrador raiz para todos os serviços
├── .gitignore
├── LICENSE
├── README.md
└── README.pt-BR.md
```

---

## CI/CD

Todos os workflows do GitHub Actions são disparados em **pull requests** para a branch main:

| Workflow | Propósito |
|----------|-----------|
| **Compose Validation** | Valida se todos os arquivos Docker Compose estão sintaticamente corretos |
| **Prometheus Validation** | Usa `promtool check config` para validar config e regras do Prometheus |
| **Alertmanager Validation** | Usa `amtool check-config` para validar config do Alertmanager |
| **Loki Validation** | Usa `loki -verify-config` para validar config do Loki |
| **Promtail Validation** | Usa `promtail -verify-config` para validar config do Promtail |
| **Smoke Test** | Sobe a stack completa e verifica se todos os endpoints respondem corretamente |

Isso garante que toda alteração seja validada antes do merge, prevenindo que configurações quebradas cheguem à produção.

---

## Roadmap

- [x] **Orquestrador Docker Compose** — `docker-compose.yaml` raiz único para gerenciar toda a stack
- [ ] **Dashboards como Código** — Arquivos JSON de dashboards Grafana versionados
- [ ] **Orquestração de contêineres** — Migrar para Docker Swarm ou K3s single-node
- [ ] **TLS/SSL** — Proxy reverso com Let's Encrypt (Traefik ou Caddy)
- [ ] **Infraestrutura como Código** — Terraform ou Ansible para deployment reproduzível
- [ ] **Retenção de métricas** — Armazenamento de longo prazo com Thanos ou Mimir

---

## Licença

Distribuído sob a **Licença MIT**. Veja [`LICENSE`](LICENSE) para mais informações.

---

## Agradecimentos

Este projeto foi inspirado pelo repositório homelab do [Christian Lempa](https://github.com/christianlempa). O trabalho dele forneceu uma base sólida para entender como estruturar uma stack de monitoramento profissional em ambiente doméstico.

---

<p align="center">
  <a href="README.md">🇺🇸 Read in English</a>
</p>
