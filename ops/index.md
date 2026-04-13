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
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169e1?style=flat-square&logo=postgresql&logoColor=white)
![Solr](https://img.shields.io/badge/Solr-8.11-d9411e?style=flat-square&logo=apachesolr&logoColor=white)

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
| [Database Change Logging]({{ '/ops/database-logging/' | relative_url }}) | DDL and DML audit trail — `history` and `logging` schemas, trigger setup, housekeeping |
| [Database Export Views]({{ '/ops/database-export-views/' | relative_url }}) | `export` schema — user, structure, and permissions views for reporting and backup consumers |
| [Database Performance]({{ '/ops/database-performance/' | relative_url }}) | `pg_stat_statements`, slow query analysis, index usage, cache hit ratio |
| [Solr Query Reference]({{ '/ops/solr-queries/' | relative_url }}) | Query syntax, field types, range queries, relevance tuning, performance monitoring |
| [Solr Search Core Schema]({{ '/ops/solr-search-core/' | relative_url }}) | `schema.xml`, `solrconfig.xml`, field types, analyser pipelines, dynamic fields, text config files |
| [Solr 9 — DSpace 2025 Upgrade]({{ '/ops/solr9-upgrade/' | relative_url }}) | `solrconfig.xml` and `schema.xml` diff from Solr 8, mandatory startup flags, migration steps, new compose files |
| [Solr 9 — New Features]({{ '/ops/solr9-features/' | relative_url }}) | Scripted `UpdateRequestProcessor`, KNN vector search, highlighting, nested documents, streaming expressions |
| [Kubernetes & OpenShift]({{ '/ops/kubernetes/' | relative_url }}) | K8s manifests, OpenShift Routes/SCC, image build pipeline, secrets management |

<div class="callout callout-info">
<span class="callout-title">Monitoring is optional</span>
The monitoring stack is completely independent of the application stack. The core application runs fine without it.
</div>

<div class="callout callout-info">
<span class="callout-title">Database logging applies to production only</span>
The DDL and DML change logging setup targets the <code>dspace</code> database.
</div>

<div class="page-nav">
  <span></span>
  <a href="{{ '/ops/stack/' | relative_url }}">Base Stack →</a>
</div>
