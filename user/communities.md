---
layout: page
title: Communities & Collections
permalink: /user/communities/
parent: User Guide
---

# Communities & Collections

The **Communities** tab (`#/communities`) shows the full hierarchy of the repository — communities, sub-communities, and collections. Browse it to explore the repository structure or to find items within a specific collection.

---

## Browsing the Hierarchy

<div class="ui-mock">
  <div class="mock-titlebar">
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#e05252;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#f59e0b;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#3ecf8e;margin-right:8px;"></span>
    <span style="font-size:12px;color:#7a8599;">Communities</span>
  </div>
  <div style="padding:16px;font-size:13px;">
    <div style="display:flex;align-items:center;gap:8px;padding:8px 0;color:#60a5fa;font-weight:500;cursor:pointer;">
      <span>▼</span> 🏛 Faculty of Natural Sciences
    </div>
    <div style="padding-left:24px;">
      <div style="display:flex;align-items:center;gap:8px;padding:6px 0;color:#e2e8f0;cursor:pointer;">
        📁 Publications
        <span style="font-size:11px;color:#4a5568;margin-left:auto;">142 items</span>
        <span style="font-size:12px;color:#60a5fa;cursor:pointer;margin-left:8px;">Browse →</span>
      </div>
      <div style="display:flex;align-items:center;gap:8px;padding:6px 0;color:#e2e8f0;cursor:pointer;">
        📁 Theses &amp; Dissertations
        <span style="font-size:11px;color:#4a5568;margin-left:auto;">67 items</span>
        <span style="font-size:12px;color:#60a5fa;cursor:pointer;margin-left:8px;">Browse →</span>
      </div>
      <div style="display:flex;align-items:center;gap:8px;padding:6px 0;color:#e2e8f0;cursor:pointer;">
        📁 Research Data
        <span style="font-size:11px;color:#4a5568;margin-left:auto;">31 items</span>
        <span style="font-size:12px;color:#60a5fa;cursor:pointer;margin-left:8px;">Browse →</span>
      </div>
    </div>
    <div style="display:flex;align-items:center;gap:8px;padding:8px 0;color:#60a5fa;font-weight:500;cursor:pointer;">
      <span>▶</span> 🏛 Centre for Environmental Research
    </div>
    <div style="display:flex;align-items:center;gap:8px;padding:8px 0;color:#60a5fa;font-weight:500;cursor:pointer;">
      <span>▶</span> 🏛 Institute of Digital Humanities
    </div>
  </div>
</div>

- Click a **community name** or the **▶** arrow to expand and see sub-communities and collections inside it.
- Click **Browse →** or a collection name to navigate into that collection and see its published items.
- Click **▼** on an expanded community to collapse it.

---

## Viewing a Collection's Items

Clicking a collection opens the **Item List** — a paginated table of all published items in that collection.

| Column | Description |
|---|---|
| Title | Item title, linked to the item detail page |
| Entity Type | The CRIS entity type (Publication, Person, Project, etc.) |
| Date Issued | Publication or creation date |

Click any row or the **View →** link to open the item detail page. Use **← Back** to return to the collection list.

<div class="callout callout-info">
<span class="callout-title">Only published items are shown</span>
The collection browser shows only <strong>archived (published)</strong> items. Workspace drafts are not visible here — find them in <a href="{{ '/user/workspace/' | relative_url }}">My Submissions</a>.
</div>

---

## Admin Actions

Administrators and Community Admins see additional action buttons that regular users do not:

| Button | Who sees it | What it does |
|---|---|---|
| **+ Create Community** | Administrators | Opens the Create Community modal |
| **+ Collection** | Admins / Community Admins | Opens the Create Collection modal for this community |
| **⚙ Manage Roles** | Administrators | Opens the Role Management modal for the community |
| **⚙** (on a collection) | Administrators | Opens the Collection Permissions modal |

See the [Admin Guide → Communities]({{ '/admin/communities/' | relative_url }}) for full details on these actions.

<div class="page-nav">
  <a href="{{ '/user/quicklinks/' | relative_url }}">← Quicklinks</a>
  <a href="{{ '/user/profile/' | relative_url }}">Profile →</a>
</div>
