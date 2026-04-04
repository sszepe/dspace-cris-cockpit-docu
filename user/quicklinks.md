---
layout: page
title: Quicklinks
permalink: /user/quicklinks/
parent: User Guide
---

# Quicklinks

Quicklinks (`#/quicklinks`) are **pre-configured search shortcuts** for common entity types. If enabled by your administrator, they provide a faster, more focused search experience than the general Search tab — no query required.

<div class="callout callout-info">
<span class="callout-title">Quicklinks may not be visible</span>
Quicklinks is an optional feature. Your administrator may have disabled it, or restricted it to administrators only. If you do not see the Quicklinks tab, contact your repository manager.
</div>

---

## How Quicklinks Work

Each **preset** is a tab that targets one entity type and automatically applies a base filter — for example, the Publication preset always restricts results to items of type Publication. You then use the filter fields to narrow further.

<div class="ui-mock">
  <div class="mock-titlebar">
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#e05252;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#f59e0b;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#3ecf8e;margin-right:8px;"></span>
    <span style="font-size:12px;color:#7a8599;">Quicklinks</span>
  </div>
  <div style="padding:0 16px 16px;">
    <div style="display:flex;gap:0;border-bottom:1px solid #2a3145;margin-bottom:16px;">
      <div style="padding:10px 16px;font-size:12px;font-weight:600;color:#3ecf8e;border-bottom:2px solid #3ecf8e;cursor:pointer;">Publication</div>
      <div style="padding:10px 16px;font-size:12px;color:#7a8599;cursor:pointer;">Project</div>
      <div style="padding:10px 16px;font-size:12px;color:#7a8599;cursor:pointer;">Person</div>
      <div style="padding:10px 16px;font-size:12px;color:#7a8599;cursor:pointer;">OrgUnit</div>
      <div style="padding:10px 16px;font-size:12px;color:#7a8599;cursor:pointer;">Funding</div>
    </div>
    <div style="display:grid;grid-template-columns:1fr 1fr 1fr;gap:10px;margin-bottom:16px;">
      <div>
        <div style="font-size:11px;font-weight:600;color:#4a5568;text-transform:uppercase;letter-spacing:0.06em;margin-bottom:5px;">Type</div>
        <div style="background:#13161d;border:1px solid #2a3145;border-radius:6px;padding:7px 10px;font-size:12px;color:#7a8599;">Any type…</div>
      </div>
      <div>
        <div style="font-size:11px;font-weight:600;color:#4a5568;text-transform:uppercase;letter-spacing:0.06em;margin-bottom:5px;">Year</div>
        <div style="background:#13161d;border:1px solid #2a3145;border-radius:6px;padding:7px 10px;font-size:12px;color:#7a8599;">Any year…</div>
      </div>
      <div>
        <div style="font-size:11px;font-weight:600;color:#4a5568;text-transform:uppercase;letter-spacing:0.06em;margin-bottom:5px;">Author</div>
        <div style="background:#13161d;border:1px solid #2a3145;border-radius:6px;padding:7px 10px;font-size:12px;color:#7a8599;">Any author…</div>
      </div>
    </div>
    <div style="font-size:12px;color:#7a8599;margin-bottom:10px;">Showing all Publications · 247 results</div>
    <table style="width:100%;border-collapse:collapse;font-size:12px;">
      <tbody>
        <tr style="border-bottom:1px solid rgba(42,49,69,0.5);">
          <td style="padding:9px 10px;color:#60a5fa;cursor:pointer;">Precision medicine approaches in rare diseases</td>
          <td style="padding:9px 10px;color:#7a8599;">Journal Article · Smith, J. · 2024</td>
          <td style="padding:9px 10px;color:#60a5fa;white-space:nowrap;">View →</td>
        </tr>
        <tr style="border-bottom:1px solid rgba(42,49,69,0.5);">
          <td style="padding:9px 10px;color:#60a5fa;cursor:pointer;">Open data practices in the humanities</td>
          <td style="padding:9px 10px;color:#7a8599;">Conference Paper · Lee, M. · 2023</td>
          <td style="padding:9px 10px;color:#60a5fa;white-space:nowrap;">View →</td>
        </tr>
        <tr>
          <td style="padding:9px 10px;color:#60a5fa;cursor:pointer;">Algorithmic fairness in public sector AI</td>
          <td style="padding:9px 10px;color:#7a8599;">Preprint · Müller, A. · 2024</td>
          <td style="padding:9px 10px;color:#60a5fa;white-space:nowrap;">View →</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

---

## Switching Between Presets

Click a **tab** at the top of the Quicklinks page to switch to that entity type. Each preset automatically filters results to its entity type and shows the relevant filter fields for that type.

Default presets (if configured by your institution):

| Tab | Shows | Filter fields |
|---|---|---|
| Publication | Publications only | Type, Year, Author |
| Project | Projects only | Investigator, Coordinator, Status, Start date, End date |
| Funding | Funding items only | Type, Funder |
| Person | Person profiles only | Affiliation |
| OrgUnit | Organisations only | Type, Country |
| Equipment | Equipment records only | Type |
| Event | Events only | Type, Start date, End date |

Your institution may have added, removed, or renamed these presets.

---

## Using Filter Fields

Type into any filter field to narrow results for the current preset. Filter fields are specific to each entity type — for example, Publications show an Author field, while Projects show an Investigator field.

Results update as you type. You do not need to press Enter.

### Combining filters

Filling in multiple filter fields narrows results further — each active filter is combined. For example, filtering by **Year = 2023** and **Author = Smith** shows only Publications by Smith from 2023.

### Clearing a filter

Delete the text in a filter field to remove that constraint. All results for the preset's entity type are shown again (unless other filters are still active).

---

## Opening an Item

Click any result row or the **View →** link to open the item detail page. Use the **← Back** button to return to Quicklinks with your filters intact.

<div class="page-nav">
  <a href="{{ '/user/search/' | relative_url }}">← Search</a>
  <a href="{{ '/user/communities/' | relative_url }}">Communities →</a>
</div>
