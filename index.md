---
layout: home
title: Home
---

# DSpace CRIS Cockpit Documentation

![Version](https://img.shields.io/badge/version-2024.02.04-blue?style=flat-square)
![React](https://img.shields.io/badge/React-18-61dafb?style=flat-square&logo=react&logoColor=white)
![Django](https://img.shields.io/badge/Django-4.2-092e20?style=flat-square&logo=django&logoColor=white)
![DSpace](https://img.shields.io/badge/DSpace-CRIS%208-purple?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ed?style=flat-square&logo=docker&logoColor=white)

A single-page React application layered on top of **DSpace 7 / DSpace CRIS** providing an enhanced submission workspace, discoverable entity browsing, admin configuration, and an optional Django sidecar for runtime-configurable features.

---

## Documentation Guides

<div class="service-grid">
  <div class="service-card">
    <h4>🛠 Developer Guide</h4>
    <p>Architecture, project structure, routing, auth, config system, API reference, deployment.</p>
    <br><a href="{{ '/dev/' | relative_url }}">Read →</a>
  </div>
  <div class="service-card">
    <h4>⚙️ Admin Guide</h4>
    <p>Feature flags, dashboard clusters, quicklinks presets, form builder, community management.</p>
    <br><a href="{{ '/admin/' | relative_url }}">Read →</a>
  </div>
  <div class="service-card">
    <h4>👤 User Guide</h4>
    <p>Logging in, dashboard, workspace, search, quicklinks, communities, profile, FAQ.</p>
    <br><a href="{{ '/user/' | relative_url }}">Read →</a>
  </div>
  <div class="service-card">
    <h4>📊 Operations Guide</h4>
    <p>Docker Compose stacks, monitoring with Prometheus + Loki + Grafana, alerting setup.</p>
    <br><a href="{{ '/ops/' | relative_url }}">Read →</a>
  </div>
</div>

---

## Architecture at a Glance

```mermaid
graph LR
    Browser["🌐 Browser / SPA"]
    Nginx["⚙️ nginx :80"]
    DSpace["🗄 DSpace CRIS :8080"]
    Django["🐍 Django :5189"]
    PG1[("🐘 PostgreSQL\ndspace")]
    PG2[("🐘 PostgreSQL\ndjango_config")]
    Solr["🔍 Solr :8983"]

    Browser -->|HTTPS| Nginx
    Nginx -->|"/server/*"| DSpace
    Nginx -->|"/api/dspace-config/*"| Django
    Nginx -->|"/* static"| Browser
    DSpace --- PG1
    DSpace --- Solr
    Django --- PG2
    Django -->|JWT validation| DSpace
```

---

## Quick Links

| I want to… | Go to |
|---|---|
| Run the full stack locally | [Quick Start](/dev/quickstart/) |
| Understand the system design | [Architecture](/dev/architecture/) |
| Configure environment variables | [Deployment](/dev/deployment/) |
| Set up monitoring & alerts | [Monitoring](/ops/monitoring/) |
| Manage dashboard clusters | [Admin → Clusters](/admin/clusters/) |
| Manage quicklinks presets | [Admin → Quicklinks](/admin/quicklinks/) |
| Customise submission forms | [Admin → Form Builder](/admin/formbuilder/) |
| Log in and get started | [User → Getting Started](/user/getting-started/) |
| Find my draft submissions | [User → Workspace](/user/workspace/) |
| Search the repository | [User → Search](/user/search/) |
| Understand a term or badge | [User → FAQ & Glossary](/user/faq/) |
