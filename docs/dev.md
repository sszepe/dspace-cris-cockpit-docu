---
layout: page
title: Developer Guide
permalink: /dev/
---

*Introduction*

# DSpace CRIS Cockpit

A single-page React application layered on top of DSpace 7 / DSpace CRIS providing
an enhanced submission workspace, discoverable entity browsing, admin configuration,
and an optional Django sidecar for runtime-configurable features.

* React 18 + TypeScript
* Django 4.2 REST sidecar
* DSpace 7+ / CRIS backend
* Hash-based SPA routing

The Cockpit acts as a **thin UI shell** that authenticates against DSpace's native JWT auth endpoint, then proxies all repository operations directly to the DSpace REST API. An optional **Django sidecar** service provides a small configuration API for features that need DB-backed runtime settings (dashboard clusters, quicklink presets, site flags, form layouts).

* 🗂 Workspace — Browse "My Submissions" and "Submissions by Others", inspect workspace items, and trigger entity creation flows.
* 🏛 Communities & Collections — Hierarchical community/collection browser with admin-gated creation and role-management modals.
* 🔗 Quicklinks — Preset-based faceted search shortcuts for common entity types, configurable via TS or Django DB.
* ⚙️ Admin Settings — Runtime toggles for feature flags, dashboard cluster CRUD, form-layout builder — all admin-gated.

*System Design*

## Architecture

All services run inside Docker. Nginx acts as the public reverse proxy and routes traffic to the appropriate upstream based on path prefix.

```
Browser / SPA

→

nginx :80

Single entry point

          ├── /server/*              → DSpace Backend :8080

          ├── /api/dspace-config/*   → Django Sidecar :8000

          └── /* (static)            → React SPA (dist/)

DSpace CRIS :8080

↔

Solr :8983

Search index

Django :8000

↔

PostgreSQL (django_config)

Config DB

DSpace

↔

PostgreSQL (dspace)

Repository DB
```

## Component Responsibilities

| Component | Technology | Responsibility |
| --- | --- | --- |
| React SPA | React 18, TypeScript, Vite | UI shell — all data comes from DSpace REST or Django API |
| DSpace 7/CRIS | Java / Spring Boot | Repository backend — auth, item CRUD, workspace, bitstreams, entities |
| Django Sidecar | Django 4.2 + DRF | Runtime config API — clusters, presets, site settings, form layouts |
| Nginx | nginx | Reverse proxy, CORS, static file serving, path-based routing |
| PostgreSQL | PostgreSQL 15+ | Two databases: `dspace` (DSpace) and `django_config` (sidecar) |
| Solr | Apache Solr | Full-text search index used by DSpace; queried via Discovery API |

**Django is optional** — Set VITE\_CLUSTER\_CONFIG\_SOURCE=ts and VITE\_QUICKLINKS\_CONFIG\_SOURCE=ts to run with only static TypeScript config. Django is only required for runtime-editable configuration without redeployment.

*Getting Started*

## Quick Start

1. **Clone and install dependencies** —

   ```
   # Frontend
   npm install

   # Django sidecar (optional)
   pip install -r requirements.txt --break-system-packages
   ```
2. **Configure environment variables** —

   Create a `.env` file with at minimum:

   ```
   VITE_API_BASE_URL=http://localhost:8080/server
   VITE_CLUSTER_CONFIG_SOURCE=ts
   VITE_QUICKLINKS_ENABLED=true
   VITE_QUICKLINKS_ADMIN_ONLY=true
   ```
3. **Start with Docker Compose** —

   ```
   docker compose -f docker-compose_2024.yml up -d
   ```

   Starts DSpace, Solr, PostgreSQL, Nginx, and optionally the Django sidecar.
4. **Start the frontend dev server** —

   ```
   npm run dev
   # Opens at http://localhost:5173
   ```
5. **(Optional) Initialize Django sidecar** —

   ```
   python manage.py migrate
   python manage.py createsuperuser
   python manage.py import_plain_config input-forms.xml
   python manage.py loaddata initial_data.json
   ```

*Frontend*

## Project Structure

