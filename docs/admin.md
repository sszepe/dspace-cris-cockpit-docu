---
layout: page
title: Admin Guide
permalink: /admin/
---

*Introduction*

# Administration Guide

This guide covers all administrative tasks in the DSpace CRIS Cockpit — from toggling runtime feature flags and managing dashboard clusters to setting up communities, collections, and role permissions.

* 🔐 DSpace Administrator account required
* ⚙️ Django sidecar for runtime config

The Cockpit exposes three tiers of admin functionality. Some features require the optional **Django sidecar** to be running and configured — these are noted throughout the guide.

* Feature Flags — Toggle Quicklinks, community creation, and role management on or off at runtime.
* Dashboard Clusters — Group entity types into labelled sections shown on the dashboard.
* Quicklinks Presets — Define faceted search shortcuts for entity types with custom filters.
* Communities & Collections — Create communities and collections, manage admin groups and permissions.

*Introduction*

## Roles & Permissions

The Cockpit derives all roles from DSpace group membership — there is no separate admin database. Every feature is gated by at least one role check at the React layer.

### Role Definitions

| Role | How It's Detected | DSpace Group Name |
| --- | --- | --- |
| Administrator | Member of the global "Administrator" group | `Administrator` |
| Community Admin | Member of a community-specific admin group | `COMMUNITY_<uuid>_ADMIN` |
| Regular User | Any authenticated DSpace EPerson | — |

### Permission Matrix

| Feature | Admin | Com. Admin | User |
| --- | --- | --- | --- |
| Admin Settings page | ✓ | – | – |
| Toggle feature flags | ✓ | – | – |
| Manage dashboard clusters | ✓ | – | – |
| Manage quicklinks presets | ✓ | – | – |
| Create community | ✓ | – | – |
| Create collection | ✓ | Own communities | – |
| Manage community role groups | ✓ | Own communities | – |
| View quicklinks tab | ✓ | If not admin-only | If not admin-only |
| Access workspace | ✓ | ✓ | ✓ |

**ℹ️ Role sync** — Role changes made in DSpace take effect in the Cockpit after the user's next login. The group list is fetched once at login and cached for the session.

*Introduction*

## Accessing Admin Pages

Admin-only pages are accessible from the navigation bar once you are logged in as a DSpace Administrator. Non-admin users will not see these nav items.

| Page | URL Hash | Purpose |
| --- | --- | --- |
| Admin Settings | `#/admin/settings` | Feature flags, clusters, and quicklinks presets (3 tabs) |
| Manage Clusters | `#/admin/clusters` | Dedicated cluster management page |
| Form Builder | `#/admin/form-builder` | Submission form layout customisation |

**⚠️ Django mode required for some features** — The Admin Settings page cluster/quicklinks management require VITE\_CLUSTER\_CONFIG\_SOURCE=django (and/or VITE\_QUICKLINKS\_CONFIG\_SOURCE=django ). In ts mode these tabs show an informational message instead of the editor.

*Feature Flags*

## General Settings

Navigate to **Admin Settings → General** (`#/admin/settings`) to view and manage runtime feature flags. This tab shows both the build-time environment variable values and the live runtime DB state (in Django mode).

Each feature flag row shows:

* The feature name and description
* The environment variable controlling the build-time gate
* A toggle switch (only active in Django mode, only usable by admins)

**ℹ️ Two-gate system** — Each feature has a build-time gate (env var set at deployment) and optionally a runtime gate (Django DB toggle). Both must be enabled for the feature to appear. The runtime toggle has no effect if the build-time gate is off — you must redeploy to change that.

*Feature Flags*

## Quicklinks Toggle

The Quicklinks tab is hidden from all users by default. Enabling it requires clearing at least two gates.

### Gate 1 — Build-Time Master Switch

