---
layout: page
title: Operations Guide
permalink: /ops/
---

# Operations Guide

![Docker](https://img.shields.io/badge/Docker-Compose-2496ed?style=flat-square&logo=docker&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat-square&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-logs-yellow?style=flat-square)

This guide covers running and monitoring the DSpace CRIS Cockpit in production. Two Docker Compose files are provided:

| File | Purpose |
|---|---|
| `docker-compose_2024.yml` | Core application stack — DSpace, Django, Frontend, Config Cockpit, Postgres, Solr |
| `docker-compose_2024-monitoring.yml` | Monitoring overlay — Prometheus, Loki, Grafana, Fluent Bit, exporters, alerting |

## Sections

| Section | Contents |
|---|---|
| [Base Stack]({{ '/ops/stack/' | relative_url }}) | Core services, healthchecks, startup order, maintenance commands |
| [Monitoring]({{ '/ops/monitoring/' | relative_url }}) | Full monitoring stack setup, Grafana, log queries, metric reference |
| [Alerting]({{ '/ops/alerting/' | relative_url }}) | Alert rules, Alertmanager routing, Slack/email configuration |

<div class="callout callout-info">
<span class="callout-title">Monitoring is optional</span>
The monitoring stack is completely independent of the application stack. The core application runs fine without it.
</div>

<div class="page-nav">
  <span></span>
  <a href="{{ '/ops/stack/' | relative_url }}">Base Stack →</a>
</div>
