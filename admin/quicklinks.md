---
layout: page
title: Quicklinks
permalink: /admin/quicklinks/
parent: Admin Guide
---

# Quicklinks Presets

![Requires](https://img.shields.io/badge/requires-VITE__QUICKLINKS__CONFIG__SOURCE%3Ddjango-green?style=flat-square)
![Managed in](https://img.shields.io/badge/managed%20in-Config%20Cockpit%20%3A5174-blue?style=flat-square)

Quicklinks are pre-configured faceted searches at `#/quicklinks`. Each **preset** targets one entity type and exposes **interactive filters** users can apply without typing a query.

<div class="callout callout-info">
<span class="callout-title">Manage presets in the Config Cockpit</span>
Navigate to <strong>Config Cockpit → Quicklink Presets</strong> (<code>http://localhost:5174</code>) to create, edit, and delete presets and their filters. The admin pages in the main Cockpit are transitioning to read-only overviews.
</div>

<div class="callout callout-warn">
<span class="callout-title">Main Cockpit quicklinks management is deprecated</span>
The <strong>Admin Settings → Quicklinks</strong> tab in the main Cockpit will become read-only in a future release. All write operations should use the Config Cockpit.
</div>

## Concepts

| Concept | Description | Example |
|---|---|---|
| **Preset** | One tab in the Quicklinks page. Always applies base filters. | "Publication" tab |
| **Base Filters** | DSpace facet filters always merged into the query, invisible to the user. | `{"entityType": ["Publication"]}` |
| **Interactive Filter** | A facet input shown to the user. Can be `text` or `date` kind. | "Author" → facet `dc.contributor.author` |

```mermaid
flowchart LR
    User["User visits\n#/quicklinks"]
    Tab["Selects a preset tab\ne.g. 'Publication'"]
    Base["Base filters always applied\nentityType=Publication"]
    UI["User fills in\ninteractive filters\n'Type: Journal Article'"]
    Query["Discovery query:\nentityType=Publication\n+ dc.type=Journal Article"]
    Results["Results list"]

    User --> Tab
    Tab --> Base
    Base --> Query
    UI --> Query
    Query --> Results
```

---

## Creating a Preset

Navigate to **Config Cockpit → Quicklink Presets** (`http://localhost:5174`). Click **+ New Preset**.

<ol class="steps">
<li>
<div><strong>Enter a label and key</strong><br>
The Label is the tab name shown to users. The key is auto-generated from the label but can be customised. Keys must be unique (e.g. <code>publication</code>, <code>my-entity</code>).</div>
</li>
<li>
<div><strong>Add an optional description and set sort order</strong><br>
Sort order controls the tab position. Toggle <strong>Enabled</strong> to show/hide the preset without deleting it.</div>
</li>
<li>
<div><strong>Save the preset</strong><br>
Click <strong>Save</strong> in the create modal. The preset appears in the list immediately.</div>
</li>
<li>
<div><strong>Add interactive filters</strong><br>
Select the new preset in the left panel to open the filter editor. See the next section.</div>
</li>
</ol>

### Editing a Preset

Click **Edit** on a preset card to open the edit modal. All fields can be updated.

### Deleting a Preset

Click **✕** on the preset card. This also deletes all filters belonging to that preset. **This cannot be undone.**

---

## Managing Filters

Interactive filters appear as search inputs in the Quicklinks page, allowing users to narrow results within a preset.

### Filter Fields

| Field | Required | Description |
|---|---|---|
| **Key** | ✅ | Unique within the preset, e.g. `dc.type`. Used internally. |
| **Label** | ✅ | Display label shown to the user, e.g. "Publication Type". |
| **Facet Name** | ✅ | The DSpace Discovery facet field. Must match `discovery.xml`. |
| **Kind** | ❌ | `text` (default) — free-text facet. `date` — year/date range picker. |
| **Placeholder** | ❌ | Hint text shown in the input when empty. |

<div class="callout callout-warn">
<span class="callout-title">Facet name must match discovery.xml</span>
The Facet Name must exactly match a Discovery facet configured in DSpace's <code>discovery.xml</code>. Common values:

<ul>
<li><code>dc.type</code></li>
<li><code>dc.date.issued</code></li>
<li><code>dc.contributor.author</code></li>
<li><code>dc.publisher</code></li>
<li><code>itemtype</code></li>
<li><code>entityType</code></li>
<li>CRIS fields: <code>crispj.investigator</code>, <code>person.affiliation.name</code>, <code>oairecerif.funder</code></li>
</ul>

Verify available facets at: <code>GET /server/api/discover/facets</code>
</div>

### Adding a Filter

Select a preset in the left panel of the Config Cockpit Quicklink Presets page. The right panel shows the filter editor. Click **+ Add Filter** to open the filter modal. Fill in the required fields and click **Save**.

### Removing a Filter

Click **Delete** on any filter row in the right panel. Removal is immediate and cannot be undone via the UI.

---

## Default Presets (TS mode)

These presets are included in `quicklinks-config.ts` as the static defaults:

| Preset | Key | Base Filter | Filters |
|---|---|---|---|
| Publication | `publication` | `entityType=Publication` | dc.type, dc.date.issued, dc.contributor.author |
| Project | `project` | `entityType=Project` | investigator, coordinator, status, startDate, endDate |
| Funding | `funding` | `entityType=Funding` | itemtype, funder |
| Person | `person` | `entityType=Person` | affiliation |
| OrgUnit | `orgunit` | `entityType=OrgUnit` | dc.type, country |
| Equipment | `equipment` | `entityType=Equipment` | itemtype |
| Event | `event` | `entityType=Event` | itemtype, startDate, endDate |
| Product | `product` | `entityType=Product` | dc.type, dc.contributor.author |
| Patent | `patent` | `entityType=Patent` | dc.contributor.author |
| Journal | `journal` | `entityType=Journal` | dc.publisher |

---

## Adding a Custom Preset (TS mode)

Edit `src/config/quicklinks-config.ts` and add to `QUICKLINKS_CRIS_PRESETS`:

```typescript
{
  key: "my-entity",
  label: "My Entity",
  description: "Browse My Entity records.",
  baseFilters: { entityType: ["MyEntity"] },
  filters: [
    { key: "dc.type",    label: "Type",   facetName: "dc.type" },
    { key: "dc.date.issued", label: "Year", facetName: "dc.date.issued", kind: "date" },
  ],
  sort_order: 11,
  enabled: true,
}
```

<div class="page-nav">
  <a href="{{ '/admin/clusters/' | relative_url }}">← Dashboard Clusters</a>
  <a href="{{ '/admin/formbuilder/' | relative_url }}">Form Builder →</a>
</div>
