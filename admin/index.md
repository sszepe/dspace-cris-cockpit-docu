---
layout: page
title: Admin Guide
permalink: /admin/
---

# Administration Guide

![Requires](https://img.shields.io/badge/requires-DSpace%20Administrator-orange?style=flat-square)
![Django](https://img.shields.io/badge/some%20features-Django%20mode%20required-green?style=flat-square)

This guide covers all administrative features of the DSpace CRIS Cockpit. Administration is split across two interfaces with distinct roles and auth models.

## Admin Interfaces

<div class="callout callout-info">
<span class="callout-title">Config Cockpit is the primary admin tool</span>
All runtime configuration — clusters, quicklinks, collection mappings, form layouts, CRIS entity layouts, and feature flags — is managed through the <strong>Config Cockpit</strong> at <code>http://localhost:5174</code>. It uses Django staff credentials and does not require a DSpace login. See the <a href="{{ '/dev/django-frontend/' | relative_url }}">Config Cockpit developer reference</a> for technical details.
</div>

| Interface | URL | Auth | Purpose |
|---|---|---|---|
| **Config Cockpit** | `:5174` | Django staff (session) | Full CRUD for all runtime config — **primary admin tool** |
| **Main Cockpit** | `:4000` | DSpace JWT | Read-only overviews of config + community/collection management |

The main Cockpit at `:4000` retains admin-only pages for **Communities & Collections** (creation and role management) because those operations act directly on DSpace via its REST API and require DSpace Administrator credentials. All other configuration management has moved to the Config Cockpit.

## Sections

| Section | Contents |
|---|---|
| [Roles & Permissions]({{ '/admin/roles/' | relative_url }}) | Role definitions for both interfaces, permission matrix |
| [Feature Flags]({{ '/admin/flags/' | relative_url }}) | Three-gate system, flag reference, Config Cockpit Site Settings |
| [Dashboard Clusters]({{ '/admin/clusters/' | relative_url }}) | Manage clusters via Config Cockpit; read-only overview in main Cockpit |
| [Quicklinks]({{ '/admin/quicklinks/' | relative_url }}) | Manage presets via Config Cockpit; read-only overview in main Cockpit |
| [Collection Mappings]({{ '/admin/collection-mappings/' | relative_url }}) | Entity type → collection routing rules (Config Cockpit) |
| [Form Builder]({{ '/admin/formbuilder/' | relative_url }}) | Form layout overlays (Config Cockpit); read-only in main Cockpit |
| [CRIS Layout]({{ '/admin/cris-layout/' | relative_url }}) | Entity detail page layout editor (Config Cockpit) |
| [Communities]({{ '/admin/communities/' | relative_url }}) | Community/collection creation and role management (main Cockpit only) |

<div class="callout callout-warn">
<span class="callout-title">Django mode required for runtime config</span>
All Config Cockpit features require <code>VITE_CLUSTER_CONFIG_SOURCE=django</code> and <code>VITE_QUICKLINKS_CONFIG_SOURCE=django</code>. In TS mode, edit the static config files and redeploy. The Config Cockpit itself always requires Django to be running.
</div>

<div class="page-nav">
  <span></span>
  <a href="{{ '/admin/roles/' | relative_url }}">Roles & Permissions →</a>
</div>
