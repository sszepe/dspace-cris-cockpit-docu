---
layout: page
title: Search
permalink: /user/search/
parent: User Guide
---

# Search

The **Search** tab (`#/search`) provides full-text search across all published items in the repository using DSpace's Discovery search engine.

## Search Interface

<div class="ui-mock">
  <div class="mock-titlebar">
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#e05252;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#f59e0b;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#3ecf8e;margin-right:8px;"></span>
    <span style="font-size:12px;color:#7a8599;">Search</span>
  </div>
  <div style="padding:16px;">
    <div style="display:flex;align-items:center;gap:10px;background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:10px 14px;margin-bottom:14px;">
      <span style="color:#4a5568;">🔍</span>
      <span style="font-size:13px;color:#7a8599;">Search all items…</span>
    </div>
    <div style="display:grid;grid-template-columns:190px 1fr;gap:16px;">
      <div>
        <div style="font-size:11px;font-weight:600;color:#7a8599;text-transform:uppercase;letter-spacing:0.08em;margin-bottom:10px;">Filters</div>
        <div style="margin-bottom:14px;">
          <div style="font-size:11px;font-weight:600;color:#4a5568;text-transform:uppercase;letter-spacing:0.06em;margin-bottom:6px;">Entity Type</div>
          <div style="display:flex;flex-direction:column;gap:5px;font-size:12px;">
            <label style="color:#7a8599;cursor:pointer;display:flex;align-items:center;gap:6px;"><input type="checkbox" checked> Publication <span style="color:#4a5568;">(124)</span></label>
            <label style="color:#7a8599;cursor:pointer;display:flex;align-items:center;gap:6px;"><input type="checkbox"> Project <span style="color:#4a5568;">(38)</span></label>
            <label style="color:#7a8599;cursor:pointer;display:flex;align-items:center;gap:6px;"><input type="checkbox"> Person <span style="color:#4a5568;">(29)</span></label>
            <label style="color:#7a8599;cursor:pointer;display:flex;align-items:center;gap:6px;"><input type="checkbox"> OrgUnit <span style="color:#4a5568;">(14)</span></label>
          </div>
        </div>
        <div>
          <div style="font-size:11px;font-weight:600;color:#4a5568;text-transform:uppercase;letter-spacing:0.06em;margin-bottom:6px;">Date Issued</div>
          <div style="background:#13161d;border:1px solid #2a3145;border-radius:6px;padding:7px 10px;font-size:12px;color:#7a8599;">Any year…</div>
        </div>
      </div>
      <div>
        <div style="font-size:12px;color:#7a8599;margin-bottom:12px;">124 results for <strong style="color:#e2e8f0;">"climate"</strong></div>
        <table style="width:100%;border-collapse:collapse;font-size:12px;">
          <tbody>
            <tr style="border-bottom:1px solid rgba(42,49,69,0.5);">
              <td style="padding:9px 10px;color:#60a5fa;cursor:pointer;">Machine learning in climate science</td>
              <td style="padding:9px 10px;color:#7a8599;">Publication · 2024</td>
              <td style="padding:9px 10px;color:#60a5fa;white-space:nowrap;">View →</td>
            </tr>
            <tr style="border-bottom:1px solid rgba(42,49,69,0.5);">
              <td style="padding:9px 10px;color:#60a5fa;cursor:pointer;">Biodiversity monitoring using remote sensing</td>
              <td style="padding:9px 10px;color:#7a8599;">Publication · 2023</td>
              <td style="padding:9px 10px;color:#60a5fa;white-space:nowrap;">View →</td>
            </tr>
            <tr>
              <td style="padding:9px 10px;color:#60a5fa;cursor:pointer;">Quantum computing for drug discovery</td>
              <td style="padding:9px 10px;color:#7a8599;">Publication · 2024</td>
              <td style="padding:9px 10px;color:#60a5fa;white-space:nowrap;">View →</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</div>

---

## Using the Search Bar

Type any keyword, title, author name, or phrase and press **Enter** (or click the search icon). Results update automatically.

Search uses DSpace's **Discovery full-text index** — the same engine powering the standard DSpace UI. It searches across all metadata fields and, where configured, the full text of attached documents.

### Tips for better results

| Goal | Technique |
|---|---|
| Exact phrase | `"climate change resilience"` — wrap in double quotes |
| Author search | Type the author's last name, e.g. `Smith` |
| Narrow by year | Use the Date Issued facet on the left |
| Narrow by type | Tick one or more Entity Type checkboxes |
| Broaden results | Remove facet filters one by one |

---

## Using Facet Filters

The left panel shows **facets** — filter dimensions that help you narrow results without changing the search term.

| Common Facet | Use Case |
|---|---|
| Entity Type | Show only Publications, only Projects, etc. |
| Date Issued | Limit to items published in a specific year or year range |
| Author | Find all items by a specific author |
| Subject | Filter by subject classification or keyword |
| Language | Show only items in a specific language |

**Selecting a facet value** narrows results to only items matching that value. **Ticking multiple values** within the same facet broadens results (OR logic). **Combining across facets** narrows further (AND logic).

Click a selected facet value again to **deselect** it and widen the results.

---

## Sharing a Search

The URL in your browser's address bar updates as you search:

```
#/search/machine%20learning
```

Copy this URL and share it with a colleague. When they open it, they see the same search results (subject to their own access permissions for restricted items).

<div class="callout callout-info">
<span class="callout-title">Search only covers published items</span>
The Search tab searches DSpace's public Discovery index — only <strong>archived (published)</strong> items appear. Your draft workspace items are not included in search results.
</div>

<div class="page-nav">
  <a href="{{ '/user/workspace/' | relative_url }}">← Workspace</a>
  <a href="{{ '/user/quicklinks/' | relative_url }}">Quicklinks →</a>
</div>
