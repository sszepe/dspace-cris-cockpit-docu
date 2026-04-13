---
layout: page
title: Solr Query Reference
permalink: /ops/solr-queries/
parent: Operations Guide
---

# Solr Query Reference & Performance

![Solr](https://img.shields.io/badge/Solr-8.11.4-d9411e?style=flat-square&logo=apachesolr&logoColor=white)
![Access](https://img.shields.io/badge/access-internal%20only-orange?style=flat-square)

<div class="callout callout-warn">
<span class="callout-title">Internal access only</span>
Solr is not exposed to the public internet. The <code>dspacesolr</code> container binds to <code>:8983</code> on the internal Docker network only. Direct Solr queries are available to ops, admin, and developer roles with access to the Docker host or a forwarded port. End users access the search index exclusively through the DSpace Discovery API — see <a href="#querying-via-the-dspace-api">Querying via the DSpace API</a>.
</div>

All examples use the `search` core (`solr:8983/solr/search/select`), which is the primary full-text index for DSpace CRIS items. Other available cores are `authority`, `statistics`, `oai`, `qaevent`, `suggestion`, `dedup`, and `audit`.

---

## Accessing Solr

```bash
# Port-forward from the Docker host for local access
docker compose -f docker-compose_2024.yml exec dspacesolr bash

# Or forward the port to your local machine (use with caution — dev only)
docker compose -f docker-compose_2024.yml port dspacesolr 8983

# Solr admin UI
open http://localhost:8983/solr/#/search/query

# Quick health check
curl -s "http://localhost:8983/solr/search/admin/ping" | jq .status
```

---

## Query Basics

### All documents in the index

```
GET http://solr:8983/solr/search/select?q=*:*
```

Returns all indexed documents with default pagination (10 rows). The `numFound` field in the response header shows the total document count.

### Pagination

Solr uses `start` (offset) and `rows` (page size) rather than `page`/`size`:

```
# First 5 documents starting at offset 10
GET http://solr:8983/solr/search/select?q=*:*&start=10&rows=5

# Limit to 1 result (useful for count checks)
GET http://solr:8983/solr/search/select?q=*:*&rows=1

# Return 0 rows — only get the count
GET http://solr:8983/solr/search/select?q=*:*&rows=0
```

### Field:value queries

```
# Match a specific field value
GET http://solr:8983/solr/search/select?q=schema_keyword:dc

# All archived (published) items
GET http://solr:8983/solr/search/select?q=archived:true

# All items of a specific entity type
GET http://solr:8983/solr/search/select?q=entityType:Person
```

### Select response fields (`fl`)

Reduce response size by specifying which fields to return:

```
# Return only dc.title for archived items
GET http://solr:8983/solr/search/select?q=archived:true&fl=dc.title

# Return multiple fields
GET http://solr:8983/solr/search/select?q=archived:true&fl=dc.title,entityType,handle,score

# Always include score if you want relevance ranking info
GET http://solr:8983/solr/search/select?q=dc.title:Szepe&fl=dc.title,score
```

### Pretty-print JSON response

Add `&wt=json&indent=true` for readable output in the browser or curl:

```
GET http://solr:8983/solr/search/select?q=archived:true&rows=2&wt=json&indent=true
```

---

## Boolean Operators

Solr supports standard boolean syntax. Operators must be **uppercase**.

### AND — both conditions must match

```
GET http://solr:8983/solr/search/select?q=archived:true AND entityType:Person
```

URL-encoded (required in curl / scripts):

```bash
curl "http://solr:8983/solr/search/select?q=archived:true%20AND%20entityType:Person&fl=dc.title,entityType"
```

### OR — either condition matches

```
GET http://solr:8983/solr/search/select?q=entityType:Person OR entityType:Project
```

### NOT — exclude matching documents

```
# Archived items that are NOT of type Person
GET http://solr:8983/solr/search/select?q=archived:true NOT entityType:Person
```

### `+` (MUST) and `-` (MUST NOT) — inline term operators

These apply per-term within a field value and are equivalent to AND/NOT respectively:

```
# dc.title MUST contain both "Szepe" AND "Stefan"
GET http://solr:8983/solr/search/select?q=dc.title:(+Szepe +Stefan)

# dc.title MUST contain "Szepe" but MUST NOT contain "foo"
GET http://solr:8983/solr/search/select?q=dc.title:(+Szepe -foo)
```

### Combining operators

```
# Archived items where entityType is Person OR Project
GET http://solr:8983/solr/search/select?q=archived:true AND entityType:(Person OR Project)

# Archived Publications with a specific author
GET http://solr:8983/solr/search/select?q=archived:true AND entityType:Publication AND dc.contributor.author:Szepe
```

---

## String Matching

### Exact match — field type matters

Standard text fields (e.g. `dc.title`) use a tokenising analyser. Quoted phrases match adjacent tokens but do not enforce an exact full-value match:

```
# Phrase match — finds documents where "Szepe," and "Stefan" appear adjacent
# ⚠ May not work as expected on tokenised fields
GET http://solr:8983/solr/search/select?q=dc.title:"Szepe, Stefan"
```

For **true exact string matching**, use a `_keyword` or `_sort` variant of the field, which uses the `KeywordTokenizer` (no tokenisation):

```
# Exact match on dc.title_sort — reliable for full-value equality
GET http://solr:8983/solr/search/select?q=dc.title_sort:"Szepe, Stefan"

# Exact match on a keyword field
GET http://solr:8983/solr/search/select?q=birthDate_keyword:"1974-04-17"
```

<div class="callout callout-info">
<span class="callout-title">Which fields support exact matching?</span>
Fields typed as <code>solr.StrField</code> or using <code>KeywordTokenizerFactory</code> in the schema support exact matching. In DSpace CRIS, these are typically the <code>_keyword</code> and <code>_sort</code> suffixed variants. Check the field type in the Solr admin UI: <strong>Core Selector → search → Schema → [field name] → Field Type</strong>, or query the schema API:
<pre><code>curl "http://solr:8983/solr/search/schema/fields/dc.title_sort"</code></pre>
</div>

### Wildcard

```
# Prefix wildcard — titles starting with "Sze"
GET http://solr:8983/solr/search/select?q=dc.title:Sze*

# Contains wildcard — titles with "niversit" anywhere
GET http://solr:8983/solr/search/select?q=dc.title:*niversit*
```

<div class="callout callout-warn">
<span class="callout-title">Leading wildcards are expensive</span>
Queries like <code>*niversit*</code> (leading wildcard) require a full index scan and are significantly slower than <code>Sze*</code>. Avoid leading wildcards on large indexes or use <code>ReversedWildcardFilterFactory</code> if the field schema supports it.
</div>

### Fuzzy search (edit distance)

Finds terms within a specified edit distance (number of single-character edits):

```
# Match "Uni" with up to 2 character substitutions/insertions/deletions
GET http://solr:8983/solr/search/select?q=dc.title:Uni~2
```

The `~N` suffix sets the maximum [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance). Values of 1 or 2 are typical; higher values are increasingly expensive and imprecise.

### Proximity search

Finds two terms within a specified word distance of each other:

```
# "university" and "arts" within 4 words of each other
GET http://solr:8983/solr/search/select?q=dc.title:"university arts"~4
```

<div class="callout callout-info">
<span class="callout-title">Stopwords are removed before proximity counting</span>
The analyser removes stopwords before measuring distance. For example, the phrase <em>"university of music and performing arts vienna"</em> becomes <em>"university music performing arts vienna"</em> after stopword removal (see <code>stopwords.txt</code> in the Solr core config). A proximity query of <code>~4</code> would match this phrase even though "arts" appears further than 4 positions away in the original text.
</div>

---

## Range Queries

Range queries use the syntax `[lower TO upper]`. Use `*` for open-ended bounds.

### Date ranges

```
# Items indexed after a specific timestamp
GET http://solr:8983/solr/search/select?q=SolrIndexer.lastIndexed:[2023-11-23T14:03:51.848Z TO *]

# Items indexed before a specific timestamp
GET http://solr:8983/solr/search/select?q=SolrIndexer.lastIndexed:[* TO 2023-11-23T14:03:51.848Z]

# Items indexed within a date range
GET http://solr:8983/solr/search/select?q=SolrIndexer.lastIndexed:[2023-11-20T00:00:00.000Z TO 2023-11-23T14:03:51.848Z]

# Items indexed in the last 24 hours (using Solr date math)
GET http://solr:8983/solr/search/select?q=SolrIndexer.lastIndexed:[NOW-24HOURS TO NOW]

# Items indexed in the last 7 days
GET http://solr:8983/solr/search/select?q=SolrIndexer.lastIndexed:[NOW-7DAYS TO NOW]
```

<div class="callout callout-info">
<span class="callout-title">Solr date math</span>
Solr supports relative date expressions like <code>NOW-24HOURS</code>, <code>NOW-7DAYS</code>, <code>NOW/DAY</code> (start of today). These are more robust than hardcoded timestamps in recurring scripts. See the <a href="https://solr.apache.org/guide/8_11/working-with-dates.html" target="_blank">Solr date math docs</a>.
</div>

### Integer / numeric ranges

```
# Persons born after 1986
GET http://solr:8983/solr/search/select?q=birthDate:[1986 TO *]

# Persons born before 1986
GET http://solr:8983/solr/search/select?q=birthDate:[* TO 1986]

# Persons born between 1972 and 1986 (inclusive)
GET http://solr:8983/solr/search/select?q=birthDate:[1972 TO 1986]

# Exclusive bounds use curly braces
# Born strictly between 1972 and 1986 (1972 and 1986 not included)
GET http://solr:8983/solr/search/select?q=birthDate:{1972 TO 1986}
```

---

## Relevance Tuning

### Boosting

Boost a term or clause to increase its contribution to the relevance score. A boost factor of `2` means the boosted clause counts twice as much:

```
# entityType:Project scores twice as much as entityType:Person
GET http://solr:8983/solr/search/select?q=archived:true AND entityType:(Person OR Project^2)

# Boost an exact phrase match over a single-term match
GET http://solr:8983/solr/search/select?q=dc.title:music^1 OR dc.title:"music and performing arts"^3
```

### Field-level boosting in `qf` (when using `edismax`)

Switch to the `edismax` query parser for multi-field search with per-field boosts:

```
GET http://solr:8983/solr/search/select
  ?defType=edismax
  &q=music+arts
  &qf=dc.title^3+dc.description^1
  &fl=dc.title,score
  &rows=10
```

| Parameter | Purpose |
|---|---|
| `defType=edismax` | Use the Extended DisMax query parser |
| `qf` | Query fields with optional boosts (`field^boost`) |
| `pf` | Phrase fields — extra boost when all query terms appear as a phrase |
| `mm` | Minimum should-match percentage/count |
| `bf` | Additive boost functions (e.g. boost by date recency) |

---

## Querying via the DSpace API

The DSpace Discovery API proxies Solr queries through its HAL REST interface. These are the equivalent queries accessible to the frontend and external consumers without direct Solr access.

```
# Equivalent to: q=archived:true AND entityType:Person
GET https://<dspace-host>/server/api/discover/search/objects
    ?q=archived:true%20AND%20entityType:Person

# With field filter and sorting
GET https://<dspace-host>/server/api/discover/search/objects
    ?q=entityType:Publication
    &sort=dc.date.issued,DESC
    &size=10
    &page=0
```

Key differences from direct Solr queries:

| Aspect | Direct Solr | DSpace Discovery API |
|---|---|---|
| Access | Internal Docker network only | Public HTTPS |
| Auth | None (network-level security) | DSpace JWT for restricted items |
| Response format | Solr JSON / XML | HAL+JSON with embedded links |
| Field selection | `fl` parameter | Fixed set of exposed fields |
| Facets | Full Solr faceting | Pre-configured Discovery facets (`discovery.xml`) |
| Score | Available via `fl=score` | Not exposed |

---

## Useful Admin Queries

### Core statistics

```bash
# Document count and index size for all cores
curl "http://solr:8983/solr/admin/cores?action=STATUS&wt=json" | \
  jq '.status | to_entries[] | {core: .key, numDocs: .value.index.numDocs, size: .value.index.size}'

# Just the search core
curl "http://solr:8983/solr/search/admin/luke?numTerms=0&wt=json" | \
  jq '{numDocs: .index.numDocs, maxDoc: .index.maxDoc, deletedDocs: .index.deletedDocs}'
```

### Schema inspection

```bash
# List all field names in the search core
curl "http://solr:8983/solr/search/schema/fields?wt=json" | \
  jq '.fields[].name'

# Field type details for a specific field
curl "http://solr:8983/solr/search/schema/fields/dc.title?wt=json"
curl "http://solr:8983/solr/search/schema/fields/dc.title_sort?wt=json"

# List all field types (shows tokenisers and filters)
curl "http://solr:8983/solr/search/schema/fieldtypes?wt=json" | \
  jq '.fieldTypes[].name'
```

### Check if a specific document is indexed

```bash
# Look up a DSpace item by UUID
curl "http://solr:8983/solr/search/select?q=search.resourceid:<uuid>&fl=search.resourceid,archived,entityType,dc.title&wt=json&indent=true"
```

### Find recently modified documents

```bash
# Documents indexed in the last hour
curl "http://solr:8983/solr/search/select?q=SolrIndexer.lastIndexed:[NOW-1HOUR%20TO%20NOW]&fl=dc.title,SolrIndexer.lastIndexed&rows=20&sort=SolrIndexer.lastIndexed+desc&wt=json&indent=true"
```

---

## Performance Monitoring

### 1. Solr Admin UI — built-in metrics

The Solr Admin UI at `http://localhost:8983/solr/#/search` provides real-time metrics without any additional setup:

```
Core Selector → search → Plugins/Stats
```

Key panels to watch:

| Panel | What to look at |
|---|---|
| **Query Handler `/select`** | `requestTimes` — avgTimePerRequest, 75th/95th/99th percentile; `requests` — total and errors |
| **Update Handler `/update`** | `commits` — number and time; `autoCommits`; pending docs |
| **Cache** | `filterCache`, `queryResultCache`, `documentCache` — hit ratio, evictions |
| **JVM** | Heap used / max — high heap pressure causes GC pauses that spike query latency |

### 2. Solr Metrics API

Query the metrics programmatically for scripting or Prometheus scraping:

```bash
# All metrics for the search core
curl "http://solr:8983/solr/admin/metrics?group=core&prefix=QUERY&wt=json" | jq .

# Request count and timing for the /select handler
curl "http://solr:8983/solr/admin/metrics?group=core&key=solr.core.search:QUERY./select.requestTimes&wt=json" | \
  jq '.metrics."solr.core.search:QUERY./select.requestTimes"'

# Cache hit ratios
curl "http://solr:8983/solr/admin/metrics?group=core&prefix=CACHE.searcher&wt=json" | jq .

# JVM heap
curl "http://solr:8983/solr/admin/metrics?group=jvm&prefix=memory.heap&wt=json" | jq .
```

### 3. Slow query log

Solr can log queries that exceed a threshold. Add to `solrconfig.xml` inside the `<requestHandler name="/select">` block:

```xml
<requestHandler name="/select" class="solr.SearchHandler">
  <!-- Log queries slower than 1000ms -->
  <str name="slowQueryThresholdMillis">1000</str>
  ...
</requestHandler>
```

Then watch the slow query log in Loki:

```logql
# Solr slow queries (from container logs)
{service="dspacesolr"} |= "QTime=" | regexp "QTime=(?P<qtime>[0-9]+)" | qtime > 1000

# All Solr queries with their QTime
{service="dspacesolr"} |~ "QTime=[0-9]+"

# Solr errors
{service="dspacesolr"} |= "ERROR"
```

### 4. Query performance via `debugQuery`

Add `&debugQuery=true` to any query to get a full explanation of how Solr scored and executed it:

```bash
curl "http://solr:8983/solr/search/select?q=dc.title:music&fl=dc.title,score&debugQuery=true&wt=json&indent=true" | \
  jq '{timing: .debug.timing, explain: .debug.explain}'
```

The `timing` block shows how long each phase took (prepare, process, response). The `explain` block shows how each document's score was calculated — useful for diagnosing unexpected ranking.

### 5. Prometheus + Grafana (monitoring stack)

When running the monitoring stack (`docker-compose_2024-monitoring.yml`), the Blackbox Exporter probes the four main Solr core ping endpoints every 15 seconds:

```promql
# All Solr core availability
probe_success{job="solr_health"}

# Specific core
probe_success{job="solr_health", instance="http://dspacesolr:8983/solr/search/admin/ping"}
```

For deeper Solr JMX/metrics scraping, add the [solr_exporter](https://solr.apache.org/guide/8_11/monitoring-solr-with-prometheus-and-grafana.html) to the monitoring stack:

```yaml
# Add to docker-compose_2024-monitoring.yml
solr-exporter:
  image: solr:8.11.4
  command: >
    /opt/solr/contrib/prometheus-exporter/bin/solr-exporter
    -p 9854
    -b http://dspacesolr:8983/solr
    -f /opt/solr/contrib/prometheus-exporter/conf/solr-exporter-config.xml
    -n 8
  ports:
    - "9854:9854"
  networks:
    - dspacenet
    - monitoring
```

Then add a scrape target in `monitoring/prometheus/prometheus.yml`:

```yaml
  - job_name: solr_exporter
    static_configs:
      - targets: ['solr-exporter:9854']
        labels:
          service: dspacesolr
```

This exposes metrics like `solr_requests_total`, `solr_query_response_time_ms_bucket` (histogram), `solr_cache_hitratio`, `solr_jvm_heap_used_bytes`, and many more. Import the [official Solr Grafana dashboard](https://grafana.com/grafana/dashboards/12456) (ID `12456`) to visualise them.

### 6. Index health checks

Run these periodically or after a reindex to verify the search core is consistent:

```bash
# Check for uncommitted pending documents
curl "http://solr:8983/solr/search/admin/mbeans?cat=UPDATEHANDLER&stats=true&wt=json" | \
  jq '.["solr-mbeans"][1].updateHandler.stats | {pendingDocs: .pendingDocs, cumulative_adds: ."cumulative_adds"}'

# Optimize the index (merges segments — run during low-traffic window)
curl "http://solr:8983/solr/search/update?optimize=true&waitFlush=true"

# Force a full reindex from DSpace
docker compose -f docker-compose_2024.yml exec dspace \
  /dspace/bin/dspace index-discovery -f

# Reindex a specific item by UUID
docker compose -f docker-compose_2024.yml exec dspace \
  /dspace/bin/dspace index-discovery -i <uuid>
```

### Performance baseline — key metrics to track

| Metric | Healthy baseline | Alert threshold |
|---|---|---|
| `/select` avg request time | < 100ms | > 500ms |
| `/select` 95th percentile | < 300ms | > 2000ms |
| `filterCache` hit ratio | > 80% | < 60% |
| `queryResultCache` hit ratio | > 70% | < 50% |
| JVM heap used | < 75% of `-Xmx` | > 85% |
| Pending uncommitted docs | < 1000 | > 10 000 |
| Index segment count | < 20 | > 50 (needs optimize) |

---

## Reference

| Resource | Link |
|---|---|
| Solr 8.11 query syntax | [solr.apache.org/guide/8_11/the-standard-query-parser](https://solr.apache.org/guide/8_11/the-standard-query-parser.html) |
| edismax query parser | [solr.apache.org/guide/8_11/the-extended-dismax-query-parser](https://solr.apache.org/guide/8_11/the-extended-dismax-query-parser.html) |
| Solr date math | [solr.apache.org/guide/8_11/working-with-dates](https://solr.apache.org/guide/8_11/working-with-dates.html) |
| Solr field types | [solr.apache.org/guide/8_11/field-types-included-with-solr](https://solr.apache.org/guide/8_11/field-types-included-with-solr.html) |
| Solr caches | [solr.apache.org/guide/8_11/query-settings-in-solrconfig](https://solr.apache.org/guide/8_11/query-settings-in-solrconfig.html#query-result-cache) |
| Solr metrics API | [solr.apache.org/guide/8_11/metrics-reporting](https://solr.apache.org/guide/8_11/metrics-reporting.html) |
| Prometheus + Grafana for Solr | [solr.apache.org/guide/8_11/monitoring-solr-with-prometheus-and-grafana](https://solr.apache.org/guide/8_11/monitoring-solr-with-prometheus-and-grafana.html) |
| Grafana Solr dashboard (ID 12456) | [grafana.com/grafana/dashboards/12456](https://grafana.com/grafana/dashboards/12456) |
| DSpace Discovery configuration | [wiki.lyrasis.org/display/DSDOC7x/Discovery](https://wiki.lyrasis.org/display/DSDOC7x/Discovery) |

<div class="page-nav">
  <a href="{{ '/ops/database-performance/' | relative_url }}">← Database Performance</a>
  <a href="{{ '/ops/solr-search-core/' | relative_url }}">Solr Search Core Schema →</a>
</div>
