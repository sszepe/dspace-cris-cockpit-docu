---
layout: page
title: Dashboard
permalink: /user/dashboard/
parent: User Guide
---

# Dashboard

The Dashboard (`#/dashboard`) is your starting point after login. It shows all entity types you are authorised to create, organised into named groups called **clusters** — configured by your administrator.

## Reading the Dashboard

<div class="ui-mock">
  <div class="mock-titlebar">
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#e05252;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#f59e0b;margin-right:4px;"></span>
    <span style="display:inline-block;width:10px;height:10px;border-radius:50%;background:#3ecf8e;margin-right:8px;"></span>
    <span style="font-size:12px;color:#7a8599;">Dashboard</span>
  </div>
  <div style="padding:20px;">
    <div style="font-size:20px;font-weight:700;color:#e2e8f0;margin-bottom:4px;">Dashboard</div>
    <div style="font-size:13px;color:#7a8599;margin-bottom:20px;">Welcome, Dr. Smith. Select what you want to create.</div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;">
      <div style="background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:14px;">
        <div style="font-size:13px;font-weight:700;color:#e2e8f0;margin-bottom:10px;">Research Outputs</div>
        <div style="display:flex;flex-wrap:wrap;gap:6px;">
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">Publication</span>
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">Product</span>
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(74,85,104,0.2);border:1px solid #2a3145;color:#4a5568;cursor:not-allowed;" title="No submit permission">Patent</span>
        </div>
      </div>
      <div style="background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:14px;">
        <div style="font-size:13px;font-weight:700;color:#e2e8f0;margin-bottom:10px;">People &amp; Organisations</div>
        <div style="display:flex;flex-wrap:wrap;gap:6px;">
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">Person</span>
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">OrgUnit</span>
        </div>
      </div>
      <div style="background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:14px;">
        <div style="font-size:13px;font-weight:700;color:#e2e8f0;margin-bottom:10px;">Projects &amp; Funding</div>
        <div style="display:flex;flex-wrap:wrap;gap:6px;">
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">Project</span>
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">Funding</span>
        </div>
      </div>
      <div style="background:#1c2132;border:1px solid #2a3145;border-radius:8px;padding:14px;">
        <div style="font-size:13px;font-weight:700;color:#e2e8f0;margin-bottom:10px;">Other</div>
        <div style="display:flex;flex-wrap:wrap;gap:6px;">
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(96,165,250,0.1);border:1px solid rgba(96,165,250,0.3);color:#60a5fa;cursor:pointer;">Equipment</span>
          <span style="padding:5px 11px;border-radius:6px;font-size:12px;font-weight:500;background:rgba(74,85,104,0.2);border:1px solid #2a3145;color:#4a5568;cursor:not-allowed;" title="No submit permission">Event</span>
        </div>
      </div>
    </div>
  </div>
</div>

### Button states

| Appearance | Meaning |
|---|---|
| **Coloured, clickable** | You have submit permission on at least one collection of this type. Click to create. |
| **Grey, not clickable** | No submit permission. Hover to see the tooltip confirming why. |
| **Not shown at all** | Entity type not assigned to any cluster by your administrator. |

<div class="callout callout-info">
<span class="callout-title">Why is a button greyed out?</span>
Submit permission is granted per-collection in DSpace. If you believe you should be able to create a particular entity type, contact your repository manager and ask them to add you to the appropriate submitter group for the relevant collection.
</div>

---

## Creating an Item

Click any active (coloured) entity type button to begin creating a record of that type. A creation form or modal opens.

<ol class="steps">
<li>
<div><strong>Click the entity type button</strong><br>
For example, click <strong>Publication</strong> to start a new publication record. The button navigates to the Quicklinks preset for that type where you can begin the submission workflow.</div>
</li>
<li>
<div><strong>Fill in the metadata fields</strong><br>
A form opens with all available fields. Fields marked with a red asterisk (<span style="color:#e05252;">*</span>) are required by DSpace and cannot be left empty. Filling in optional fields improves discoverability.</div>
</li>
<li>
<div><strong>Upload files if needed</strong><br>
Many entity types allow file attachments (bitstreams). Use the file upload section of the form to attach PDFs, datasets, or other files.</div>
</li>
<li>
<div><strong>Save or submit</strong><br>
Click <strong>Save</strong> to keep the item as a draft in your workspace, or <strong>Submit</strong> to enter it into the DSpace review workflow. Saved drafts appear in <a href="{{ '/user/workspace/' | relative_url }}">My Submissions</a>.</div>
</li>
</ol>

<div class="callout callout-warn">
<span class="callout-title">Validation errors on save</span>
If required fields are missing or formatted incorrectly when you try to submit, DSpace returns validation errors. These appear as a red badge on the item in My Submissions. Open the item to see exactly which fields need attention.
</div>

### Entity Types Reference

| Entity Type | Typical Use |
|---|---|
| Publication | Journal articles, conference papers, books, theses |
| Project | Research projects with start/end dates, investigators |
| Funding | Grants and funding programmes |
| Person | Researcher profiles |
| OrgUnit | Departments, institutes, research groups |
| Equipment | Research equipment and facilities |
| Product | Datasets, software, research outputs that are not publications |
| Patent | Patent records |
| Event | Conferences, workshops, symposia |
| Journal | Journal metadata records |

<div class="page-nav">
  <a href="{{ '/user/getting-started/' | relative_url }}">← Getting Started</a>
  <a href="{{ '/user/workspace/' | relative_url }}">Workspace →</a>
</div>
