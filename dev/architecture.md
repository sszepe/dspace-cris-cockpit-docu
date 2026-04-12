---
layout: page
title: Architecture
permalink: /dev/architecture/
parent: Developer Guide
---

# Architecture

![Status](https://img.shields.io/badge/stack-Docker%20Compose-2496ed?style=flat-square&logo=docker&logoColor=white)
![DSpace](https://img.shields.io/badge/DSpace-CRIS%208%20%282024.02.04%29-purple?style=flat-square)

All services run inside Docker. Nginx acts as the public reverse proxy and routes traffic to the appropriate upstream based on path prefix.

## System Overview

```mermaid
graph TB
    subgraph Public["Public Interface"]
        Browser["🌐 Browser\n(React SPA)"]
        AdminBrowser["🔧 Admin Browser\n(Config Cockpit)"]
    end

    subgraph Proxy["Reverse Proxy"]
        Nginx["⚙️ nginx\n:80 / :443"]
        AdminNginx["⚙️ nginx\n:5174"]
    end

    subgraph App["Application Layer"]
        DSpace["🗄 DSpace CRIS\n:8080\n(Spring Boot)"]
        Django["🐍 Django Sidecar\n:5189\n(Gunicorn + DRF)"]
        Frontend["📦 Static Assets\n(Vite build)"]
        DjangoFrontend["🛠 Config Cockpit\n(Vite + React SPA)"]
    end

    subgraph Data["Data Layer"]
        PG["🐘 PostgreSQL 15\n:5432"]
        Solr["🔍 Solr 8\n:8983"]
    end

    Browser -->|"HTTP/HTTPS"| Nginx
    AdminBrowser -->|"HTTP/HTTPS"| AdminNginx
    Nginx -->|"/server/*"| DSpace
    Nginx -->|"/api/dspace-config/*"| Django
    Nginx -->|"/* (SPA fallback)"| Frontend
    AdminNginx -->|"/* (SPA fallback)"| DjangoFrontend
    DjangoFrontend -->|"/api/dspace-config/*"| Django

    DSpace -->|"dspace DB"| PG
    Django -->|"django_config DB"| PG
    DSpace -->|"search index"| Solr
    Django -->|"JWT validation"| DSpace
```

## Request Path Routing

```mermaid
flowchart LR
    Req["Incoming request"]
    Req --> N{"nginx\npath match?"}
    N -->|"/server/*"| DS["DSpace :8080\nRepository API"]
    N -->|"/api/dspace-config/*"| DJ["Django :5189\nConfig API"]
    N -->|"/* (catch-all)"| SPA["index.html\nReact SPA"]
    SPA -->|"hash routing"| Route["#/dashboard\n#/search\n#/item/:uuid\netc."]
```

## Component Responsibilities

| Component | Technology | Port | Responsibility |
|---|---|---|---|
| React SPA | React 18 + TypeScript + Vite | — | UI shell — all data from DSpace REST or Django API |
| Config Cockpit | React 18 + TypeScript + Vite | 5174 | Standalone admin SPA for Django config API — uses Django session auth |
| DSpace 7/CRIS | Java / Spring Boot | 8080 | Repository backend — auth, item CRUD, workspace, bitstreams |
| Django Sidecar | Django 4.2 + DRF | 5189 | Runtime config — clusters, presets, site settings, form layouts |
| Nginx (frontend) | nginx:alpine | 80 | Reverse proxy, static file serving, SPA fallback |
| Nginx (cockpit) | nginx:alpine | 5174 | Static file serving for Config Cockpit SPA |
| PostgreSQL | PostgreSQL 15 | 5432 | Two databases: `dspace` and `django_config` |
| Solr | Apache Solr 8 | 8983 | Full-text + faceted search index (Discovery API) |

## Authentication Flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant N as nginx
    participant DS as DSpace
    participant DJ as Django

    B->>DS: GET /server/api/security/csrf
    DS-->>B: Set-Cookie: DSPACE-XSRF-COOKIE + DSPACE-XSRF-TOKEN header

    B->>DS: POST /server/api/authn/login (user + password)
    DS-->>B: Authorization: Bearer <JWT>

    Note over B: JWT stored in sessionStorage

    B->>N: GET /api/dspace-config/clusters/<br/>Authorization: Bearer <JWT>
    N->>DJ: proxy_pass
    DJ->>DS: GET /server/api/authn/status<br/>Authorization: Bearer <JWT>
    DS-->>DJ: { authenticated: true, ... }
    DJ-->>B: 200 OK — cluster config JSON
```

<div class="callout callout-info">
<span class="callout-title">Django validates every request</span>
Django has no user database — it forwards the JWT to DSpace's <code>/api/authn/status</code> endpoint to verify each request. This means Django is always in sync with DSpace's auth state, but adds one extra HTTP hop per API call.
</div>

<div class="callout callout-info">
<span class="callout-title">Config Cockpit uses a separate auth model</span>
The Config Cockpit (<code>:5174</code>) authenticates with Django's own session auth (username + password for Django staff users), not the DSpace JWT flow. This makes it independently accessible without a running DSpace instance.
</div>

## Database Layout

```mermaid
erDiagram
    DSPACE_DB["dspace (PostgreSQL)"] {
        string items
        string collections
        string communities
        string epersons
        string workspaceitems
        string bundles
        string bitstreams
    }

    DJANGO_CONFIG_DB["django_config (PostgreSQL)"] {
        table EntityCluster
        table EntityTypeEntry
        table QuickPreset
        table QuickPresetFilter
        table SiteSettings
        table MetadataSchema
        table MetadataField
        table SubmissionForm
        table SubmissionFormField
        table FormLayout
        table FormSection
        table FormFieldOverride
        table FormConditionalBlock
    }
```

<div class="page-nav">
  <a href="{{ '/dev/' | relative_url }}">← Developer Guide</a>
  <a href="{{ '/dev/quickstart/' | relative_url }}">Quick Start →</a>
</div>