| Env Var | Value | Effect |
| --- | --- | --- |
| `VITE_QUICKLINKS_ENABLED` | `false` (default) | Quicklinks hidden for everyone; runtime toggle has no effect |
| `VITE_QUICKLINKS_ENABLED` | `true` | Feature active; audience determined by Gate 2 |

### Gate 2 — Audience Gate

| Env Var | Value | Effect |
| --- | --- | --- |
| `VITE_QUICKLINKS_ADMIN_ONLY` | `true` (default) | Quicklinks tab visible to Administrators only |
| `VITE_QUICKLINKS_ADMIN_ONLY` | `false` | Quicklinks tab visible to all authenticated users |

### Gate 3 — Runtime Toggle (Django mode only)

When `VITE_QUICKLINKS_CONFIG_SOURCE=django`, a third gate is controlled from **Admin Settings → General**. The *Quicklinks enabled* toggle writes to `SiteSettings.quicklinks_enabled` in the Django DB and takes effect immediately — no redeploy needed.

**✓ Typical setup** — Set VITE\_QUICKLINKS\_ENABLED=true and VITE\_QUICKLINKS\_ADMIN\_ONLY=true at deployment. Use the runtime toggle to enable/disable for admins without redeploying. When ready to open to all users, change VITE\_QUICKLINKS\_ADMIN\_ONLY=false and redeploy.

*Feature Flags*

## Communities Toggles

Three independent flags control which community/collection management actions are available. All are admin-only regardless of the flag value.

\*\*Community Creation\*\* — Shows the "Create Community" button on the Communities page. — `VITE\_COMMUNITIES\_CREATION\_ENABLED · DB: communities\_creation\_enabled`

\*\*Community Role Management\*\* — Shows the "Manage Roles" button on each community, opening the admin-group management modal. — `VITE\_COMMUNITIES\_ROLE\_MANAGEMENT\_ENABLED · DB: communities\_role\_management\_enabled`

\*\*Collection Creation\*\* — Shows the "Create Collection" button inside a community. Allows full admins (and community admins for their own communities) to add new collections. — `VITE\_COLLECTIONS\_CREATION\_ENABLED · DB: collections\_creation\_enabled`

*Dashboard*

## Dashboard Clusters

The Dashboard displays entity type counts grouped into **clusters** — named sections that each contain one or more entity types. For example, a "Research Output" cluster might contain Publication, Product, and Patent.

Research Output
3 entity types

Publication
Product
Patent

| Mode | How to change | Requires redeploy? |
| --- | --- | --- |
| **TypeScript mode** (ts) | Edit `src/config/entity-clusters.ts` | Yes |
| **Django mode** (django) | Admin Settings → Dashboard Clusters tab | No — changes are immediate |

**⚠️ Prerequisite** — Cluster management in the UI requires VITE\_CLUSTER\_CONFIG\_SOURCE=django . In ts mode the clusters tab shows an informational message.

### How Clusters Are Resolved

At login, the user's authorized entity types are fetched from DSpace and cross-referenced against the cluster config. Only entity types the user can access appear on the dashboard. Entity types not assigned to any cluster appear in an automatic "Other" group.

*Dashboard*

## Creating Clusters

Navigate to **Admin Settings → Dashboard Clusters** or **#/admin/clusters**.

1. **Enter a cluster label** —

   Type a descriptive name in the "Label" field of the "Create new cluster" form. A URL-safe key is auto-generated (e.g. "Research Output" → `research-output`).
2. **Add an optional description** —

   The description appears as a subtitle under the cluster heading on the dashboard.
3. **Click "Create cluster"** —

   The cluster is saved immediately to the Django database and appears in the list below with no entity types assigned yet.

### Editing an Existing Cluster

All cluster fields are editable inline — click into any field and modify the text, then click outside (blur) to auto-save. Label, Order, and Description all save on blur.

### Enabling / Disabling a Cluster

Use the **Active** toggle on each cluster card to soft-disable it without deleting it. Disabled clusters are hidden from the dashboard but remain in the database.

