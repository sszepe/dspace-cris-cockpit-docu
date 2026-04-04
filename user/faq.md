---
layout: page
title: FAQ & Glossary
permalink: /user/faq/
parent: User Guide
---

# FAQ & Glossary

## Frequently Asked Questions

### Login & Access

**I can't log in — what should I do?**
Confirm you are using your DSpace account email and password (not a separate Cockpit account). If you have forgotten your password, use the password reset in the DSpace administration UI, or contact your repository manager.

**My session expired — where did my draft go?**
Workspace drafts are stored in DSpace, not in your browser. They do not expire. Log back in and find your item in **My Submissions** — it will be exactly where you left it.

**I don't see the Quicklinks tab.**
Quicklinks may be disabled or restricted to administrators only by your institution. Contact your repository manager if you expect to have access.

---

### Dashboard & Creating Items

**Why are some entity type buttons greyed out on the Dashboard?**
A greyed-out button means you do not have submit permission for any collection of that entity type. Contact your repository manager and ask them to add you to the appropriate submitter group for the relevant collection.

**I don't see a particular entity type at all.**
Either the entity type has not been assigned to any dashboard cluster by your administrator, or your institution does not use that entity type. Contact your repository manager.

**What happens if I click Save vs Submit?**
- **Save** keeps the item as a private draft in your workspace. Only you and admins can see it. You can continue editing and submit later.
- **Submit** sends the item into DSpace's review workflow (if one is configured). Once submitted, editing may be restricted depending on the workflow step.

---

### Workspace

**My submission shows a red error badge — what do I do?**
Click **View →** to open the item detail page. The validation errors list the specific fields that need attention (e.g. "dc.title is required"). Fix each flagged field, then try submitting again.

**Can I edit a published (Archived) item?**
Editing published items requires administrator or collection admin privileges. If you need to correct a published record, contact your repository manager.

**What does cloning a submission do?**
Clone creates a new workspace draft with the same metadata copied from the original. It is useful for creating several similar records without re-entering all fields. Attached files (bitstreams) are **not** copied — you need to re-upload files to the clone.

**I accidentally deleted a file. Can I recover it?**
No — file deletion via the Cockpit is immediate and permanent. Contact your repository administrator as soon as possible. They may be able to restore the file from a database backup if acted upon quickly.

---

### Search

**Search only covers published items — where are my drafts?**
Drafts (workspace items) are not indexed in Discovery. Find them in **Workspace → My Submissions**.

**Can I share a search link with a colleague?**
Yes. The URL updates as you search (e.g. `#/search/climate%20change`). Copy and share the URL. Your colleague will see the same results (subject to their access permissions).

**A facet value I expect is not showing — why?**
Facets only show values that exist in the current result set. If no items match a particular facet value after other filters are applied, that value disappears from the facet list. Try broadening your search first.

---

### Communities & Files

**I can see a community but no collections inside it.**
The community exists but either has no collections yet, or all its collections are empty. Contact your repository manager.

**Why can't I upload a file to an item?**
You need edit permissions for that item. If the upload area does not appear, you likely do not have edit access. For workspace items you own, the upload area should always appear. For published items, admin access is required.

---

## Glossary

| Term | Definition |
|---|---|
| **Archived** | An item that has been published to the DSpace repository. Publicly visible (subject to access policies). |
| **Bitstream** | A file attached to a repository item — for example, a PDF, dataset file, or image. |
| **Bundle** | A grouping of bitstreams within an item. The `ORIGINAL` bundle contains the primary content files. |
| **Cluster** | A named group of entity types shown as a card on the Dashboard. Configured by your administrator. |
| **Collection** | A container that holds repository items. Collections live within communities and have their own access policies and submission workflows. |
| **Community** | A top-level grouping in the repository hierarchy, typically representing a faculty, department, research centre, or institute. |
| **Discovery** | DSpace's faceted search engine, powered by Apache Solr. Powers both the Search tab and the Quicklinks filters. |
| **Entity Type** | The CRIS type classification of an item — e.g. Publication, Person, Project, OrgUnit, Equipment. |
| **EPerson** | A DSpace user account. Your profile, permissions, and group memberships are all stored on your EPerson record. |
| **Facet** | A filter dimension available in search — e.g. "Entity Type", "Date Issued", "Author". Multiple facets can be combined. |
| **Preset** | A pre-configured Quicklinks search tab targeting a specific entity type with predefined filter fields. |
| **Workspace Item** | An item that is still in draft or review state — not yet published. Visible only to the submitter, reviewers, and administrators. |
| **Withdrawn** | An item previously published but now removed from public access. Still visible to administrators. |

<div class="page-nav">
  <a href="{{ '/user/profile/' | relative_url }}">← Profile</a>
  <a href="{{ '/user/' | relative_url }}">↑ User Guide</a>
</div>
