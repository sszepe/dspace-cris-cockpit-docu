---
layout: page
title: Backend (Django)
permalink: /dev/backend/
parent: Developer Guide
---

# Backend — Django Sidecar

![Django](https://img.shields.io/badge/Django-4.2-092e20?style=flat-square&logo=django&logoColor=white)
![DRF](https://img.shields.io/badge/DRF-3.14-red?style=flat-square)
![Python](https://img.shields.io/badge/Python-3.11-3776ab?style=flat-square&logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169e1?style=flat-square&logo=postgresql&logoColor=white)

The Django sidecar is a lightweight **Django REST Framework** application serving runtime-configurable settings for the Cockpit. It does not interact with the repository directly.

## Tech Stack

| Package | Version | Purpose |
|---|---|---|
| Django | ≥4.2, <5.0 | Web framework |
| djangorestframework | ≥3.14 | REST API layer |
| django-cors-headers | ≥4.3 | CORS for React dev server |
| psycopg2-binary | ≥2.9 | PostgreSQL driver |
| gunicorn | ≥21.2 | Production WSGI server |
| drf-spectacular | ≥0.27 | OpenAPI schema generation |
| whitenoise | ≥6.6 | Static file serving |
| requests | ≥2.31 | Proxy JWT validation to DSpace |

---

## Authentication

Django has no user database. The custom `DSpaceJWTAuthentication` backend validates every request against DSpace's `/api/authn/status` endpoint.

```python
class DSpaceJWTAuthentication(BaseAuthentication):
    def authenticate(self, request):
        token = extract_bearer_token(request)
        r = requests.get(
            f"{DSPACE_BASE_URL}/api/authn/status",
            headers={"Authorization": f"Bearer {token}"},
            timeout=5,
        )
        if not r.json().get("authenticated"):
            raise AuthenticationFailed(...)
        return (DSpaceUser(email, name, groups), token)

class DSpaceUser:
    @property
    def is_admin(self):
        return "Administrator" in self.groups  # DSpace special groups
```

<div class="callout callout-warn">
<span class="callout-title">One extra HTTP hop per request</span>
Every Django API request makes one outbound call to DSpace to validate the JWT. Consider caching the validation result per-token (e.g. with a short TTL) if API latency matters.
</div>

### Debug Endpoint

`GET /api/dspace-config/debug/auth/` — available without authentication, shows connectivity status:

```json
{
  "django_reachable": true,
  "dspace_base_url": "http://dspace:8080/server",
  "dspace_reachable": true,
  "dspace_authenticated": true,
  "jwt_received": true,
  "jwt_preview": "eyJhbGciOi…",
  "settings_module": "dspace_config.settings.local"
}
```

---

## API Reference

All endpoints under prefix `/api/dspace-config/`.
All require `Authorization: Bearer <JWT>` except `/debug/auth/`.
Write endpoints require `is_admin = true`.

### Auth & Diagnostics

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/debug/auth/` | Public | Connectivity probe |

### Dashboard Clusters

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/dashboard-config/` | User | Enabled clusters with entity types |
| `GET` | `/clusters/` | User | All clusters including disabled |
| `POST` | `/clusters/` | **Admin** | Create cluster |
| `GET` | `/clusters/:id/` | User | Single cluster |
| `PATCH` | `/clusters/:id/` | **Admin** | Update cluster (partial) |
| `DELETE` | `/clusters/:id/` | **Admin** | Delete cluster + entries |
| `POST` | `/clusters/:id/entity-types/` | **Admin** | Add entity type to cluster |
| `PATCH` | `/entity-types/:id/` | **Admin** | Update entity type label/order |
| `DELETE` | `/entity-types/:id/` | **Admin** | Remove entity type |

### Site Settings

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/site-settings/` | User | Singleton settings row |
| `PATCH` | `/site-settings/` | **Admin** | Toggle feature flags |

### Quicklinks

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/quicklinks/` | User | Enabled presets with filters |
| `GET` | `/quickpresets/` | User | All presets |
| `POST` | `/quickpresets/` | **Admin** | Create preset |
| `PATCH` | `/quickpresets/:id/` | **Admin** | Update preset |
| `DELETE` | `/quickpresets/:id/` | **Admin** | Delete preset + filters |
| `POST` | `/quickpresets/:id/filters/` | **Admin** | Add filter to preset |
| `PATCH` | `/quickpreset-filters/:id/` | **Admin** | Update filter |
| `DELETE` | `/quickpreset-filters/:id/` | **Admin** | Remove filter |

### Submission Forms & Metadata

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/submission-forms/` | User | List all imported forms |
| `GET` | `/submission-forms/:id/` | User | Full form with fields |
| `GET` | `/submission-forms/:id/fields/` | User | Fields + metadata registry annotation |
| `GET` | `/form-layouts/` | User | Layouts; filterable by form_name, profile, collection |
| `POST` | `/form-layouts/` | **Admin** | Create layout |
| `PATCH` | `/form-layouts/:id/` | **Admin** | Update layout |
| `DELETE` | `/form-layouts/:id/` | **Admin** | Delete layout |
| `GET` | `/metadata-schemas/` | User | All schemas; searchable with `?q=` |
| `GET` | `/metadata-schemas/:id/` | User | Schema with all fields |
| `GET` | `/metadata-fields/` | User | Field autocomplete `?q=dc.title&schema=dc` (max 100) |

### Audit

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/audit/field-usage/` | User | Form fields vs metadata registry — flags unknowns |
| `GET` | `/audit/forms-summary/` | User | Form count, field count, required flags |

---

## Data Models

```mermaid
erDiagram
    EntityCluster ||--o{ EntityTypeEntry : "entity_types"
    QuickPreset   ||--o{ QuickPresetFilter : "filters"
    FormLayout    ||--o{ FormSection : "sections"
    FormSection   ||--o{ FormFieldOverride : "field_overrides"
    FormLayout    ||--o{ FormConditionalBlock : "conditional_blocks"
    SubmissionForm ||--o{ SubmissionFormField : "fields"
    MetadataSchema ||--o{ MetadataField : "fields"

    EntityCluster {
        string key UK
        string label
        text description
        int sort_order
        bool enabled
    }
    EntityTypeEntry {
        fk cluster_id
        string entity_type_label
        int sort_order
    }
    QuickPreset {
        string key UK
        string label
        json base_filters
        int sort_order
        bool enabled
    }
    QuickPresetFilter {
        fk preset_id
        string key
        string label
        string facet_name
        string kind
        string placeholder
        int sort_order
    }
    SiteSettings {
        bool quicklinks_enabled
        bool communities_creation_enabled
        bool communities_role_management_enabled
        bool collections_creation_enabled
    }
    FormLayout {
        string form_name
        string profile
        uuid collection
        string label
    }
    FormSection {
        fk layout_id
        string key
        string label
        int sort_order
        bool collapsed_by_default
        text helper_text_above
        text helper_text_below
    }
    FormFieldOverride {
        fk section_id
        string field_name
        string label_override
        text hint_override
        bool hidden
        int sort_order
    }
    FormConditionalBlock {
        fk layout_id
        string trigger_field
        string trigger_value
        json revealed_fields
        string revealed_section
    }
```

<div class="page-nav">
  <a href="{{ '/dev/frontend/' | relative_url }}">← Frontend</a>
  <a href="{{ '/dev/dspace/' | relative_url }}">DSpace Integration →</a>
</div>