### Deleting a Cluster

Click the **Delete** button. A browser confirmation dialog appears. Deletion is permanent and cascades to all entity type entries within that cluster.

**⛔ Deletion is permanent** — There is no undo. If you might want the cluster again, use the Active toggle to disable it instead of deleting it.

*Dashboard*

## Assigning Entity Types

### Assigning an Entity Type to a Cluster

1. **Open the cluster card** —

   Find the cluster on the Admin Clusters page.
2. **Choose from the dropdown** —

   The "Add entity type…" dropdown lists only entity types not yet assigned to this cluster. Select the one to add.
3. **Click "Assign"** —

   The entity type is added immediately and appears as a pill in the cluster card. The dashboard reflects this on next load.

### Removing an Entity Type

Click the **×** button on any entity type pill within a cluster card. The removal is immediate.

### Sort Order

Entity types within a cluster are displayed in `sort_order` ascending order. Initial order follows the order of assignment. To reorder, use the API directly (`PATCH /api/dspace-config/entity-types/:id/`). In-UI drag-to-reorder is not yet supported.

*Quicklinks*

## Quicklinks Presets

Quicklinks are pre-configured faceted searches at `#/quicklinks`. Each **preset** targets one entity type and exposes **interactive filters** the user can apply.

Preset management requires `VITE_QUICKLINKS_CONFIG_SOURCE=django`. Navigate to **Admin Settings → Quicklinks Presets**.

| Concept | What it does | Example |
| --- | --- | --- |
| **Preset** | Defines the base entity type filter always applied. Shown as a tab in the Quicklinks page. | Preset "Publication" always filters entityType=Publication |
| **Base Filters** | Additional DSpace facet filters always merged into the query, invisible to the user. | `{"entityType": ["Publication"], "dc.language": ["en"]}` |
| **Interactive Filter** | A facet input shown to the user to narrow results. Can be text or date kind. | "Author" filter → facet field `dc.contributor.author` |

*Quicklinks*

## Creating Presets

1. **Open Quicklinks Presets tab** —

   Navigate to **Admin Settings → Quicklinks Presets**.
2. **Fill in the "Create new preset" form** —

   Enter a **Label** (the tab name), a **Key** (auto-generated, must be unique), and an optional **Description**.
3. **Set the Base Filters** —

   Enter a JSON object of always-applied DSpace Discovery facet filters:

   ```
   {"entityType": ["Publication"]}
   ```

   Multiple filters can be combined:

   ```
   {"entityType": ["Project"], "oairecerif.project.status": ["finished"]}
   ```
4. **Click "Create preset"** —

   The preset is saved to the Django database and becomes visible in the Quicklinks tab immediately (if the feature is enabled).

### Editing a Preset

Preset fields (label, description, base filters) are editable inline. The **Base filters** JSON field requires clicking the explicit **Save** button next to it for validation. All other fields auto-save on blur.

### Enabling / Disabling a Preset

Use the **Active** checkbox on each preset card. Disabled presets are hidden from the Quicklinks tab but remain in the database.

*Quicklinks*

## Managing Filters

Interactive filters appear as search inputs in the Quicklinks page, allowing users to narrow results within a preset.

| Field | Required | Description |
| --- | --- | --- |
| **Key** | Yes | Unique identifier within the preset, e.g. `dc.type` |
| **Label** | Yes | Display name shown above the filter input, e.g. "Publication Type" |
| **Facet Name** | Yes | The DSpace Discovery facet field to filter on. Must match a configured Discovery facet in your DSpace instance. |
| **Kind** | No | `text` (default) — text/autocomplete input. `date` — year/date picker. |
| **Placeholder** | No | Hint text shown inside the filter input when empty. |

### Adding a Filter

On any preset card, use the **"Add filter"** section at the bottom. Fill in Key, Label, and Facet Name (all required), then click **Add filter**. The filter is saved immediately.