```
src/

├──
api/

# DSpace + Django API client modules

│   ├──
client.ts

# Base fetch wrapper (CSRF + JWT headers)

│   ├──
apiClient.ts

# Typed DSpace REST helpers

│   ├──
django-client.ts

# Django config API helpers (get/post/patch/delete)

│   ├──
person-api.ts

# /api/core/epersons + auth/status

│   ├──
bitstream-api.ts

# Bitstream upload/delete

│   ├──
community-api.ts

# Community list + create

│   ├──
collection-api.ts

# Collection list + items

│   └──
dspace.ts

# Discovery search / item detail

│
├──
auth/

│   ├──
AuthContext.tsx

# React context: login/logout/groups/roles

│   ├──
client.ts

# JWT + CSRF storage helpers

│   └──
storage.ts

# sessionStorage key constants

│
├──
config/

# Feature configuration modules

│   ├──
dashboard-config.ts

# Cluster loading: TS or Django mode

│   ├──
quicklinks-config.ts

# Quicklinks presets + feature flags

│   ├──
communities-config.ts
# Communities feature flags

│   ├──
entity-clusters.ts

# Static cluster definitions

│   └──
entities-config.ts

# Entity type → UI mapping

│
├──
navigation/

│   └──
hash.ts

# Hash-based router (parseHash + routes)

│
├──
pages/

│   ├──
dashboard.tsx

│   ├──
communities.tsx

│   ├──
workspace-list.tsx

│   ├──
item-list.tsx

│   ├──
item-detail.tsx

│   ├──
search-page.tsx

│   ├──
quicklinks-page.tsx

│   ├──
admin-clusters.tsx

│   ├──
admin-settings.tsx

│   └──
form-builder.tsx

│
├──
components/

# Shared UI components & modals

└──
styles/

# Global CSS (theme, nav, app-shell)
```

### Django Sidecar Layout

```
dspace_config/

├──
settings/base.py

# Shared Django settings

├──
settings/local.py

# Dev overrides (DEBUG=True)

└──
urls.py

# Root URL conf (mounts /api/dspace-config/)

api/

├──
models.py

# All DB models

├──
views.py

# API views (APIView subclasses)

├──
serializers.py

# DRF serializers

├──
urls.py

# App URL patterns

├──
authentication.py

# DSpaceJWTAuthentication backend

└──
import_plain_config.py

# Management cmd: parse DSpace XML forms
```

*Frontend*

## Routing

The SPA uses **hash-based routing** — no server-side route config needed. All navigation is driven by `window.location.hash`. The router lives in `navigation/hash.ts`.

### Route Definitions

| Hash Pattern | Route Name | Page Component |
| --- | --- | --- |
| #/dashboard | dashboard | Dashboard |
| #/communities | communities | CommunitiesPage |
| #/collection/:id | collection | ItemListPage |
| #/workspace/mine | workspace (mine) | WorkspaceListPage |
| #/workspace/others | workspace (others) | WorkspaceListPage |
| #/item/:uuid | item | ItemDetailPage |
| #/workspaceitem/:id | workspaceitem | ItemDetailPage (workspace mode) |
| #/search/:query | search | SearchPage |
| #/quicklinks/:preset | quicklinks | QuicklinksPage |
| #/profile | profile | ProfilePage |
| #/admin/clusters | adminClusters | AdminClustersPage |
| #/admin/settings | adminSettings | AdminSettingsPage |
| #/admin/form-builder/:proc | formBuilder | FormBuilderPage |
| #/login | login | LoginPage (unauthenticated only) |

### Navigation Helpers

```
// Use the typed routes object for all navigation
import { routes } from "../navigation/hash";

window.location.hash = routes.dashboard;       // "#/dashboard"
window.location.hash = routes.item(uuid);      // "#/item/<uuid>"
window.location.hash = routes.collection(id);  // "#/collection/<id>"
window.location.hash = routes.searchQuery(q);  // "#/search/<encoded>"
window.location.hash = routes.quicklinksPreset("publication");
```

*Frontend*

## Auth & Context

Authentication state is managed via React Context in `auth/AuthContext.tsx`. The `AuthProvider` wraps the entire app; consume state with `useAuth()`.

### useAuth() API

