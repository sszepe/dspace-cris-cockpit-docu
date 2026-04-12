---
layout: page
title: Collection Mappings
permalink: /admin/collection-mappings/
parent: Admin Guide
---

# Collection Mappings

![Managed in](https://img.shields.io/badge/managed%20in-Config%20Cockpit%20%3A5174-blue?style=flat-square)

Collection mappings route entity types to specific DSpace collection UUIDs. When a user creates an item of a given entity type, the mapping determines which collection it is submitted to. Optional conditions allow multiple mappings for the same entity type, each targeting a different collection based on item metadata.

---

## Concepts

| Term | Description |
|---|---|
| **Entity type** | DSpace entity type label, e.g. `Publication`, `Funding` |
| **Collection UUID** | The full UUID of the target DSpace collection |
| **Conditions** | Optional rules that restrict when this mapping applies |
| **Sort order** | When multiple mappings exist for the same entity type, they are evaluated in ascending sort order — the first match wins |

### Condition Types

| Condition | Field | How it matches |
|---|---|---|
| `dcTypeIncludes` | `dc.type` | Item's `dc.type` value contains any of the listed substrings |
| `risfundingStatusIn` | risfunding status | Item's status is exactly one of the listed values |

Leave both conditions empty to create a catch-all mapping that matches all items of that entity type.

---

## Managing Mappings

Navigate to **Config Cockpit → Collection Mappings** (`http://localhost:5174`). Click **+ New Mapping**.

### Creating a mapping

| Field | Required | Description |
|---|---|---|
| Entity Type | ✅ | e.g. `Funding`, `Publication` |
| Collection UUID | ✅ | Full UUID of the DSpace collection |
| Label | ✅ | Human-readable name, e.g. "EC Grants" |
| Sort Order | ✅ | Evaluation order when multiple mappings exist for the same entity type |
| dc.type includes | ❌ | Comma-separated list of `dc.type` substrings |
| risfunding status in | ❌ | Comma-separated list of exact status values |

### Example — conditional Funding mappings

Two mappings for `Funding`, evaluated in sort order:

| Sort | Label | Conditions | Collection |
|---|---|---|---|
| 1 | EC Grants | `dcTypeIncludes: ["Grant"]`, `risfundingStatusIn: ["approved"]` | `abc-123-...` |
| 2 | All Funding | _(none — catch-all)_ | `def-456-...` |

An approved Grant item matches mapping 1. All other Funding items fall through to mapping 2.

### Editing and deleting

Click **Edit** on any row to update its fields. Click **Delete** to permanently remove it.

---

## API Reference

```bash
# List all mappings
GET /api/dspace-config/collection-mappings/

# Create
POST /api/dspace-config/collection-mappings/
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "entity_type": "Funding",
  "collection_id": "abc-123-...",
  "label": "EC Grants",
  "sort_order": 1,
  "dc_type_includes": ["Grant"],
  "risfunding_status_in": ["approved"]
}

# Update
PATCH /api/dspace-config/collection-mappings/:id/

# Delete
DELETE /api/dspace-config/collection-mappings/:id/
```

<div class="page-nav">
  <a href="{{ '/admin/clusters/' | relative_url }}">← Dashboard Clusters</a>
  <a href="{{ '/admin/formbuilder/' | relative_url }}">Form Builder →</a>
</div>
