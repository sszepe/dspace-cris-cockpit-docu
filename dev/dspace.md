---
layout: page
title: DSpace Integration
permalink: /dev/dspace/
parent: Developer Guide
---

# DSpace Integration

![DSpace](https://img.shields.io/badge/DSpace-CRIS%208-purple?style=flat-square)
![HAL](https://img.shields.io/badge/API-HAL%20REST-blueviolet?style=flat-square)

## Key REST Endpoints Used

| Endpoint | Used For |
|---|---|
| `GET /api/authn/status` | Session validation, eperson href retrieval |
| `POST /api/authn/login` | Username/password → JWT |
| `POST /api/authn/logout` | Session invalidation |
| `GET /api/core/epersons/:id` | Full EPerson profile |
| `GET /api/core/epersons/:id/groups` | Group memberships (role detection) |
| `GET /api/core/entitytypes` | Accessible entity types |
| `GET /api/core/entitytypes/search/findAllByAuthorizedCollection` | `canCreate()` gate |
| `GET /api/discover/search/objects` | Full-text + faceted Discovery search |
| `GET /api/core/communities` | Community list |
| `POST /api/core/communities` | Community creation |
| `GET /api/core/collections` | Collection list |
| `POST /api/core/collections` | Collection creation |
| `GET /api/submission/workspaceitems` | Workspace item list |
| `GET /api/core/items/:uuid` | Published item detail |
| `POST /api/core/bundles/:id/bitstreams` | Bitstream upload |
| `DELETE /api/core/bitstreams/:id` | Bitstream delete |

## JWT + CSRF Auth Flow

DSpace 7 uses **stateless JWT auth** combined with a **CSRF cookie** for write protection.

```mermaid
sequenceDiagram
    participant B as Browser
    participant DS as DSpace REST

    B->>DS: GET /api/security/csrf
    DS-->>B: Set-Cookie: DSPACE-XSRF-COOKIE<br/>Header: DSPACE-XSRF-TOKEN

    B->>DS: POST /api/authn/login<br/>Content-Type: x-www-form-urlencoded<br/>X-XSRF-TOKEN: <csrf><br/>user=...&password=...
    DS-->>B: Authorization: Bearer <JWT><br/>DSPACE-XSRF-TOKEN: <rotated>

    Note over B: JWT → sessionStorage["jwt"]<br/>CSRF → sessionStorage["csrf"]

    B->>DS: POST /api/core/communities<br/>Authorization: Bearer <JWT><br/>X-XSRF-TOKEN: <csrf><br/>Content-Type: application/json
    DS-->>B: 201 Created<br/>DSPACE-XSRF-TOKEN: <re-rotated>
```

<div class="callout callout-info">
<span class="callout-title">CSRF rotation</span>
DSpace rotates the CSRF token on every successful mutation. The client reads the new value from the <code>DSPACE-XSRF-TOKEN</code> response header and updates sessionStorage via <code>setStoredCsrfToken()</code> automatically.
</div>

<div class="callout callout-danger">
<span class="callout-title">Development CSRF cookie</span>
Set <code>rest.cors.cookie.secure = false</code> in <code>local.cfg</code> for plain HTTP development. <strong>Never use in production</strong> — require HTTPS instead.
</div>

## Entity Types

| Entity Type | Quicklink Preset | Default Filters |
|---|---|---|
| Publication | ✓ | dc.type, dc.date.issued, dc.contributor.author |
| Project | ✓ | investigator, coordinator, status, start/end date |
| Funding | ✓ | itemtype, funder |
| Person | ✓ | affiliation |
| OrgUnit | ✓ | dc.type, country |
| Equipment | ✓ | itemtype |
| Event | ✓ | itemtype, start/end date |
| Product | ✓ | dc.type, dc.contributor.author |
| Patent | ✓ | dc.contributor.author (Inventor) |
| Journal | ✓ | dc.publisher |
| Place, ConceptScheme, Concept, ArchivalResource | Institution-specific | Separate API modules |

## Submission API Modules

| Entity Type | API Module | Notes |
|---|---|---|
| Publication / generic | DSpace `/submission/workspaceitems` | Standard workflow |
| OrgUnit | `orgunit-creation-api.ts` | Direct creation |
| Person | `person-api.ts` | Workspace or import |
| Funding | `funding-creation-api.ts` | Includes `funding-programme-api.ts` |
| Equipment | `equipment-creation-api.ts` | Direct creation |
| Place | `place-creation-api.ts` / `place-import-api.ts` | Manual or Geonames import |
| SKOS Concept/Scheme | `skos-creation-api.ts` / `skos-edit-api.ts` | Controlled vocabulary |
| Archival Resource | `archival-resource-api.ts` | Archival metadata |
| OrgUnit (batch) | `orgunit-import-api.ts` | Batch import |

<div class="callout callout-tip">
<span class="callout-title">Adding a new entity type</span>
Create <code>my-entity-creation-api.ts</code> in <code>src/api/</code>, a modal in <code>src/components/</code>, add to the entity-clusters config or Django DB, and wire the <code>canCreate("MyEntity")</code> gate from <code>useAuth()</code>.
</div>

<div class="page-nav">
  <a href="{{ '/dev/backend/' | relative_url }}">← Backend</a>
  <a href="{{ '/dev/deployment/' | relative_url }}">Deployment →</a>
</div>