| Property / Method | Type | Description |
| --- | --- | --- |
| isAuthenticated | boolean | True after successful DSpace login |
| isLoading | boolean | True while restoring session from sessionStorage |
| isAdmin | boolean | Member of DSpace "Administrator" group |
| isCommunityAdmin | boolean | Member of any COMMUNITY\_<uuid>\_ADMIN group |
| communityAdminIds | ReadonlySet<string> | Set of community UUIDs this user administers |
| isCommunityAdminOf(id) | (id: string) → boolean | Check admin status for a specific community |
| canCreate(type) | (type: string) → boolean | User has submit permission on a collection of this entity type |
| eperson | EPerson | null | Full DSpace EPerson object with metadata |
| groups | Group[] | All DSpace groups the user belongs to |
| entityTypes | EntityType[] | Entity types from external authorization source |
| collectionEntityTypes | EntityType[] | Entity types with submit permission via a collection |
| login(u, p) | async fn | Authenticates against DSpace and populates state |
| logout() | async fn | Calls DSpace logout and clears sessionStorage |
| refreshUser() | async fn | Re-fetches the current user state |

### Login Flow

```
// 1. Fetch CSRF token from DSpace
const csrf = await ensureCsrfToken(true);

// 2. POST credentials to DSpace /api/authn/login
const res = await fetch(`${API}/api/authn/login`, {
  method: "POST",
  headers: { "X-XSRF-TOKEN": csrf },
  body: new URLSearchParams({ user, password }),
});

// 3. Store JWT from Authorization response header
const auth = res.headers.get("Authorization");
setStoredJwt(auth.replace(/^Bearer\s+/i, "").trim());

// 4. Load full profile (eperson, groups, entity types)
const next = await loadCurrentUser();
```

**Session storage** — The JWT is stored in sessionStorage under the key jwt . On page refresh the app restores the session via /api/authn/status . Expired tokens redirect to login.

### Role Detection

```
// isAdmin — direct group name match
const isAdmin = state.groups.some(g => g.name === "Administrator");

// isCommunityAdmin — regex match: COMMUNITY_<uuid>_ADMIN
const COMMUNITY_ADMIN_RE = /^COMMUNITY_([0-9a-f-]+)_ADMIN$/i;
```

*Frontend*

## Configuration System

Several features support two configuration sources: a **static TypeScript** definition (zero runtime dependencies) or a **Django database** (runtime-editable by admins). The source is selected per-feature via a `VITE_*_CONFIG_SOURCE` environment variable.

| Feature | Env Var | TS Source File | Django Endpoint |
| --- | --- | --- | --- |
| Dashboard Clusters | `VITE_CLUSTER_CONFIG_SOURCE` | `entity-clusters.ts` | `/dashboard-config/` |
| Quicklinks Presets | `VITE_QUICKLINKS_CONFIG_SOURCE` | `quicklinks-config.ts` | `/quicklinks/` |

### Fallback Behaviour

When Django mode is selected but the API call fails, the app falls back to the TS static config (controlled by `VITE_CLUSTER_CONFIG_FALLBACK` / `VITE_QUICKLINKS_CONFIG_FALLBACK`). Set to `false` to hard-fail instead of silently using static config.

### Runtime Window Injection

The Django base URL can be injected at container start time without rebuilding via a script tag in `index.html`:

```
<script>
  window.__DJANGO_CONFIG_API_BASE_URL__ = "/api/dspace-config";
  window.__CLUSTER_CONFIG_SOURCE__ = "django";
</script>
```

This takes priority over the `VITE_*` build-time variables.

*Frontend*

## Feature Flags

Feature availability is controlled by a combination of build-time env vars and (in Django mode) runtime DB settings. All gates must pass for a feature to be visible.

### Quicklinks — Three Gates

```
// Gate 1: VITE_QUICKLINKS_ENABLED=true  (build-time env)
// Gate 2: Role — !isQuicklinksAdminOnly() || isAdmin  (build-time env)
// Gate 3: SiteSettings.quicklinks_enabled  (runtime DB, django mode only)
const quicklinksAccessible =
  isQuicklinksEnabled() &&
  runtimeQuicklinks &&
  (!isQuicklinksAdminOnly() || isAdmin);
```

### Communities Feature Flags

