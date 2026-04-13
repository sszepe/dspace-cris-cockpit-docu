---
layout: page
title: Database Export Views
permalink: /ops/database-export-views/
parent: Operations Guide
---

# Database Export Views

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169e1?style=flat-square&logo=postgresql&logoColor=white)
![Schema](https://img.shields.io/badge/schema-export-4169e1?style=flat-square)

The `export` schema provides a curated set of read-only views over the DSpace data model. Rather than exposing the raw normalised tables directly to reporting tools, backup scripts, or external consumers, these views pre-join, decode, and aggregate the most commonly needed data into stable, queryable surfaces.

<div class="callout callout-info">
<span class="callout-title">Production database context</span>
This setup targets <code>dspace</code>. The role names in this page use generic placeholders — replace <code>dspace_owner</code> and <code>dspace_export</code> with the actual roles in your environment. See <a href="#roles-and-permissions">Roles and Permissions</a> below.
</div>

---

## Roles and Permissions

The export schema uses two roles with distinct responsibilities:

| Role | Purpose | Default privilege on views |
|---|---|---|
| `dspace_owner` | Schema and object owner — the main DSpace application role | `ALL` |
| `dspace_export` | Read-only consumer — backup scripts, reporting tools, ETL pipelines | `SELECT` |

The SQL in this page uses these generic names. Substitute your site's actual role names when running:

```sql
-- Substitute your actual role names here:
-- dspace_owner  → e.g. dspace, repo_dspace_staging, mdw_dspace
-- dspace_export → e.g. dspace_backup, reporting_user, etl_reader
```

### Schema setup

```sql
-- Create the export schema
CREATE SCHEMA export
  AUTHORIZATION dspace_owner;

-- Grant usage to the read-only consumer
GRANT USAGE ON SCHEMA export TO dspace_export;

-- Convenience: grant SELECT on all future views automatically
ALTER DEFAULT PRIVILEGES IN SCHEMA export
  GRANT SELECT ON TABLES TO dspace_export;
```

---

## Dependency Views

Two helper views are referenced by multiple other views in this schema. Create these first.

### `export.new_users`

Tracks when EPerson accounts were first created by watching the `logging.history` table (see [Database Change Logging]({{ '/ops/database-logging/' | relative_url }})). This is used by `users_added_24h`.

```sql
CREATE OR REPLACE VIEW export.new_users AS
SELECT
  (new_val->>'uuid')::uuid  AS uuid,
  min(tstamp)               AS added_at
FROM logging.history
WHERE tabname   = 'eperson'
  AND operation = 'INSERT'
GROUP BY new_val->>'uuid';

ALTER TABLE export.new_users OWNER TO dspace_owner;
GRANT SELECT ON export.new_users TO dspace_export;
```

<div class="callout callout-warn">
<span class="callout-title">Requires DML logging</span>
<code>export.new_users</code> reads from <code>logging.history</code>. This table only exists when the row-level audit trigger is active (see <a href="{{ '/ops/database-logging/' | relative_url }}#part-2--dml-tracking-logging-schema">Database Change Logging — Part 2</a>). If <code>logging.history</code> is not available, replace this view with a direct creation-date column if one exists in your schema, or omit <code>users_added_24h</code>.
</div>

### `export.group_with_all_children`

Recursively expands `epersongroup` membership hierarchies. Used by the permissions views to show which groups (and their nested sub-groups) hold a policy on a resource.

```sql
CREATE OR REPLACE VIEW export.group_with_all_children AS
WITH RECURSIVE group_tree AS (
  -- Anchor: top-level groups
  SELECT
    g.uuid  AS parent_uuid,
    g.name  AS parent_name,
    g.uuid  AS child_uuid,
    g.name  AS child_name,
    0       AS depth
  FROM epersongroup g

  UNION ALL

  -- Recursive: add child groups
  SELECT
    gt.parent_uuid,
    gt.parent_name,
    child.uuid  AS child_uuid,
    child.name  AS child_name,
    gt.depth + 1
  FROM group_tree gt
  JOIN group2group g2g   ON gt.child_uuid  = g2g.parent_id
  JOIN epersongroup child ON child.uuid    = g2g.child_id
)
SELECT DISTINCT
  parent_uuid,
  parent_name,
  child_uuid,
  child_name,
  depth
FROM group_tree;

ALTER TABLE export.group_with_all_children OWNER TO dspace_owner;
GRANT SELECT ON export.group_with_all_children TO dspace_export;
```

---

## View Reference

### Overview

| View | Category | Description |
|---|---|---|
| [`users_last_active_24h`](#users_last_active_24h) | Users | EPersons active in the last 24 hours |
| [`users_added_24h`](#users_added_24h) | Users | EPersons whose accounts were created in the last 24 hours |
| [`all_users_with_netid`](#all_users_with_netid) | Users | All EPersons with an institutional SSO `netid` |
| [`metadata_field_view`](#metadata_field_view) | Metadata | Denormalised metadata field registry with full dotted field names |
| [`collections_metadata_view`](#collections_metadata_view) | Structure | Collections with title and item count statistics |
| [`communities_metadata_view`](#communities_metadata_view) | Structure | Community tree with recursive sub-community and collection counts |
| [`community_permissions_view`](#community_permissions_view) | Permissions | Community resource policies with human-readable action and type names |
| [`collection_permissions_view`](#collection_permissions_view) | Permissions | Collection resource policies with human-readable action names |

---

### `users_last_active_24h`

Returns all EPerson accounts that have logged in within the last 24 hours. Useful for activity reporting, session auditing, and health checks on the authentication pipeline.

```sql
CREATE OR REPLACE VIEW export.users_last_active_24h AS
SELECT
  eperson.email,
  eperson.can_log_in,
  eperson.last_active,
  eperson.netid,
  eperson.uuid
FROM eperson
WHERE eperson.last_active >= (now() - interval '24 hours');

ALTER TABLE export.users_last_active_24h OWNER TO dspace_owner;
GRANT SELECT ON export.users_last_active_24h TO dspace_export;
```

**Columns**

| Column | Type | Description |
|---|---|---|
| `email` | text | EPerson email address |
| `can_log_in` | boolean | Whether the account is allowed to log in |
| `last_active` | timestamptz | Timestamp of last successful authentication |
| `netid` | text | Institutional SSO identifier (NULL for local accounts) |
| `uuid` | uuid | EPerson UUID |

**Example queries**

```sql
-- Count of active users in the last 24h
SELECT count(*) FROM export.users_last_active_24h;

-- Active users without a netid (local password accounts only)
SELECT email, last_active
FROM export.users_last_active_24h
WHERE netid IS NULL
ORDER BY last_active DESC;

-- Vary the window — users active in the last hour
SELECT email, last_active
FROM eperson
WHERE last_active >= now() - interval '1 hour'
ORDER BY last_active DESC;
```

---

### `users_added_24h`

Returns EPerson accounts created within the last 24 hours, by joining with `export.new_users` which reads first-INSERT timestamps from `logging.history`.

```sql
CREATE OR REPLACE VIEW export.users_added_24h AS
SELECT
  e.email,
  e.can_log_in,
  e.last_active,
  e.netid,
  e.uuid
FROM eperson e
JOIN export.new_users nu ON e.uuid = nu.uuid
WHERE nu.added_at >= (now() - interval '24 hours');

ALTER TABLE export.users_added_24h OWNER TO dspace_owner;
GRANT SELECT ON export.users_added_24h TO dspace_export;
```

**Example queries**

```sql
-- Count of new registrations today
SELECT count(*) FROM export.users_added_24h;

-- New SSO users vs new local accounts
SELECT
  CASE WHEN netid IS NOT NULL THEN 'SSO' ELSE 'Local' END AS account_type,
  count(*) AS new_accounts
FROM export.users_added_24h
GROUP BY 1;
```

---

### `all_users_with_netid`

Returns all EPerson accounts that have a non-null `netid` — i.e. accounts linked to the institutional identity provider (SSO/LDAP/Shibboleth). Useful for synchronisation checks and user provisioning audits.

```sql
CREATE OR REPLACE VIEW export.all_users_with_netid AS
SELECT
  eperson.email,
  eperson.can_log_in,
  eperson.last_active,
  eperson.netid,
  eperson.uuid
FROM eperson
WHERE eperson.netid IS NOT NULL;

ALTER TABLE export.all_users_with_netid OWNER TO dspace_owner;
GRANT SELECT ON export.all_users_with_netid TO dspace_export;
```

**Example queries**

```sql
-- Total SSO-linked accounts
SELECT count(*) FROM export.all_users_with_netid;

-- Accounts that have never logged in (provisioned but dormant)
SELECT email, netid
FROM export.all_users_with_netid
WHERE last_active IS NULL
ORDER BY email;

-- Accounts that can't log in despite having a netid (disabled SSO users)
SELECT email, netid, last_active
FROM export.all_users_with_netid
WHERE can_log_in = false
ORDER BY last_active DESC NULLS LAST;
```

---

### `metadata_field_view`

Denormalises the metadata field registry by joining `metadatafieldregistry` with `metadataschemaregistry`, producing the familiar dotted field names (`dc.title`, `dc.contributor.author`, etc.) alongside scope notes and namespace URIs. This is the reference view for any query that needs to look up a field by name rather than numeric ID.

```sql
CREATE OR REPLACE VIEW export.metadata_field_view AS
SELECT
  mf.metadata_field_id,
  msr.short_id,
  mf.element,
  mf.qualifier,
  CASE
    WHEN mf.qualifier IS NOT NULL
      THEN msr.short_id || '.' || mf.element || '.' || mf.qualifier
    ELSE
      msr.short_id || '.' || mf.element
  END AS field_name,
  mf.scope_note,
  msr.namespace
FROM metadatafieldregistry  mf
JOIN metadataschemaregistry msr ON mf.metadata_schema_id = msr.metadata_schema_id;

ALTER TABLE export.metadata_field_view OWNER TO dspace_owner;
```

**Columns**

| Column | Description |
|---|---|
| `metadata_field_id` | Numeric ID used in `metadatavalue.metadata_field_id` |
| `short_id` | Schema prefix, e.g. `dc`, `dcterms`, `crisrp` |
| `element` | Field element, e.g. `title`, `contributor` |
| `qualifier` | Field qualifier, e.g. `author`, `editor` — NULL for unqualified fields |
| `field_name` | Full dotted name, e.g. `dc.contributor.author` |
| `scope_note` | Human-readable description from the registry |
| `namespace` | Schema namespace URI |

**Example queries**

```sql
-- Look up a field ID by name
SELECT metadata_field_id
FROM export.metadata_field_view
WHERE field_name = 'dc.title';

-- All fields in the crisrp schema
SELECT field_name, scope_note
FROM export.metadata_field_view
WHERE short_id = 'crisrp'
ORDER BY field_name;

-- Find fields by partial name
SELECT field_name, metadata_field_id
FROM export.metadata_field_view
WHERE field_name ILIKE '%contributor%'
ORDER BY field_name;
```

<div class="callout callout-info">
<span class="callout-title">Use this view to avoid hardcoded field IDs</span>
Queries against <code>metadatavalue</code> often use hardcoded numeric IDs like <code>WHERE metadata_field_id = 82</code>. This is brittle across DSpace versions and migrations. Join through <code>export.metadata_field_view</code> instead:
<pre><code>JOIN export.metadata_field_view mfv
  ON mv.metadata_field_id = mfv.metadata_field_id
  AND mfv.field_name = 'dc.title'</code></pre>
</div>

---

### `collections_metadata_view`

Aggregates each collection's title and item statistics — total items, items in archive, withdrawn items, discoverable items, and items owned by the collection (as opposed to mapped items).

```sql
CREATE OR REPLACE VIEW export.collections_metadata_view AS
WITH collection_metadata AS (
  SELECT
    col.uuid          AS collection_uuid,
    col.collection_id,
    mv.text_value     AS title,
    mv.security_level AS title_sl
  FROM collection col
  LEFT JOIN community2collection c2c ON col.uuid = c2c.collection_id
  LEFT JOIN metadatavalue mv
    ON mv.dspace_object_id = col.uuid
   AND mv.metadata_field_id = (
     SELECT metadata_field_id FROM export.metadata_field_view
     WHERE field_name = 'dc.title' LIMIT 1
   )
),
item_counts AS (
  SELECT
    c2i.collection_id,
    count(i.uuid)                                           AS nr_items,
    sum(CASE WHEN i.in_archive   THEN 1 ELSE 0 END)        AS nr_in_archive,
    sum(CASE WHEN i.withdrawn    THEN 1 ELSE 0 END)        AS nr_withdrawn,
    sum(CASE WHEN i.discoverable THEN 1 ELSE 0 END)        AS nr_discoverable,
    sum(CASE WHEN i.owning_collection = c2i.collection_id
             THEN 1 ELSE 0 END)                            AS nr_where
  FROM collection2item c2i
  LEFT JOIN item i ON c2i.item_id = i.uuid
  GROUP BY c2i.collection_id
)
SELECT
  col.collection_uuid,
  col.collection_id,
  col.title,
  col.title_sl,
  ic.nr_items,
  ic.nr_in_archive,
  ic.nr_withdrawn,
  ic.nr_discoverable,
  ic.nr_where
FROM collection_metadata col
LEFT JOIN item_counts ic ON col.collection_uuid = ic.collection_id;

ALTER TABLE export.collections_metadata_view OWNER TO dspace_owner;
```

**Columns**

| Column | Description |
|---|---|
| `collection_uuid` | Collection UUID |
| `collection_id` | Internal integer ID |
| `title` | Collection title (`dc.title`) |
| `title_sl` | Security level of the title field |
| `nr_items` | Total items in the collection (owned + mapped) |
| `nr_in_archive` | Items with `in_archive = true` |
| `nr_withdrawn` | Items with `withdrawn = true` |
| `nr_discoverable` | Items with `discoverable = true` |
| `nr_where` | Items whose `owning_collection` is this collection |

**Example queries**

```sql
-- Collections with the most items
SELECT title, nr_items, nr_in_archive, nr_withdrawn
FROM export.collections_metadata_view
ORDER BY nr_items DESC NULLS LAST
LIMIT 20;

-- Collections with withdrawn items
SELECT title, nr_items, nr_withdrawn
FROM export.collections_metadata_view
WHERE nr_withdrawn > 0
ORDER BY nr_withdrawn DESC;

-- Collections where mapped items exceed owned items
SELECT title, nr_items, nr_where,
       (nr_items - nr_where) AS mapped_items
FROM export.collections_metadata_view
WHERE (nr_items - nr_where) > 0
ORDER BY mapped_items DESC;

-- Empty collections (no items at all)
SELECT title, collection_uuid
FROM export.collections_metadata_view
WHERE nr_items IS NULL OR nr_items = 0;
```

---

### `communities_metadata_view`

Builds a recursive community tree using a CTE, attaching collection counts, sub-community depth, metadata (title, description, handle), and security levels to each community node. `tree_collection_count` aggregates the total number of collections reachable from each root community including all descendant sub-communities.

```sql
CREATE OR REPLACE VIEW export.communities_metadata_view AS
WITH RECURSIVE community_tree AS (
  -- Anchor: root communities (no parent)
  SELECT
    c.uuid          AS community_uuid,
    c.uuid          AS root_uuid,
    c.community_id,
    c.admin,
    NULL::text      AS sub_communities,
    0               AS sub_community_count
  FROM community c
  LEFT JOIN community2community c2c ON c.uuid = c2c.child_comm_id
  WHERE c2c.parent_comm_id IS NULL

  UNION ALL

  -- Recursive: child communities
  SELECT
    child.uuid                                             AS community_uuid,
    ct.root_uuid,
    child.community_id,
    child.admin,
    coalesce(ct.sub_communities || '|' || child.uuid::text,
             child.uuid::text)                            AS sub_communities,
    ct.sub_community_count + 1
  FROM community child
  JOIN community2community c2c ON child.uuid = c2c.child_comm_id
  JOIN community_tree ct       ON ct.community_uuid = c2c.parent_comm_id
),
community_collections AS (
  SELECT
    comm.uuid                                            AS community_uuid,
    string_agg(col.uuid::text, '|')                     AS collections,
    count(col.uuid)                                      AS collection_count
  FROM community comm
  LEFT JOIN community2collection c2col ON comm.uuid = c2col.community_id
  LEFT JOIN collection col              ON c2col.collection_id = col.uuid
  GROUP BY comm.uuid
),
tree_collection_count AS (
  SELECT
    ct.root_uuid,
    sum(cc.collection_count) AS tree_collection_count
  FROM community_tree ct
  LEFT JOIN community_collections cc ON ct.community_uuid = cc.community_uuid
  GROUP BY ct.root_uuid
),
metadata_values AS (
  SELECT
    mv.dspace_object_id,
    max(CASE WHEN mv.metadata_field_id = 82 THEN mv.text_value END)     AS title,
    max(CASE WHEN mv.metadata_field_id = 82 THEN mv.security_level END) AS title_sl,
    max(CASE WHEN mv.metadata_field_id = 39 THEN mv.text_value END)     AS description,
    max(CASE WHEN mv.metadata_field_id = 39 THEN mv.security_level END) AS description_sl,
    max(CASE WHEN mv.metadata_field_id = 34 THEN mv.text_value END)     AS handle,
    max(CASE WHEN mv.metadata_field_id = 34 THEN mv.security_level END) AS handle_sl
  FROM metadatavalue mv
  GROUP BY mv.dspace_object_id
)
SELECT
  c.uuid          AS community_uuid,
  c.community_id,
  c.admin,
  ct.sub_communities,
  ct.sub_community_count,
  cc.collections,
  cc.collection_count,
  tcc.tree_collection_count,
  mv.title,
  mv.title_sl,
  mv.description,
  mv.description_sl,
  mv.handle,
  mv.handle_sl
FROM community c
LEFT JOIN community_tree       ct  ON c.uuid = ct.community_uuid
LEFT JOIN community_collections cc  ON c.uuid = cc.community_uuid
LEFT JOIN tree_collection_count tcc ON c.uuid = tcc.root_uuid
LEFT JOIN metadata_values       mv  ON c.uuid = mv.dspace_object_id;

ALTER TABLE export.communities_metadata_view OWNER TO dspace_owner;
```

**Columns**

| Column | Description |
|---|---|
| `community_uuid` | Community UUID |
| `community_id` | Internal integer ID |
| `admin` | UUID of the community's admin group |
| `sub_communities` | Pipe-separated list of all descendant community UUIDs |
| `sub_community_count` | Depth of nesting (0 = root community) |
| `collections` | Pipe-separated list of directly attached collection UUIDs |
| `collection_count` | Number of collections directly in this community |
| `tree_collection_count` | Total collections reachable from this community's root (including sub-communities) |
| `title` | Community title (`dc.title`) |
| `title_sl` | Security level of the title field |
| `description` | Community description (`dc.description`) |
| `description_sl` | Security level of the description field |
| `handle` | DSpace handle (`dc.identifier.uri`) |
| `handle_sl` | Security level of the handle field |

**Example queries**

```sql
-- Top-level communities with their total collection counts
SELECT title, collection_count, tree_collection_count
FROM export.communities_metadata_view
WHERE sub_community_count = 0
ORDER BY tree_collection_count DESC;

-- Deeply nested communities
SELECT title, sub_community_count, handle
FROM export.communities_metadata_view
WHERE sub_community_count > 1
ORDER BY sub_community_count DESC;

-- Communities with no collections directly attached
SELECT title, community_uuid
FROM export.communities_metadata_view
WHERE collection_count = 0 OR collection_count IS NULL;
```

<div class="callout callout-warn">
<span class="callout-title">Hardcoded metadata field IDs</span>
The <code>metadata_values</code> CTE uses numeric IDs 82 (dc.title), 39 (dc.description), and 34 (dc.identifier.uri). These may differ across DSpace installations or after metadata schema migrations. Verify with:
<pre><code>SELECT metadata_field_id, field_name
FROM export.metadata_field_view
WHERE field_name IN ('dc.title','dc.description','dc.identifier.uri');</code></pre>
</div>

---

### `community_permissions_view`

Joins community resource policies with human-readable `resource_type_name` and `action_name` labels, EPerson and group details, and the full recursive group membership expansion from `group_with_all_children`. This gives a complete picture of who has what access to each community.

```sql
CREATE OR REPLACE VIEW export.community_permissions_view AS
SELECT
  comm.uuid                                       AS community_uuid,
  comm.admin,
  rp.policy_id,
  rp.resource_type_id,
  CASE rp.resource_type_id
    WHEN 0 THEN 'BITSTREAM'   WHEN 1 THEN 'BUNDLE'
    WHEN 2 THEN 'ITEM'        WHEN 3 THEN 'COLLECTION'
    WHEN 4 THEN 'COMMUNITY'   WHEN 5 THEN 'SITE'
    WHEN 6 THEN 'GROUP'       WHEN 7 THEN 'EPERSON'
    ELSE 'UNKNOWN'
  END                                             AS resource_type_name,
  rp.action_id,
  CASE rp.action_id
    WHEN 0  THEN 'READ'                  WHEN 1  THEN 'WRITE'
    WHEN 2  THEN 'DELETE'                WHEN 3  THEN 'ADD'
    WHEN 4  THEN 'REMOVE'                WHEN 5  THEN 'WORKFLOW_STEP_1'
    WHEN 6  THEN 'WORKFLOW_STEP_2'       WHEN 7  THEN 'WORKFLOW_STEP_3'
    WHEN 8  THEN 'WORKFLOW_ABORT'        WHEN 9  THEN 'DEFAULT_BITSTREAM_READ'
    WHEN 10 THEN 'DEFAULT_ITEM_READ'     WHEN 11 THEN 'ADMIN'
    WHEN 12 THEN 'WITHDRAWN_READ'
    ELSE 'UNKNOWN'
  END                                             AS action_name,
  rp.start_date,
  rp.end_date,
  rp.rpname,
  rp.rptype,
  rp.rpdescription,
  rp.eperson_id,
  p.email,
  p.netid,
  rp.epersongroup_id,
  g.name                                          AS group_name,
  mv.text_value                                   AS title,
  gh.parent_name,
  gh.child_uuid,
  gh.child_name
FROM community comm
LEFT JOIN resourcepolicy rp
       ON comm.uuid = rp.dspace_object
LEFT JOIN metadatavalue mv
       ON mv.dspace_object_id = comm.uuid
      AND mv.metadata_field_id = 82
LEFT JOIN export.group_with_all_children gh
       ON rp.epersongroup_id = gh.parent_uuid
LEFT JOIN eperson p
       ON p.uuid = rp.eperson_id
LEFT JOIN epersongroup g
       ON g.uuid = rp.epersongroup_id
ORDER BY comm.uuid, rp.policy_id;

ALTER TABLE export.community_permissions_view OWNER TO dspace_owner;
GRANT SELECT ON export.community_permissions_view TO dspace_export;
```

**Key columns**

| Column | Description |
|---|---|
| `community_uuid` | Community UUID |
| `action_name` | Human-readable action (READ, WRITE, ADMIN, etc.) |
| `resource_type_name` | Human-readable resource type (COMMUNITY, COLLECTION, etc.) |
| `group_name` | Name of the group holding this policy (NULL if policy is on an individual EPerson) |
| `email` / `netid` | EPerson details when policy is on an individual (NULL if group-based) |
| `parent_name` / `child_uuid` / `child_name` | Rows from `group_with_all_children` — one row per group member in the policy's group hierarchy |
| `start_date` / `end_date` | Policy date range (NULL = no expiry) |

**Example queries**

```sql
-- All ADMIN policies on communities
SELECT title, community_uuid, group_name, email, action_name
FROM export.community_permissions_view
WHERE action_name = 'ADMIN'
ORDER BY title;

-- Policies granted to individual EPersons (not groups)
SELECT title, email, netid, action_name
FROM export.community_permissions_view
WHERE eperson_id IS NOT NULL
ORDER BY title, action_name;

-- Policies with expiry dates (time-limited access)
SELECT title, group_name, action_name, start_date, end_date
FROM export.community_permissions_view
WHERE end_date IS NOT NULL
ORDER BY end_date;

-- Who has access to a specific community (by UUID)
SELECT DISTINCT group_name, child_name, action_name
FROM export.community_permissions_view
WHERE community_uuid = '<community-uuid>'
ORDER BY action_name, group_name;
```

---

### `collection_permissions_view`

Identical pattern to `community_permissions_view` but scoped to collections. Includes the same human-readable action decoding and recursive group expansion.

```sql
CREATE OR REPLACE VIEW export.collection_permissions_view AS
SELECT
  col.uuid                                        AS collection_uuid,
  col.admin,
  rp.policy_id,
  rp.resource_type_id,
  rp.action_id,
  CASE rp.action_id
    WHEN 0  THEN 'READ'                  WHEN 1  THEN 'WRITE'
    WHEN 2  THEN 'DELETE'                WHEN 3  THEN 'ADD'
    WHEN 4  THEN 'REMOVE'                WHEN 5  THEN 'WORKFLOW_STEP_1'
    WHEN 6  THEN 'WORKFLOW_STEP_2'       WHEN 7  THEN 'WORKFLOW_STEP_3'
    WHEN 8  THEN 'WORKFLOW_ABORT'        WHEN 9  THEN 'DEFAULT_BITSTREAM_READ'
    WHEN 10 THEN 'DEFAULT_ITEM_READ'     WHEN 11 THEN 'ADMIN'
    WHEN 12 THEN 'WITHDRAWN_READ'
    ELSE 'UNKNOWN'
  END                                             AS action_name,
  rp.start_date,
  rp.end_date,
  rp.rpname,
  rp.rptype,
  rp.rpdescription,
  rp.eperson_id,
  rp.epersongroup_id,
  mv.text_value                                   AS title,
  gh.parent_name,
  gh.child_uuid,
  gh.child_name
FROM collection col
LEFT JOIN resourcepolicy rp
       ON col.uuid = rp.dspace_object
LEFT JOIN metadatavalue mv
       ON mv.dspace_object_id = col.uuid
      AND mv.metadata_field_id = 82
LEFT JOIN export.group_with_all_children gh
       ON rp.epersongroup_id = gh.parent_uuid
ORDER BY col.uuid, rp.policy_id;

ALTER TABLE export.collection_permissions_view OWNER TO dspace_owner;
GRANT SELECT ON export.collection_permissions_view TO dspace_export;
```

**Example queries**

```sql
-- Collections where workflow steps are configured
SELECT DISTINCT title, collection_uuid, action_name
FROM export.collection_permissions_view
WHERE action_name LIKE 'WORKFLOW%'
ORDER BY title, action_name;

-- Submitter groups per collection
SELECT title, collection_uuid, parent_name AS submitter_group
FROM export.collection_permissions_view
WHERE action_name = 'ADD'
  AND epersongroup_id IS NOT NULL
ORDER BY title;

-- Collections with no resource policies (open/unprotected)
SELECT title, c.uuid
FROM collection c
LEFT JOIN export.collection_permissions_view cpv ON c.uuid = cpv.collection_uuid
WHERE cpv.collection_uuid IS NULL;

-- All members of a collection's workflow groups (expanded)
SELECT title, action_name, parent_name, child_name
FROM export.collection_permissions_view
WHERE collection_uuid = '<collection-uuid>'
  AND action_name LIKE 'WORKFLOW%'
ORDER BY action_name, child_name;
```

---

## Suggested Additional Views

These views are not part of the current setup but follow naturally from the existing ones:

```sql
-- Items with their owning collection and community path
CREATE OR REPLACE VIEW export.items_with_location AS
SELECT
  i.uuid         AS item_uuid,
  i.in_archive,
  i.withdrawn,
  i.discoverable,
  oc.uuid        AS owning_collection_uuid,
  col_mv.text_value AS collection_title,
  comm.uuid      AS community_uuid,
  comm_mv.text_value AS community_title
FROM item i
LEFT JOIN collection oc ON i.owning_collection = oc.uuid
LEFT JOIN community2collection c2col ON oc.uuid = c2col.collection_id
LEFT JOIN community comm ON c2col.community_id = comm.uuid
LEFT JOIN metadatavalue col_mv
       ON col_mv.dspace_object_id = oc.uuid   AND col_mv.metadata_field_id = 82
LEFT JOIN metadatavalue comm_mv
       ON comm_mv.dspace_object_id = comm.uuid AND comm_mv.metadata_field_id = 82;

-- EPerson group membership (flat, non-recursive)
CREATE OR REPLACE VIEW export.group_members AS
SELECT
  g.uuid   AS group_uuid,
  g.name   AS group_name,
  e.uuid   AS eperson_uuid,
  e.email,
  e.netid,
  e.can_log_in,
  e.last_active
FROM epersongroup g
JOIN epersongroup2eperson g2e ON g.uuid = g2e.eperson_group_id
JOIN eperson e                 ON e.uuid  = g2e.eperson_id;
```

---

## Maintenance

### Refresh after DSpace upgrades

Views do not need to be rebuilt unless the underlying table structure changes. After a DSpace migration, verify the hardcoded field IDs are still correct:

```sql
SELECT metadata_field_id, field_name
FROM export.metadata_field_view
WHERE field_name IN (
  'dc.title',
  'dc.description',
  'dc.identifier.uri'
)
ORDER BY field_name;
```

If any IDs have changed, recreate the affected views with the updated values.

### Grant SELECT on new views to the export role

When adding new views, always grant SELECT explicitly (or rely on `ALTER DEFAULT PRIVILEGES` set during schema setup):

```sql
GRANT SELECT ON export.<new_view_name> TO dspace_export;
```

### List all views in the export schema

```sql
SELECT viewname, viewowner
FROM pg_views
WHERE schemaname = 'export'
ORDER BY viewname;
```

---

## Reference

| Resource | Link |
|---|---|
| PostgreSQL views | [postgresql.org/docs/current/sql-createview](https://www.postgresql.org/docs/current/sql-createview.html) |
| Recursive CTEs | [postgresql.org/docs/current/queries-with](https://www.postgresql.org/docs/current/queries-with.html) |
| `ALTER DEFAULT PRIVILEGES` | [postgresql.org/docs/current/sql-alterdefaultprivileges](https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html) |
| DSpace resource policies | [wiki.lyrasis.org/display/DSPACE/Authorization+System](https://wiki.lyrasis.org/display/DSPACE/Authorization+System) |
| DSpace EPerson model | [wiki.lyrasis.org/display/DSDOC7x/People+and+Groups](https://wiki.lyrasis.org/display/DSDOC7x/People+and+Groups) |

<div class="page-nav">
  <a href="{{ '/ops/database-logging/' | relative_url }}">← Database Change Logging</a>
  <a href="{{ '/ops/database-performance/' | relative_url }}">Database Performance →</a>
</div>
