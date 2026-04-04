---
layout: page
title: Getting Started
permalink: /user/getting-started/
parent: User Guide
---

# Getting Started

## What is the Cockpit?

The DSpace CRIS Cockpit is a web interface for managing your institution's research information. Use it to:

<div class="service-grid">
  <div class="service-card">
    <h4>🗂 Workspace</h4>
    <p>Track your in-progress and submitted items. See validation status, continue drafts, and monitor workflow.</p>
  </div>
  <div class="service-card">
    <h4>🔍 Search</h4>
    <p>Full-text search across all published items with facet filters to narrow results by type, date, author, and more.</p>
  </div>
  <div class="service-card">
    <h4>🏛 Communities</h4>
    <p>Browse the repository hierarchy — communities, sub-communities, and collections — and view items within them.</p>
  </div>
  <div class="service-card">
    <h4>🔗 Quicklinks</h4>
    <p>Preset search shortcuts for common entity types (Publications, Projects, People, etc.). Visibility depends on your institution's configuration.</p>
  </div>
</div>

---

## Logging In

The Cockpit uses your existing DSpace account. There is no separate Cockpit password.

<ol class="steps">
<li>
<div><strong>Open the Cockpit URL</strong><br>
Navigate to your institution's Cockpit URL. You are redirected to the login page automatically if you are not already signed in.</div>
</li>
<li>
<div><strong>Enter your email and password</strong><br>
Use your DSpace account credentials. If you do not have an account, contact your institution's repository manager.</div>
</li>
<li>
<div><strong>Accept the Terms of Use (if shown)</strong><br>
If your institution has enabled the End User Agreement, the Terms of Use modal appears on your first login. Read the terms, tick the checkbox, and click <strong>Accept &amp; Continue</strong>. This only happens once — you will not be asked again on future logins. See <a href="{{ '/user/profile/#terms' | relative_url }}">Terms of Use</a> for details.</div>
</li>
<li>
<div><strong>The Dashboard appears</strong><br>
You are logged in. The Dashboard shows the entity types you are authorised to create. The navigation bar at the top gives access to all features.</div>
</li>
</ol>

<div class="callout callout-warn">
<span class="callout-title">Session is tab-scoped</span>
Your login session lives in the current browser tab. Closing the tab or browser window signs you out — you will need to log in again on the next visit. Your draft submissions are safely stored in DSpace and will be waiting for you.
</div>

### Logging Out

Click **Log out** in the top navigation bar. Your session is invalidated on the server immediately — your credentials are never stored in the browser.

---

## Navigation

<div class="ui-mock">
  <div class="mock-titlebar">
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#e05252;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#f59e0b;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#3ecf8e;margin-right:8px;"></span>
    <span style="font-size:12px;color:#7a8599;">your-institution.edu/cockpit</span>
  </div>
  <div style="background:#13161d;border-bottom:1px solid #2a3145;display:flex;align-items:center;padding:0 16px;gap:2px;">
    <div style="padding:10px 14px;font-size:12px;font-weight:600;color:#3ecf8e;border-bottom:2px solid #3ecf8e;">Dashboard</div>
    <div style="padding:10px 14px;font-size:12px;color:#7a8599;">Communities</div>
    <div style="padding:10px 14px;font-size:12px;color:#7a8599;">Workspace</div>
    <div style="padding:10px 14px;font-size:12px;color:#7a8599;">Search</div>
    <div style="padding:10px 14px;font-size:12px;color:#7a8599;">Quicklinks</div>
    <div style="padding:10px 14px;font-size:12px;color:#7a8599;margin-left:auto;">Profile</div>
    <div style="padding:10px 14px;font-size:12px;color:#7a8599;">Log out</div>
  </div>
  <div style="padding:14px 16px;font-size:13px;color:#7a8599;">Page content appears here</div>
</div>

| Tab | Route | Description |
|---|---|---|
| Dashboard | `#/dashboard` | Entity type tiles for quick item creation |
| Communities | `#/communities` | Repository hierarchy browser |
| Workspace | `#/workspace/mine` | Your in-progress and submitted items |
| Search | `#/search` | Full-text search across all published items |
| Quicklinks | `#/quicklinks` | Preset search shortcuts (if enabled by your admin) |
| Profile | `#/profile` | Your account details and group memberships |

<div class="callout callout-info">
<span class="callout-title">Not all tabs are always visible</span>
Your institution may have disabled certain features — for example, Quicklinks may be hidden or restricted to administrators. Administrators see additional tabs such as Admin Settings.
</div>

### Sub-navigation

The **Workspace** tab has two sub-views accessible via a secondary navigation bar:

- **My Submissions** — items you have created or are working on
- **Submissions by Others** — items submitted by colleagues that you have oversight of

<div class="page-nav">
  <a href="{{ '/user/' | relative_url }}">← User Guide</a>
  <a href="{{ '/user/dashboard/' | relative_url }}">Dashboard →</a>
</div>