| Env Var | DB Field (SiteSettings) | Controls |
| --- | --- | --- |
| VITE\_COMMUNITIES\_CREATION\_ENABLED | communities\_creation\_enabled | "Create Community" button (admin only) |
| VITE\_COMMUNITIES\_ROLE\_MANAGEMENT\_ENABLED | communities\_role\_management\_enabled | Role management UI (admin only) |
| VITE\_COLLECTIONS\_CREATION\_ENABLED | collections\_creation\_enabled | "Create Collection" button (admin only) |

**TS mode note** — Runtime DB toggles only take effect when VITE\_\*\_CONFIG\_SOURCE=django . In ts mode, only the build-time env vars apply.

*Backend · Django Sidecar*

## Django Sidecar Overview

The Django sidecar is a lightweight Django REST Framework application serving runtime-configurable settings for the Cockpit frontend. It does not interact with the repository directly — it only serves configuration consumed by the React SPA.

### Tech Stack

| Package | Version | Purpose |
| --- | --- | --- |
| Django | >=4.2, <5.0 | Web framework |
| djangorestframework | >=3.14 | REST API layer |
| django-cors-headers | >=4.3 | CORS support for React dev server |
| psycopg2-binary | >=2.9 | PostgreSQL driver |
| gunicorn | >=21.2 | Production WSGI server |
| drf-spectacular | >=0.27 | OpenAPI schema generation |
| whitenoise | >=6.6 | Static file serving from Gunicorn |
| requests | >=2.31 | Proxy calls to DSpace auth/status for JWT validation |

*Backend · Django Sidecar*

## Authentication

Django does not maintain its own user database. A custom `DSpaceJWTAuthentication` backend validates every request against DSpace's `/api/authn/status` endpoint.

```
# authentication.py
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
        return "Administrator" in self.groups
```

**Performance note** — Every Django API request makes one outbound HTTP call to DSpace to validate the JWT. Consider adding per-token response caching if latency becomes a concern.

### Debug Endpoint

Use `GET /api/dspace-config/debug/auth/` to diagnose Django ↔ DSpace connectivity:

