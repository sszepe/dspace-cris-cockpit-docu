---
layout: page
title: Solr Search Core — Schema & Config
permalink: /ops/solr-search-core/
parent: Operations Guide
---

# Solr Search Core — Schema & Configuration

![Solr](https://img.shields.io/badge/Solr-8.11.4-d9411e?style=flat-square&logo=apachesolr&logoColor=white)
![Lucene](https://img.shields.io/badge/Lucene-8.8.1-007396?style=flat-square)
![Access](https://img.shields.io/badge/access-internal%20only-orange?style=flat-square)

This page documents the `search` core of the DSpace CRIS 2024 Solr 8 instance — the schema (`schema.xml`), the query configuration (`solrconfig.xml`), and the text analysis configuration files (`stopwords.txt`, `synonyms.txt`, `protwords.txt`).

The `search` core is the primary index powering DSpace Discovery: full-text search, faceted browse, autocomplete, entity type filtering, and the HAL REST Discovery API. Understanding the schema is essential for writing direct Solr queries, diagnosing unexpected search behaviour, and tuning performance.

**Configuration files location inside the container:**

```
/opt/solr/server/solr/configsets/search/conf/
├── schema.xml
├── solrconfig.xml
├── stopwords.txt
├── synonyms.txt
└── protwords.txt
```

---

## solrconfig.xml

### Core settings

| Setting | Value | Notes |
|---|---|---|
| `luceneMatchVersion` | `8.8.1` | Lucene compatibility level for index format and analysis |
| `schemaFactory` | `ClassicIndexSchemaFactory` | Uses `schema.xml` directly; programmatic schema changes via API are **disabled** |
| `directoryFactory` | `NRTCachingDirectoryFactory` | Near-real-time caching — new segments cached in memory before flushing to disk |
| `ramBufferSizeMB` | `100` | RAM allocated to index write buffer before flush |
| `maxBufferedDocs` | `1000` | Flush after 1000 buffered documents regardless of RAM |

The `ClassicIndexSchemaFactory` is important: it means the schema is managed exclusively via `schema.xml` on disk. Changes to the schema require editing the file and reloading the core — you cannot add fields via the Schema API.

### Update handler & commit strategy

| Setting | Value | Meaning |
|---|---|---|
| `autoCommit.maxDocs` | `10000` | Hard commit after every 10,000 indexed documents |
| `autoCommit.maxTime` | `10000ms` (10s) | Hard commit every 10 seconds regardless of doc count |
| `autoCommit.openSearcher` | `true` | Each hard commit opens a new searcher — results become visible immediately |
| `autoSoftCommit.maxTime` | `-1` (disabled) | Soft commits are not used; only hard commits |
| `updateLog` | enabled | Required for atomic updates and crash recovery |

<div class="callout callout-info">
<span class="callout-title">Hard commit every 10 seconds</span>
DSpace indexes changes with a 10-second auto-commit window. This means newly submitted or edited items may take up to 10 seconds to appear in search results. During bulk reindexing (<code>index-discovery -f</code>), commits happen every 10,000 documents or 10 seconds — whichever comes first.
</div>

### Query settings

| Setting | Value | Notes |
|---|---|---|
| `maxBooleanClauses` | `1024` | Maximum number of clauses in a boolean query (e.g. `IN` lists) |
| `enableLazyFieldLoading` | `false` | All stored fields are loaded eagerly on document retrieval (required for atomic updates — see [SOLR-13034](https://issues.apache.org/jira/browse/SOLR-13034)) |
| `queryResultWindowSize` | `20` | Pre-fetches 20 results around the requested window for cache efficiency |
| `queryResultMaxDocsCached` | `200` | Maximum documents cached per query result |
| `useColdSearcher` | `false` | Block queries until the new searcher is warmed — no stale results served |
| `maxWarmingSearchers` | `2` | At most 2 searchers can warm simultaneously |
| `slowQueryThresholdMillis` | **`1000`** | Queries exceeding 1s are logged as slow queries |

### Caches

All three caches use `CaffeineCache` (Solr 8's default high-performance cache):

| Cache | Size | Purpose |
|---|---|---|
| `filterCache` | 512 entries | Caches `fq` filter queries as DocSets (unordered bitsets). Very effective for repeated entity type and `archived:true` filters |
| `queryResultCache` | 512 entries | Caches ordered result lists (DocList) for repeated `q` + sort + pagination combinations |
| `documentCache` | 512 entries | Caches stored fields of fetched documents — reduces disk reads for repeated document lookups |

All caches have `autowarmCount=0` — when a new searcher opens after a commit, caches start cold. This is acceptable for DSpace since queries are varied and pre-warming would add latency to commits.

<div class="callout callout-info">
<span class="callout-title">filterCache is the most important cache to watch</span>
DSpace Discovery queries almost always include <code>fq=archived:true</code> and <code>fq=search.resourcetype:2</code>. These are ideal filter cache candidates. A healthy filter cache hit ratio above 80% means most queries avoid full DocSet computation. Monitor at:<br>
<code>http://localhost:8983/solr/search/admin/mbeans?cat=CACHE&stats=true</code>
</div>

### Request handlers

| Handler | Class | Notes |
|---|---|---|
| `/select` | `solr.SearchHandler` | Main query endpoint. Default field: `search_text`. Default operator: `AND`. Default rows: `10`. Spellcheck runs as last component. |
| `/update` | `solr.UpdateRequestHandler` | Document ingestion (used by DSpace indexer) |
| `/update/json` | `solr.UpdateRequestHandler` | JSON update stream |
| `/spell` | `solr.SearchHandler` | Dedicated spellcheck endpoint (lazy-loaded) |

The `/select` handler defaults:

```xml
<str name="df">search_text</str>   <!-- default search field — the catch-all copyField -->
<str name="q.op">AND</str>         <!-- all terms must match by default -->
```

This means a bare `q=university music` query requires **both** "university" and "music" to match, which aligns with DSpace's expected search behaviour.

### Spellchecker

Uses `IndexBasedSpellChecker` — the spellcheck index is built from the `a_spell` field (which is a copyField of `fulltext`). The spellcheck index rebuilds automatically on Solr optimize. Spell suggestions are appended to `/select` responses when requested via `&spellcheck=true`.

---

## schema.xml

### Schema identity

```xml
<schema name="discovery" version="1.5">
<uniqueKey>search.uniqueid</uniqueKey>
```

The unique key is `search.uniqueid`, formatted as `<resourceid>-<resourcetype>` for standard DSpace objects (e.g. `550e8400-e29b-41d4-a716-446655440000-2` for an Item).

---

## Field Types

The schema defines 14 named field types. Understanding which type a field uses determines how it is tokenised, whether exact matching works, and whether it can be sorted.

### Primitive / numeric types

| Field type | Solr class | docValues | Notes |
|---|---|---|---|
| `string` | `StrField` | no | Exact-match, case-sensitive. Stored as-is. |
| `boolean` | `BoolField` | no | `true` / `false` |
| `int`, `long`, `float`, `double` | `*PointField` | yes | Range-queryable numeric types |
| `date` | `DatePointField` | yes | ISO 8601 timestamps; supports date math |
| `pint`, `plong`, `pfloat`, `pdouble`, `pdate` | `*PointField` | yes | `sortMissingLast` variants |
| `sint`, `slong`, `sfloat`, `sdouble` | `*PointField` | yes | Sort-safe numeric types |
| `random` | `RandomSortField` | — | For random ordering (`sort=random_* asc`) |
| `ignored` | `StrField` | — | Not indexed, not stored — silently discards values |

### Text analysis types

These are the types that matter most for search behaviour:

#### `text` — primary full-text type

Used by the catch-all `search_text` field, `fulltext`, and most metadata fields via the `*` dynamic field fallback.

**Index analyser pipeline:**

```
WhitespaceTokenizer
  → StopFilter          (stopwords.txt, case-insensitive)
  → WordDelimiterFilter (split on case change, generate word parts + number parts,
                         catenate words + numbers, do NOT catenate all)
  → ICUFoldingFilter    (Unicode normalisation — accents, diacritics, case folding)
  → KeywordRepeatFilter (preserves original token alongside stemmed version)
  → SnowballPorter      (English stemming, protected by protwords.txt)
  → RemoveDuplicates
```

**Query analyser pipeline:**

```
WhitespaceTokenizer
  → SynonymFilter       (synonyms.txt, case-insensitive, expand=true)
  → StopFilter          (stopwords.txt)
  → WordDelimiterFilter (split on case change, generate parts — NO catenation at query time)
  → ICUFoldingFilter
  → SnowballPorter      (English stemming, protected by protwords.txt)
  → RemoveDuplicates
```

Key differences between index and query pipelines:

| Step | Index | Query | Why it differs |
|---|---|---|---|
| Synonyms | ❌ not applied | ✅ applied | Synonyms expand at query time so indexed terms don't need all variants |
| Catenate words/numbers | ✅ yes | ❌ no | Index stores both `wifi` and `Wi-Fi`; query only needs one form |
| `KeywordRepeat` | ✅ yes | ❌ no | Index keeps both stemmed and unstemmed token; query doesn't need it |

#### `textgen` — general unstemmed text

Similar to `text` but without stemming. Used for fields where the language is unknown or stemming would be harmful. Both index and query pipelines use WhitespaceTokenizer → StopFilter → WordDelimiterFilter → LowerCaseFilter.

#### `textTight` — tight matching

Uses `synonyms.txt` but with `expand=false` (maps to single canonical form), no WordDelimiter parts generation, and LowerCaseFilter. Good for fields where partial word matches would produce false positives.

#### `lowerCaseSort` — sort and exact faceting

Used by `_sort` dynamic fields and location fields. Single pipeline (no index/query split):

```
KeywordTokenizer  (no splitting — entire value is one token)
  → LowerCaseFilter
  → ICUFoldingFilter  (Unicode normalisation)
  → TrimFilter
```

This is why `dc.title_sort` supports exact matching while `dc.title` does not — `KeywordTokenizer` preserves the entire string as a single token.

#### `keywordFilter` — facets, filters, autocomplete, authority

Used by all `_filter`, `_keyword`, `_ac`, `_acid`, `_authority`, `_prefix` dynamic fields:

```
KeywordTokenizer  (no splitting)
  → TrimFilter
```

Case is **preserved** — no lowercasing. This means `facet.field=entityType` returns `Person`, `Publication` etc. with their original capitalisation. Exact matching works: `entityType:Person` is case-sensitive.

<div class="callout callout-warn">
<span class="callout-title">keywordFilter is case-sensitive</span>
Fields using <code>keywordFilter</code> (including all <code>_keyword</code>, <code>_filter</code>, <code>_ac</code> fields) preserve case exactly. <code>entityType:person</code> will NOT match documents where <code>entityType</code> is stored as <code>Person</code>. Always use the correct capitalisation.
</div>

#### `dspaceMetadataProjection` — stored projection

Single `KeywordTokenizer` only — no filtering. Used for `_stored` fields that store the raw serialised metadata value (with authority, preferred label, variants, language) for projection in API responses. Not used for search.

#### `dspaceAutoComplete` — autocomplete

```
KeywordTokenizer → LowerCaseFilter → StopFilter → TrimFilter
```

Not currently wired to any named field directly — referenced by the `dspaceAutoComplete` type for potential use.

#### `itemLookup` — item authority lookup

Used by `itemauthoritylookup`:

```
KeywordTokenizer
  → WordDelimiterFilter (generate word parts, catenate words/numbers, split on case change)
  → LowerCaseFilter
  → ICUFoldingFilter
  → TrimFilter
```

Enables partial-word matching on titles for authority linking while normalising case and diacritics.

#### `textSpell` — spellcheck

Used only by the `a_spell` field (populated via copyField from `fulltext`). Index pipeline: StandardTokenizer → LowerCase → Synonyms → StopFilter → RemoveDuplicates. Query pipeline: StandardTokenizer → LowerCase → StopFilter → RemoveDuplicates.

#### `text_ws` — whitespace-only

Single `WhitespaceTokenizer` — splits on whitespace only, no other normalisation. Used for fields where only word-boundary splitting is needed.

---

## Static Fields

These fields are explicitly declared in `<fields>` and are always present in every indexed document.

### System / identity fields

| Field | Type | Multi | Required | Description |
|---|---|---|---|---|
| `_version_` | `long` | no | — | Solr internal optimistic concurrency version |
| `search.uniqueid` | `string` | no | ✅ | Unique document ID — `<uuid>-<resourcetype>` |
| `search.resourceid` | `string` | no | ✅ | DSpace object UUID |
| `search.resourcetype` | `string` | no | ✅ | DSpace type integer as string: `2`=Item, `3`=Collection, `4`=Community |
| `search.entitytype` | `string` | no | no | Entity type string (populated via copyField from `dspace.entity.type`) |
| `handle` | `string` | no | no | DSpace handle (e.g. `123456789/42`) |
| `customurl` | `string` | yes | no | Custom URL paths for CRIS entity pages |

### Item status fields

| Field | Type | Multi | Description |
|---|---|---|---|
| `archived` | — | — | Whether the item has been deposited/archived (set via DSpace indexer; matches `in_archive = true` in PostgreSQL) |
| `withdrawn` | `string` | no | `true` if the item has been withdrawn |
| `discoverable` | `string` | no | `true` if the item is publicly discoverable |
| `latestVersion` | `boolean` | no | `true` if this is the latest version (default `true`) |
| `database_status` | `string` | no | DSpace internal item database status |

### Content fields

| Field | Type | Multi | Description |
|---|---|---|---|
| `search_text` | `text` | yes | Catch-all field — populated via `copyField source="*"`. Default search field (`df`). Never stored, only indexed. |
| `fulltext` | `text` | yes | Full text extracted from bitstreams (PDFs etc.) |
| `fulltext_hl` | — | — | Copy of `fulltext` for hit highlighting (populated via copyField) |
| `a_spell` | `textSpell` | — | Copy of `fulltext` for the spellchecker (not stored) |
| `itemauthoritylookup` | `itemLookup` | yes | Copy of `title` for authority-aware item lookup |
| `itemauthoritylookupexactmatch` | `text` | yes | Copy of `title` for exact-match authority lookup |

### Access control fields

| Field | Type | Multi | Description |
|---|---|---|---|
| `read` | `string` | yes | Group/EPerson UUIDs with READ permission — used for access-controlled Discovery queries |
| `submit` | `string` | yes | Group UUIDs with SUBMIT permission on collections |
| `edit` | `string` | yes | EPerson UUIDs with EDIT permission on items |
| `submitter` | `string` | yes | UUID of the EPerson who submitted the item |
| `taskfor` | `string` | yes | Workflow task assignment |

### Date tracking

| Field | Type | Description |
|---|---|---|
| `SolrIndexer.lastIndexed` | `date` | Timestamp when this document was last indexed. Default: `NOW`. Useful for incremental monitoring. |
| `lastModified` | `date` | Last modification timestamp from DSpace. Default: `NOW`. |
| `lastModified_dt` | `date` | docValues copy of `lastModified` (populated via copyField). Required for sorting. |

### Location / hierarchy fields

| Field | Type | Multi | Description |
|---|---|---|---|
| `location` | `lowerCaseSort` | yes | Combined community + collection location identifiers |
| `location.comm` | `lowerCaseSort` | yes | Community UUID(s) the item belongs to |
| `location.coll` | `lowerCaseSort` | yes | Collection UUID(s) the item belongs to |
| `location.parent` | `lowerCaseSort` | no | Direct parent community/collection UUID |

---

## Dynamic Fields

Dynamic fields match any field name that ends with the declared suffix (or matches the pattern). They are the key to understanding how DSpace CRIS metadata fields are indexed — `dc.title`, `dc.contributor.author`, `entityType`, etc. are not declared as explicit fields; they match dynamic patterns.

### Metadata field patterns

| Pattern | Type | Multi | Purpose | Example field |
|---|---|---|---|---|
| `*_sort` | `lowerCaseSort` | **no** | Sorting and exact matching. KeywordTokenizer → lowercase → ICU fold. | `dc.title_sort` |
| `*_keyword` | `keywordFilter` | yes | Exact-match filtering, keyword facets. Case-preserved. | `birthDate_keyword`, `entityType_keyword` |
| `*_filter` | `keywordFilter` | yes | Sidebar facets and browse-by-value | `dc.type_filter` |
| `*_authority` | `keywordFilter` | yes | Authority key storage | `dc.contributor.author_authority` |
| `*_ac` | `keywordFilter` | yes | Autocomplete suggestions | `dc.title_ac` |
| `*_acid` | `keywordFilter` | yes | Autocomplete with ID (authority-linked) | `dc.contributor.author_acid` |
| `*_prefix` | `keywordFilter` | yes | Prefix-search facets (browse partial values) | `dc.title_prefix` |
| `*_partial` | `text` | yes | Full-text analysis for partial word matching | `dc.title_partial` |
| `*_hl` | `text` | yes | Hit highlighting | `dc.title_hl` |
| `*_mlt` | `text` | yes | More-Like-This (with term vectors, positions, offsets) | `dc.description_mlt` |
| `*_min` | `text` | no | Minimum value for a field | `dc.date.issued_min` |
| `*_max` | `text` | no | Maximum value for a field | `dc.date.issued_max` |
| `*_stored` | `dspaceMetadataProjection` | yes | Raw serialised metadata value for API projection | `dc.title_stored` |
| `*.year` | `sint` | yes | Year extracted from date fields | `dc.date.issued.year` |
| `*_dt` | `date` | no | Date-typed field with docValues (for sorting/range) | `dc.date.issued_dt` |

### Relation and CRIS fields

| Pattern | Type | Description |
|---|---|---|
| `relation.*` | `keywordFilter` | CRIS entity relationship metadata stored as keyword |
| `metric.*` | `double` | Bibliometric/altmetric values (e.g. citation counts) |
| `metric.acquisitionDate.*` | `date` | When the metric was acquired |
| `metric.id.*` | `int` | Metric record ID |
| `metric.remark.*` | `string` | Human-readable metric remark |
| `metric.deltaPeriod1.*` | `double` | Metric change over period 1 |
| `metric.deltaPeriod2.*` | `double` | Metric change over period 2 |
| `metric.rank.*` | `double` | Metric ranking value |

### Type-shorthand dynamic fields (legacy)

These map to primitive types for compatibility with Solr convention:

| Pattern | Type | | Pattern | Type |
|---|---|---|---|---|
| `*_i`, `*_ti` | `int` | | `*_l`, `*_tl` | `long` |
| `*_f`, `*_tf` | `float` | | `*_d`, `*_td`, `*_td` | `double` |
| `*_s` | `string` | | `*_t` | `text` |
| `*_b` | `boolean` | | `*_pi` | `pint` |

### Default catchall

```xml
<dynamicField name="*" type="text" multiValued="true"/>
```

Any field not matched by a more specific pattern falls through to `text` (full analysis pipeline with stemming). This is what `dc.title`, `dc.contributor.author`, `entityType`, and most DSpace metadata fields use unless a more specific variant is indexed.

---

## Copy Fields

Copy fields automatically populate one field from another at index time — no additional work by the DSpace indexer is needed.

| Source | Destination | Purpose |
|---|---|---|
| `*` (all fields) | `search_text` | Catch-all full-text field — everything is searchable via `q=...` without specifying a field |
| `lastModified` | `lastModified_dt` | Creates the `date`-typed docValues copy needed for date sorting |
| `title` | `itemauthoritylookup` | Enables authority-aware item lookup by title |
| `title` | `itemauthoritylookupexactmatch` | Enables exact-match title lookup for authority linking |
| `dspace.entity.type` | `search.entitytype` | Explicit typed copy to the declared `search.entitytype` field |
| `fulltext` | `a_spell` | Provides content to the spellcheck index |
| `fulltext` | `fulltext_hl` | Dedicated stored+indexed copy for hit-highlighting (avoids loading `fulltext` for every query) |

---

## Text Analysis Configuration Files

### `stopwords.txt`

Applied by `StopFilterFactory` in the `text`, `textgen`, `textTight`, and `textSpell` field types during both indexing and querying.

**Current stopwords** (English, standard set):

```
an  and  are  as  at  be  but  by
for  if  in  into  is  it  no  not
of  on  or  s  such  t  that  the
their  then  there  these  they  this
to  was  will  with
```

Plus two test tokens: `stopworda`, `stopwordb`.

**Practical implications:**

- Searching for `of` or `the` returns no results (the terms are removed before matching)
- Proximity queries like `"university of music"~3` — the stopword `of` is removed, reducing the effective word distance
- The missing prepositions (`from`, `about`, `with`) means they ARE indexed and searchable
- Single-letter tokens `s` and `t` are removed (handles possessives like `Mozart's` → `Mozart`)

**Adding custom stopwords:**

Edit `stopwords.txt` and reload the core:

```bash
# Edit the file in the container
docker compose -f docker-compose_2024.yml exec dspacesolr \
  vi /var/solr/data/search/conf/stopwords.txt

# Reload the core (no restart needed)
curl "http://localhost:8983/solr/admin/cores?action=RELOAD&core=search"
```

### `synonyms.txt`

Applied by `SynonymFilterFactory` in the `text` and `textTight` query analysers, and in the `textSpell` index analyser. In the `text` query analyser, `expand=true` means all synonym forms are searched simultaneously.

**Current synonym groups:**

| Entry | Type | Meaning |
|---|---|---|
| `GB,gib,gigabyte,gigabytes` | Equivalence | All forms match each other |
| `MB,mib,megabyte,megabytes` | Equivalence | All forms match each other |
| `Television,Televisions,TV,TVs` | Equivalence | All forms match each other |
| `aaa => aaaa` | Mapping | `aaa` at query time maps to `aaaa` |
| `pixima => pixma` | Mapping | Spelling correction via synonym |

The default file contains test entries only. For a production DSpace CRIS in a research context, adding domain-specific synonyms is highly valuable:

```
# Example additions for research repository context
doi,digital object identifier
orcid,open researcher contributor id
preprint,working paper,technical report
```

**`expand=true` vs `expand=false`:**

The `text` query analyser uses `expand=true`: a search for `TV` also searches for `Television`, `Televisions`, `TVs`. The `textTight` analyser uses `expand=false`: `TV` maps to a single canonical form only.

### `protwords.txt`

Applied by `SnowballPorterFilterFactory` (English stemmer) in the `text` and `textTight` field types. Words in this file are passed through the stemmer unchanged.

**Current entries:** `dontstems`, `zwhacky` (test entries only — no production words protected).

**When to add words:** When two words stem to the same base but have different meanings, or when a technical term should not be stemmed. For example:

```
# Example additions for music/arts research context
arts       # "arts" and "art" are different in context
performing
proceedings
```

---

## How a DSpace Metadata Field Gets Indexed

When DSpace indexes an item, each metadata value (e.g. `dc.title = "Music and Performing Arts"`) produces multiple index entries across different dynamic field variants. Here is the complete set for a typical metadata field:

```
dc.title                    → text        (full analysis: tokenised, stemmed, ICU-folded)
dc.title_sort               → lowerCaseSort (single token: "music and performing arts")
dc.title_keyword            → keywordFilter (exact: "Music and Performing Arts", case-preserved)
dc.title_filter             → keywordFilter (same as _keyword, used for facets)
dc.title_authority          → keywordFilter (authority key if linked to authority)
dc.title_ac                 → keywordFilter (autocomplete)
dc.title_hl                 → text        (for hit highlighting)
dc.title_partial            → text        (for partial-word browse)
```

Plus the value is copied into `search_text` via the catch-all `copyField source="*"`.

This explains the query patterns from the [Solr Query Reference]({{ '/ops/solr-queries/' | relative_url }}):

| Goal | Field to query | Why |
|---|---|---|
| Full-text relevance search | `dc.title` or `search_text` | `text` type — tokenised, stemmed, synonym-expanded |
| Exact full-value match | `dc.title_sort` or `dc.title_keyword` | `lowerCaseSort` / `keywordFilter` — single token |
| Sorting | `dc.title_sort` | `lowerCaseSort`, single-valued (`multiValued="false"`) |
| Faceted browsing | `dc.title_filter` | `keywordFilter`, preserves display form |
| Autocomplete | `dc.title_ac` | `keywordFilter`, used by Discovery suggest handler |

---

## Schema Inspection Queries

```bash
# List all explicit static fields
curl "http://localhost:8983/solr/search/schema/fields?wt=json" | \
  jq '.fields[] | {name: .name, type: .type, multiValued: .multiValued, stored: .stored}'

# List all dynamic field patterns
curl "http://localhost:8983/solr/search/schema/dynamicfields?wt=json" | \
  jq '.dynamicFields[] | {name: .name, type: .type}'

# Inspect a specific field's resolved type and properties
curl "http://localhost:8983/solr/search/schema/fields/dc.title_sort?wt=json&indent=true"

# Show all field types and their analyser chains
curl "http://localhost:8983/solr/search/schema/fieldtypes?wt=json&indent=true" | \
  jq '.fieldTypes[] | {name: .name, class: .class}'

# Analyse how a value gets tokenised for a specific field type
# (useful for debugging why a query does or does not match)
curl "http://localhost:8983/solr/search/analysis/field?analysis.fieldtype=text&analysis.fieldvalue=Music+and+Performing+Arts+Vienna&wt=json&indent=true"

# Analyse index vs query tokens for a field
curl "http://localhost:8983/solr/search/analysis/document" \
  --data-urlencode 'analysis.query=performing arts' \
  --data-urlencode 'analysis.fieldname=dc.title' \
  -G --data "wt=json&indent=true"

# See which fields actually exist in the index (Luke handler)
curl "http://localhost:8983/solr/search/admin/luke?numTerms=0&wt=json&indent=true" | \
  jq '.fields | keys'

# Top terms in a field (useful for facet planning)
curl "http://localhost:8983/solr/search/admin/luke?fl=entityType&numTerms=20&wt=json&indent=true" | \
  jq '.fields.entityType.topTerms'
```

---

## Common Configuration Modifications

### Adding stopwords

```bash
# Edit stopwords.txt and reload — no reindex needed
echo "vienna" >> /path/to/search/conf/stopwords.txt
curl "http://localhost:8983/solr/admin/cores?action=RELOAD&core=search"
```

Note: only affects new documents indexed after reload. To apply to existing documents, a full reindex is required.

### Adding synonyms

```bash
# Edit synonyms.txt and reload — applies to queries immediately (query-time synonyms)
# No reindex needed for query-time synonyms (text query analyser)
echo "mdw, university of music, musik universität wien" >> synonyms.txt
curl "http://localhost:8983/solr/admin/cores?action=RELOAD&core=search"
```

### Increasing cache sizes

Edit `solrconfig.xml` and reload the core. For a deployment with high query volume and stable entity type + archive status filters:

```xml
<!-- Increase filterCache for better fq= cache hit ratio -->
<filterCache class="solr.search.CaffeineCache"
             size="1024"
             initialSize="512"
             autowarmCount="0"/>
```

### Extending the slow query threshold

The current threshold is 1000ms. Reduce it to catch more slow queries during performance tuning:

```xml
<slowQueryThresholdMillis>500</slowQueryThresholdMillis>
```

---

## Reference

| Resource | Link |
|---|---|
| Solr 8.11 schema design | [solr.apache.org/guide/8_11/schema-elements](https://solr.apache.org/guide/8_11/schema-elements.html) |
| Field types in Solr 8 | [solr.apache.org/guide/8_11/field-types-included-with-solr](https://solr.apache.org/guide/8_11/field-types-included-with-solr.html) |
| Solr analysers / tokenisers / filters | [solr.apache.org/guide/8_11/understanding-analyzers-tokenizers-and-filters](https://solr.apache.org/guide/8_11/understanding-analyzers-tokenizers-and-filters.html) |
| ICU Folding Filter | [solr.apache.org/guide/8_11/language-analysis](https://solr.apache.org/guide/8_11/language-analysis.html#icu-folding-filter) |
| Solr caches | [solr.apache.org/guide/8_11/query-settings-in-solrconfig](https://solr.apache.org/guide/8_11/query-settings-in-solrconfig.html) |
| Solr copy fields | [solr.apache.org/guide/8_11/copying-fields](https://solr.apache.org/guide/8_11/copying-fields.html) |
| Snowball Porter stemmer | [snowballstem.org](https://snowballstem.org/algorithms/english/stemmer.html) |
| Solr synonyms | [solr.apache.org/guide/8_11/filter-descriptions](https://solr.apache.org/guide/8_11/filter-descriptions.html#synonym-filter) |
| DSpace Discovery configuration | [wiki.lyrasis.org/display/DSDOC7x/Discovery](https://wiki.lyrasis.org/display/DSDOC7x/Discovery) |
| `ClassicIndexSchemaFactory` | [solr.apache.org/guide/8_11/schema-factory-definition-in-solrconfig](https://solr.apache.org/guide/8_11/schema-factory-definition-in-solrconfig.html) |

<div class="page-nav">
  <a href="{{ '/ops/solr-queries/' | relative_url }}">← Solr Query Reference</a>
  <a href="{{ '/ops/solr9-upgrade/' | relative_url }}">Solr 9 — DSpace 2025 Upgrade →</a>
</div>
