---
layout: page
title: Communities & Collections
permalink: /admin/communities/
parent: Admin Guide
---

# Communities & Collections

![Role](https://img.shields.io/badge/role-Administrator%20%2F%20Community%20Admin-orange?style=flat-square)
![Interface](https://img.shields.io/badge/interface-Main%20Cockpit%20%3A4000-purple?style=flat-square)

The Communities page (`#/communities`) shows the full repository hierarchy. Administrators and Community Admins see additional action buttons that regular users do not.

<div class="callout callout-info">
<span class="callout-title">Community & collection management stays in the main Cockpit</span>
Unlike clusters, quicklinks, and form layouts — which have moved to the Config Cockpit — community and collection creation and role management remain in the main Cockpit at <code>:4000</code>. These operations call the DSpace REST API directly and require a DSpace Administrator JWT, which the Config Cockpit does not use.
</div>

## Creating a Community

**Requires:** `Administrator` role + `VITE_COMMUNITIES_CREATION_ENABLED=true`

<ol class="steps">
<li>
<div><strong>Navigate to Communities</strong><br>
Click <strong>Communities</strong> in the main navigation. Admins see a "+ Create Community" button at the top of the page.</div>
</li>
<li>
<div><strong>Open the Create Community modal</strong><br>
Click <strong>+ Create Community</strong>. The modal opens with metadata fields.</div>
</li>
<li>
<div><strong>Enter details and submit</strong><br>
Fill in at minimum the <strong>Name</strong> (required — stored as <code>dc.title</code>). Optional: introductory text, short description, copyright text, sidebar text.</div>
</li>
<li>
<div><strong>Community appears in the tree</strong><br>
Created via <code>POST /api/core/communities</code>. The page refreshes automatically.</div>
</li>
</ol>

<div class="callout callout-info">
<span class="callout-title">Sub-communities</span>
To create a sub-community, expand the parent community in the tree first, then use the community-level create action. The parent UUID is passed as <code>POST /api/core/communities?parent=&lt;uuid&gt;</code>.
</div>

---

## Creating a Collection

**Requires:** `Administrator` (or Community Admin for own community) + `VITE_COLLECTIONS_CREATION_ENABLED=true`

<ol class="steps">
<li>
<div><strong>Navigate to the parent community</strong><br>
Expand the community tree to find the community you want the collection under.</div>
</li>
<li>
<div><strong>Click "+ Collection"</strong><br>
This action appears next to each community for users with the required permissions.</div>
</li>
<li>
<div><strong>Fill in collection details</strong><br>
Enter the <strong>Name</strong> (required). Optional: abstract, short description, copyright text, provenance, sidebar text.</div>
</li>
<li>
<div><strong>Submit</strong><br>
Created via <code>POST /api/core/collections?parent=&lt;community-uuid&gt;</code> and appears nested under the parent.</div>
</li>
</ol>

---

## Role Management

Each community can have an admin group named `COMMUNITY_<uuid>_ADMIN`. Members get Community Admin privileges in the Cockpit.

**Requires:** `Administrator` role + `VITE_COMMUNITIES_ROLE_MANAGEMENT_ENABLED=true`

```mermaid
flowchart TD
    Click["Admin clicks\n'Manage Roles' on a community"]
    Check{"COMMUNITY_uuid_ADMIN\ngroup exists?"}
    Create["Admin creates the group:\nPOST /api/core/groups"]
    Modal["Role Management modal opens\nShows current member groups"]
    Add["Add group:\nSearch DSpace groups\nPOST /api/core/groups/:id/subgroups"]
    Remove["Remove group:\nDELETE /api/core/groups/:id/subgroups/:sub_id"]

    Click --> Check
    Check -->|No| Create
    Create --> Modal
    Check -->|Yes| Modal
    Modal --> Add
    Modal --> Remove
```

### Steps

<ol class="steps">
<li>
<div><strong>Open the Role Management modal</strong><br>
Find the community in the tree and click <strong>⚙ Manage Roles</strong>.</div>
</li>
<li>
<div><strong>Create the admin group (if it doesn't exist)</strong><br>
Click <strong>Create admin group</strong> to create <code>COMMUNITY_&lt;uuid&gt;_ADMIN</code> via the DSpace API.</div>
</li>
<li>
<div><strong>Add groups to the admin group</strong><br>
Search for existing DSpace groups. Results show only groups not yet members (using DSpace's <code>isNotMemberOf</code> query). Results are paginated.</div>
</li>
<li>
<div><strong>Remove groups</strong><br>
Each current member group has a <strong>Remove</strong> button. Removal immediately revokes community-admin access for all members of that group.</div>
</li>
</ol>

<div class="callout callout-warn">
<span class="callout-title">Changes are immediate</span>
Role assignments call the DSpace REST API directly. There is no staging — changes take effect immediately for all members of the affected groups.
</div>

---

## Collection Permissions

The **Manage Permissions** modal (⚙ icon on a collection row) allows viewing and managing the default read group and submitters group via DSpace's `resourcepolicies` API.

| Group | Purpose |
|---|---|
| `COLLECTION_<uuid>_SUBMIT` | Users who can submit items to this collection |
| `COLLECTION_<uuid>_ADMIN` | Admins with full control over this collection |
| `COLLECTION_<uuid>_WORKFLOW_STEP_1/2/3` | Reviewers in the DSpace workflow |

---

## Troubleshooting

**"Create Community" button missing**
- Confirm you are a DSpace Administrator
- Check `VITE_COMMUNITIES_CREATION_ENABLED=true` (and runtime DB toggle if in django mode)

**"Create Collection" button missing inside a community**
- Check `VITE_COLLECTIONS_CREATION_ENABLED=true`
- Community Admins can only create collections in their own communities

**"Manage Roles" button missing**
- Check `VITE_COMMUNITIES_ROLE_MANAGEMENT_ENABLED=true`
- Only global Administrators can manage community role groups

**End-user agreement modal keeps reappearing after accepting**
- Check browser console for a failed PATCH to `/api/eperson/epersons/:id`
- Verify the user has EPerson edit permission in DSpace

<div class="page-nav">
  <a href="{{ '/admin/formbuilder/' | relative_url }}">← Form Builder</a>
  <a href="{{ '/ops/' | relative_url }}">Operations Guide →</a>
</div>