```
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

*Backend · Django Sidecar*

## API Reference

All endpoints are under the prefix `/api/dspace-config/`. All require a valid DSpace JWT (`Authorization: Bearer <token>`). Write endpoints additionally require `is_admin = true`.

## Auth & Diagnostics

`GET` `/debug/auth/` — Django ↔ DSpace connectivity probe. Returns full diagnostic object. (Public)

## Dashboard Clusters

`GET` `/dashboard-config/` — Enabled clusters with entity type lists (frontend read).

`GET` `/clusters/` — List all clusters including disabled.

`POST` `/clusters/` — Create a cluster. (Admin)

`GET` `/clusters/:id/` — Single cluster detail.

`PATCH` `/clusters/:id/` — Update cluster (partial). (Admin)

`DELETE` `/clusters/:id/` — Delete cluster and its entity type entries. (Admin)

`POST` `/clusters/:id/entity-types/` — Add entity type entry to a cluster. (Admin)

`PATCH` `/entity-types/:id/` — Update entity type label or sort\_order. (Admin)

`DELETE` `/entity-types/:id/` — Remove entity type from cluster. (Admin)

## Site Settings

`GET` `/site-settings/` — Returns the singleton SiteSettings row (pk=1).

`PATCH` `/site-settings/` — Toggle quicklinks, communities, collections flags. (Admin)

## Quicklinks

`GET` `/quicklinks/` — All enabled presets with filters (frontend read).

`GET` `/quickpresets/` — List all presets (admin-editable view).

`POST` `/quickpresets/` — Create a preset. (Admin)

`PATCH` `/quickpresets/:id/` — Update preset metadata. (Admin)

`DELETE` `/quickpresets/:id/` — Delete preset (cascades to filters). (Admin)

`POST` `/quickpresets/:id/filters/` — Add a filter to a preset. (Admin)

`PATCH` `/quickpreset-filters/:id/` — Update filter. (Admin)

`DELETE` `/quickpreset-filters/:id/` — Remove filter from preset. (Admin)

## Submission Forms & Metadata

`GET` `/submission-forms/` — List all imported submission forms.

`GET` `/submission-forms/:id/` — Full form with all fields.

`GET` `/submission-forms/:id/fields/` — Fields annotated with metadata registry info.

`GET` `/form-layouts/` — List form layouts; filterable by form\_name, profile, collection UUID.

`POST` `/form-layouts/` — Create a form layout with sections and overrides. (Admin)

`PATCH` `/form-layouts/:id/` — Update a form layout. (Admin)

`DELETE` `/form-layouts/:id/` — Delete a form layout. (Admin)

`GET` `/metadata-schemas/` — All schemas; searchable with ?q=.

`GET` `/metadata-schemas/:id/` — Schema with all fields.

`GET` `/metadata-fields/` — Field autocomplete — ?q=dc.title&schema=dc (capped at 100).

## Audit

`GET` `/audit/field-usage/` — Cross-reference form fields vs. metadata registry; flags unknown fields.

`GET` `/audit/forms-summary/` — Summary of all imported forms: name, field count, required flags.

*Backend · Django Sidecar*

## Data Models

### EntityCluster / EntityTypeEntry

Dashboard cluster configuration — groups entity types into named sections on the dashboard.

| Field | Type | Notes |
| --- | --- | --- |
| key | CharField(100) | Unique slug identifier |
| label | CharField(200) | Display name shown on dashboard |
| description | TextField | Optional description text |
| sort\_order | PositiveIntegerField | Controls display order |
| enabled | BooleanField | Soft toggle — disabled clusters are hidden |
| entity\_types → | EntityTypeEntry[] | Nested: entity\_type\_label, sort\_order |

### QuickPreset / QuickPresetFilter

Quicklinks preset definitions — each preset is a saved faceted search with user-interactive filters.

| Field | Type | Notes |
| --- | --- | --- |
| key | CharField(100) | Unique slug, matches entity type name |
| label | CharField(200) | Tab label in the Quicklinks page |
| base\_filters | JSONField | Always-applied filters, e.g. `{"entityType": ["Publication"]}` |
| enabled | BooleanField | Soft toggle |
| filters → | QuickPresetFilter[] | Nested: key, label, facet\_name, kind, placeholder |

### SiteSettings (singleton)

One row (PK=1). Runtime feature toggles consumed by the frontend on login.

| Field | Default | Controls |
| --- | --- | --- |
| quicklinks\_enabled | true | Master Quicklinks on/off toggle |
| communities\_creation\_enabled | true | "Create Community" button (admin only) |
| communities\_role\_management\_enabled | true | Role management UI (admin only) |
| collections\_creation\_enabled | true | "Create Collection" button (admin only) |

### FormLayout / FormSection / FormFieldOverride / FormConditionalBlock

Form layout customisation — overrides field labels, hints, visibility, and section grouping per collection or profile without modifying DSpace XML.

| Model | Key Fields | Purpose |
| --- | --- | --- |
| FormLayout | form\_name, profile, collection (UUID) | Targets a specific form + profile combination. Collection is optional (profile-wide vs collection-specific) |
| FormSection | key, label, sort\_order, collapsed\_by\_default | Groups fields into collapsible sections with optional helper text |
| FormFieldOverride | field\_name, label\_override, hint\_override, hidden | Per-field display customisation |
| FormConditionalBlock | trigger\_field, trigger\_value, revealed\_fields, revealed\_section | Conditionally shows/hides fields based on another field's value |

*DSpace Integration*

## DSpace REST API

The frontend communicates directly with DSpace 7's HAL-based REST API. All calls go through `api/client.ts` which injects the JWT and CSRF token automatically.

### Key Endpoints Used

| Endpoint | Used For |
| --- | --- |
| /api/authn/status | Session validation, eperson href retrieval |
| /api/authn/login | Username/password authentication → JWT |
| /api/authn/logout | Session invalidation |
| /api/core/epersons/:id | Full EPerson profile (name, email, metadata) |
| /api/core/epersons/:id/groups | Group memberships (admin/community-admin detection) |
| /api/core/entitytypes | Accessible entity types for dashboard clusters |
| /api/core/entitytypes/search/findAllByAuthorizedCollection | Entity types with submit permission (canCreate gate) |
| /api/discover/search/objects | Full-text + faceted Discovery search |
| /api/core/communities | Community list and creation |
| /api/core/collections | Collection list and creation |
| /api/submission/workspaceitems | Workspace item list and detail |
| /api/core/items/:uuid | Published item detail |
| /api/core/bundles/:id/bitstreams | Bitstream upload |
| /api/core/bitstreams/:id | Bitstream delete |

*DSpace Integration*

## JWT Auth Flow

DSpace 7 uses **stateless JWT auth** combined with a **CSRF token cookie** for write protection.

```
1.
 Browser →
