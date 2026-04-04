---
layout: page
title: Admin Guide
permalink: /admin/
---

# Administration Guide

![Requires](https://img.shields.io/badge/requires-DSpace%20Administrator-orange?style=flat-square)
![Django](https://img.shields.io/badge/some%20features-Django%20mode%20required-green?style=flat-square)

This guide covers all administrative features of the DSpace CRIS Cockpit — feature flags, dashboard cluster management, quicklinks presets, form builder, and community/collection administration.

## Sections

| Section | Contents |
|---|---|
| [Roles & Permissions](/admin/roles/) | Role definitions, permission matrix, access pages |
| [Feature Flags](/admin/flags/) | Two-gate system, General Settings, all flag reference |
| [Dashboard Clusters](/admin/clusters/) | Create, manage and assign entity types to clusters |
| [Quicklinks](/admin/quicklinks/) | Presets, base filters, interactive facet filters |
| [Form Builder](/admin/formbuilder/) | Layout overlays, sections, field overrides, conditionals |
| [Communities](/admin/communities/) | Community/collection creation, role management |

<div class="callout callout-warn">
<span class="callout-title">Django mode required for runtime management</span>
The Clusters and Quicklinks management UIs only work when <code>VITE_CLUSTER_CONFIG_SOURCE=django</code> / <code>VITE_QUICKLINKS_CONFIG_SOURCE=django</code>. In TS mode, edit the static config files and redeploy.
</div>

<div class="page-nav">
  <span></span>
  <a href="{{ '/admin/roles/' | relative_url }}">Roles & Permissions →</a>
</div>
