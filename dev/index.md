---
layout: page
title: Developer Guide
permalink: /dev/
---

![React](https://img.shields.io/badge/React-18-61dafb?style=flat-square&logo=react&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178c6?style=flat-square&logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5-646cff?style=flat-square&logo=vite&logoColor=white)
![Django](https://img.shields.io/badge/Django-4.2-092e20?style=flat-square&logo=django&logoColor=white)

The DSpace CRIS Cockpit is a **thin UI shell** that authenticates against DSpace's native JWT auth endpoint, then proxies all repository operations directly to the DSpace REST API. An optional **Django sidecar** provides DB-backed runtime configuration.

## Sections

| Section | Contents |
|---|---|
| [Architecture]({{ '/dev/architecture/' | relative_url }}) | System diagram, component responsibilities, data flow |
| [Quick Start]({{ '/dev/quickstart/' | relative_url }}) | Local dev setup in 5 steps |
| [Frontend]({{ '/dev/frontend/' | relative_url }}) | Project structure, routing, AuthContext, config system, feature flags |
| [Backend (Django)]({{ '/dev/backend/' | relative_url }}) | Sidecar overview, auth, full API reference, data models |
| [DSpace Integration]({{ '/dev/dspace/' | relative_url }}) | REST API usage, JWT auth flow, entity types |
| [Deployment]({{ '/dev/deployment/' | relative_url }}) | Docker Compose, environment variables, nginx proxy notes |

---

## Code Repositories

| Repository | Contents |
|---|---|
| [Frontend](https://github.com/sszepe/dspace-cris-light-ui) | DSprce CRIS Vite / React / Typescript Frontend |
| [Frontend Config](https://github.com/sszepe/dspace-cris-light-ui-config) | Djangp-based Frontend Confoguration |
| [DSpace CRIS Cockpit Setup](https://github.com/sszepe/dspace-cris-cockpit) | Docker based setup for Frontend, optional Django-based config and monitoring; includes also developer script to auto-generate required types for setup |

---

## Key Concepts

<div class="callout callout-info">
<span class="callout-title">Django is optional</span>
Set <code>VITE_CLUSTER_CONFIG_SOURCE=ts</code> and <code>VITE_QUICKLINKS_CONFIG_SOURCE=ts</code> to run with only static TypeScript config. Django is only required when you need runtime-editable configuration without redeployment.
</div>

<div class="callout callout-tip">
<span class="callout-title">Hash-based routing</span>
The SPA uses <code>window.location.hash</code> for routing — no server-side route config is needed, and the nginx <code>try_files</code> fallback handles all deep links.
</div>

<div class="page-nav">
  <span></span>
  <a href="{{ '/dev/architecture/' | relative_url }}">Architecture →</a>
</div>