GET /server/api/security/csrf

              ←
Set-Cookie: DSPACE-XSRF-COOKIE
 +
DSPACE-XSRF-TOKEN
 header

2.
 Browser →
POST /server/api/authn/login
 (user + password form-encoded)

              ←
Authorization: Bearer <JWT>

3.
 Subsequent requests →

Authorization: Bearer <JWT>

X-XSRF-TOKEN: <csrf>
 (required for POST / PATCH / DELETE)

4.

POST /server/api/authn/logout
 → token invalidated server-side
```

**CSRF rotation** — DSpace rotates the CSRF token on each successful mutation. The client reads the new value from the DSPACE-XSRF-TOKEN response header and updates sessionStorage automatically via setStoredCsrfToken() .

*DSpace Integration*

## Entity Types

DSpace CRIS extends the base model with typed entities. The Cockpit ships with quicklink presets for all standard CRIS entity types:

| Entity Type | Quicklink Preset | Default Filters |
| --- | --- | --- |
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
| Place, ConceptScheme, Concept, ArchivalResource | Institution-specific | Separate API modules; not in default quicklinks |

*Deployment*

## Docker Compose

The `docker-compose_2024.yml` file defines the full stack.

| Service | Image | Port | Notes |
| --- | --- | --- | --- |
| dspace | dspace/dspace | 8080 (internal) | DSpace backend |
| dspace-solr | dspace/dspace-solr | 8983 (internal) | Search index |
| dspacedb | postgres:15 | 5432 (internal) | Hosts both dspace and django\_config databases |
| nginx | nginx:alpine | 80 | Public entry point, static file serving |
| django | ./Dockerfile | 8000 (internal) | Config API sidecar (optional) |

### Nginx Path Routing

```
# Simplified nginx config
location /server/ {
  proxy_pass http://dspace:8080/server/;
}

location /api/dspace-config/ {
  proxy_pass http://django:8000/api/dspace-config/;
}

location / {
  root /usr/share/nginx/html;
  try_files $uri $uri/ /index.html;  # SPA fallback
}
```

*Deployment*

## Environment Variables

### Frontend (Vite / .env)

| Variable | Description |
| --- | --- |
| VITE\_API\_BASE\_URL | DSpace REST base URL (default: /server) |
| VITE\_DJANGO\_CONFIG\_API\_BASE\_URL | Django sidecar URL (default: empty) |
| VITE\_CLUSTER\_CONFIG\_SOURCE | ts or django (default: ts) |
| VITE\_CLUSTER\_CONFIG\_FALLBACK | Fall back to TS if Django fails (default: true) |
| VITE\_QUICKLINKS\_ENABLED | Master quicklinks on/off (default: false) |
| VITE\_QUICKLINKS\_ADMIN\_ONLY | Restrict to admins only (default: true) |
| VITE\_QUICKLINKS\_CONFIG\_SOURCE | ts or django (default: ts) |
| VITE\_QUICKLINKS\_CONFIG\_FALLBACK | Fall back to TS if Django fails (default: true) |
| VITE\_COMMUNITIES\_CREATION\_ENABLED | Enable community creation UI (default: true) |
| VITE\_COMMUNITIES\_ROLE\_MANAGEMENT\_ENABLED | Enable role management UI (default: true) |
| VITE\_COLLECTIONS\_CREATION\_ENABLED | Enable collection creation UI (default: true) |
| VITE\_END\_USER\_AGREEMENT\_ENABLED | Show Terms of Use gate on first login (default: false) |

### Django Sidecar

| Variable | Description |
| --- | --- |
| DJANGO\_SECRET\_KEY | Required in production. Django cryptographic secret key. |
| DEBUG | true / false (default: false) |
| DSPACE\_BASE\_URL | Internal DSpace URL for JWT validation (default: http://localhost:8080/server) |
| DB\_HOST / DB\_NAME / DB\_USER / DB\_PASSWORD / DB\_PORT | PostgreSQL connection parameters for django\_config database |
| CORS\_ALLOWED\_ORIGINS | Comma-separated allowed origins (default: http://localhost:4000) |
| ALLOWED\_HOSTS | Comma-separated Django ALLOWED\_HOSTS (default: localhost,127.0.0.1,django) |

*Deployment*

## Nginx & Proxy Notes

DSpace CRIS requires specific proxy headers for CSRF and trusted-proxy detection to work correctly.

```
# Required proxy headers for DSpace
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# Pass cookie attributes through
proxy_cookie_path / "/; SameSite=Lax";
```

**Trusted proxy ranges** — In local.cfg , set proxies.trusted.ipranges to include your Docker subnet (e.g. 172.23.0 ). Without this, DSpace will reject proxied requests or use incorrect client IPs.

**Development only** — Set rest.cors.cookie.secure = false in local.cfg for plain HTTP development. This must never be used in production — use HTTPS and remove this override.

*Reference*

## Quicklinks System

Quicklinks are saved faceted search presets at `#/quicklinks`. Each preset targets a specific entity type and exposes interactive facet filters without requiring free-text queries.