### Removing a Filter

Click the **×** on any filter row. Removal is immediate.

**ℹ️ Finding the right facet name** — The Facet Name must exactly match a Discovery facet configured in DSpace's discovery.xml . Common values: dc.type , dc.date.issued , dc.contributor.author , itemtype , entityType , and CRIS-specific fields like crispj.investigator or person.affiliation.name .

*Communities*

## Creating Communities

Creating communities requires the **Administrator** role and the `communities_creation_enabled` flag to be on.

1. **Navigate to Communities** —

   Click **Communities** in the main navigation (`#/communities`). Admins see a "Create Community" button at the top.
2. **Open the Create Community modal** —

   Click **+ Create Community**. A modal opens with a form for the community name and optional metadata fields.
3. **Enter details and submit** —

   Fill in at least the **Name** (required). Optional: introductory text (`dc.description.abstract`), short description, copyright text, and sidebar text. Click **Create**.
4. **Community appears in the tree** —

   The community is created via `POST /api/core/communities` and appears immediately in the tree. The page auto-refreshes.

**ℹ️ Sub-communities** — To create a sub-community, expand the parent community in the tree first, then use the community-level create action. This passes the parent community ID to POST /api/core/communities?parent=<uuid> .

*Communities*

## Creating Collections

Collections live within communities and hold items. Requires `collections_creation_enabled` to be on.

1. **Navigate to the parent community** —

   Expand the community tree and find the community to nest the collection in.
2. **Click "+ Collection"** —

   Admin users see a "+ Collection" action next to each community. Click it to open the Create Collection modal.
3. **Fill in collection details** —

   Enter the collection **Name** (required). Optional: abstract, short description, copyright text, provenance, and sidebar text.
4. **Submit** —

   The collection is created via `POST /api/core/collections?parent=<community-uuid>` and appears nested under the parent community.

*Communities*

## Role Management

Each community has an optional **admin group** named `COMMUNITY_<uuid>_ADMIN`. Members get community-admin privileges in the Cockpit. The Role Management feature lets you create this group and manage its membership.

**ℹ️ Prerequisite** — The communities\_role\_management\_enabled flag must be on. Only the global DSpace Administrator can modify community role groups.

1. **Open the Role Management modal** —

   Find the community in the tree and click the **⚙ Manage Roles** button. The modal shows the community's current admin group status.
2. **Create the group if it doesn't exist** —

   If no admin group exists yet, click **Create admin group** to create `COMMUNITY_<uuid>_ADMIN` via the DSpace API.
3. **Add groups to the admin group** —

   Use the **Add group** section to search for and add existing DSpace groups. The search uses DSpace's `isNotMemberOf` query to show only unassigned groups. Results are paginated.
4. **Remove groups** —

   Each current member group has a **Remove** button. Removing a group revokes community-admin access for all members of that group.

### Collection Permissions

The **Manage Permissions** modal for a collection (opened via the ⚙ icon on a collection row) allows admins to view and manage the default read group and submitters group, calling the DSpace `resourcepolicies` API.

*Users & Policy*

## User Profiles

All authenticated users can view their own profile at `#/profile`. The profile page displays:

* Full name, email, and NetID from the DSpace EPerson record
* Account status flags (can log in, requires certificate, self-registered)
* Last active timestamp
* All DSpace group memberships
* Role badges — Administrator and Community Admin roles are highlighted
* List of community admin UUIDs (for Community Admins)

The profile page is read-only — user details are managed directly in DSpace (via the DSpace admin UI or REST API).

*Users & Policy*

## End-User Agreement

The Cockpit can display a mandatory Terms of Use (ToU) gate that users must accept before accessing the application. This is disabled by default.

### Enabling the Feature

```
VITE_ENABLE_END_USER_AGREEMENT=true
VITE_ENABLE_PRIVACY_STATEMENT=true   # optional privacy notice
```