### QuickPreset Shape (TypeScript)

```
type QuickPreset = {
  key:         string;                    // e.g. "publication"
  label:       string;                    // Tab label
  description: string;
  baseFilters: Record<string, string[]>;  // Always-on search filters
  filters:     QuickFilterConfig[];       // User-interactive facets
  sort_order:  number;
  enabled:     boolean;
};

type QuickFilterConfig = {
  key:          string;           // Unique key within preset
  label:        string;           // UI label
  facetName:    string;           // DSpace Discovery facet field
  kind:         "text" | "date"; // Input widget type
  placeholder?: string;
};
```

### Adding a Custom Preset (TS mode)

Edit `config/quicklinks-config.ts`, add to `QUICKLINKS_CRIS_PRESETS`:

```
{
  key: "my-entity",
  label: "My Entity",
  baseFilters: { entityType: ["MyEntity"] },
  filters: [
    { key: "dc.type", label: "Type", facetName: "dc.type" },
  ],
  sort_order: 11,
  enabled: true,
}
```

*Reference*

## Form Builder

The Form Builder (`#/admin/form-builder`) is an admin-only tool for customising DSpace submission forms without editing XML. It reads parsed form definitions from Django and allows creation of **FormLayout** overrides.

1. **Import DSpace input-forms.xml** —

   ```
   python manage.py import_plain_config /path/to/input-forms.xml
   ```

   Parses the XML and populates `SubmissionForm` + `SubmissionFormField` models.
2. **Import metadata registry (for field autocomplete)** —

   ```
   python manage.py import_plain_config --metadata /path/to/registries/
   ```

   Populates `MetadataSchema` and `MetadataField`.
3. **Create FormLayout overrides** —

   Use the Form Builder UI or POST directly to `/form-layouts/` to customise label overrides, section grouping, and conditional field visibility per collection or profile.

### Layout Targeting

A layout is matched by `form_name` + `profile` + optional `collection` UUID. The most specific match wins: collection-specific overrides profile-wide, which overrides global.

*Reference*

## Submission API

Entity creation flows use DSpace's native submission API. The frontend provides typed modal wrappers for each entity type.

| Entity Type | API Module | Notes |
| --- | --- | --- |
| Publication / generic items | DSpace /submission/workspaceitems | Standard workspace submission flow |
| OrgUnit | orgunit-creation-api.ts | Direct item creation |
| Person | person-api.ts | Via workspace or import flow |
| Funding | funding-creation-api.ts | Includes funding-programme-api.ts |
| Equipment | equipment-creation-api.ts | Direct creation |
| Place | place-creation-api.ts / place-import-api.ts | Manual creation or Geonames import |
| SKOS Concept / ConceptScheme | skos-creation-api.ts / skos-edit-api.ts | Controlled vocabulary management |
| Archival Resource | archival-resource-api.ts | Archival metadata creation |
| OrgUnit (batch) | orgunit-import-api.ts | Batch import |

**Adding a new entity type** — Create a my-entity-creation-api.ts in src/api/ , a modal component in src/components/ , add it to the entity-clusters static config or Django DB, and wire the canCreate("MyEntity") gate from useAuth() .