### How It Works

1. **User logs in** —

   The app checks the EPerson's metadata for `dspace.agreements.end-user = "true"`.
2. **Gate shown if not accepted** —

   If the agreement has not been accepted, the ToU modal is shown instead of the application. The user cannot proceed until they accept.
3. **Terms text fetched from DSpace** —

   The ToU text is read from the DSpace Site object's `dc.rights` metadata field. The app tries the user's language first, then falls back to English. Set this text in the DSpace admin interface.
4. **Acceptance recorded on the EPerson** —

   When the user clicks "Accept", a PATCH request sets `dspace.agreements.end-user = "true"` on the EPerson. The gate is not shown again on subsequent logins.

**⚠️ Setting the Terms text** — The ToU text must be set in DSpace under the Site item's dc.rights metadata. If this field is empty, the modal shows a fallback message. Update it via the DSpace administrative interface or REST API.

*Reference*

## Configuration Source Modes

Dashboard Clusters and Quicklinks Presets each support TypeScript mode (static deploy-time config) and Django mode (runtime DB-backed config).

| Feature | Mode Env Var | TS mode | Django mode |
| --- | --- | --- | --- |
| Dashboard Clusters | `VITE_CLUSTER_CONFIG_SOURCE` | Reads from `entity-clusters.ts` | Reads from Django; editable via UI |
| Quicklinks Presets | `VITE_QUICKLINKS_CONFIG_SOURCE` | Reads from `quicklinks-config.ts` | Reads from Django; editable via UI |

### Choosing a Mode

| Situation | Recommended mode |
| --- | --- |
| Small team, infrequent config changes, no Django sidecar | `ts` |
| Admins need to change clusters/presets without developer involvement | `django` |
| High-availability, no external runtime dependencies | `ts` |
| Multiple environments (staging/prod) with different cluster configs | `django` |

### Fallback Behaviour

In Django mode, if the Django sidecar is unreachable, the app automatically falls back to the TypeScript static config (unless disabled with `VITE_CLUSTER_CONFIG_FALLBACK=false`). A warning is logged to the browser console when fallback occurs.

*Reference*

## Troubleshooting

### Admin menu items not visible

Admin nav items only appear when `isAdmin = true`. Your DSpace user must be a member of the `Administrator` group. Verify in the DSpace admin interface under *Access Control → Groups → Administrator*. Log out and back in after making group changes.

### "Cluster management requires VITE\_CLUSTER\_CONFIG\_SOURCE=django"

Django mode is not enabled. Set `VITE_CLUSTER_CONFIG_SOURCE=django` and redeploy, or edit the static TypeScript file instead.

### Feature flag toggles are greyed out

Toggles are only active when: (1) you are an Administrator, and (2) the relevant config source is set to `django`. In `ts` mode the toggle is displayed but disabled.

### Django auth probe shows "jwt\_received: false"

Nginx is not forwarding the `Authorization` header to the Django upstream. Ensure your nginx config includes `proxy_set_header Authorization $http_authorization;`.

### Community / Collection create buttons missing

Check: (1) you are a DSpace Administrator, and (2) the relevant flag (`communities_creation_enabled` / `collections_creation_enabled`) is on.

### End-user agreement modal keeps appearing

The PATCH to write `dspace.agreements.end-user = "true"` on the EPerson may be failing. Check the browser console for a failed PATCH to `/api/eperson/epersons/:id`.

### Quicklinks tab not visible after enabling

All three conditions must be true: (1) `VITE_QUICKLINKS_ENABLED=true`, (2) user meets the audience gate, (3) `SiteSettings.quicklinks_enabled=true` in DB (Django mode). Check each gate in order.

### Getting support

Use the debug endpoint to diagnose Django ↔ DSpace connectivity: open `/api/dspace-config/debug/auth/` while logged in. The JSON response shows exactly where the auth chain breaks down. Share this output when reporting issues.
