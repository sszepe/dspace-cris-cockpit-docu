---
layout: page
title: Solr 9 — New Features
permalink: /ops/solr9-features/
parent: Operations Guide
---

# Solr 9 — New Features & Capabilities

![Solr](https://img.shields.io/badge/Solr-9.x-d9411e?style=flat-square&logo=apachesolr&logoColor=white)
![Lucene](https://img.shields.io/badge/Lucene-9.x-007396?style=flat-square)

Solr 9 introduces several capabilities that go significantly beyond search and indexing as DSpace has historically used them. This page documents the most relevant new features for a DSpace CRIS deployment, with a concrete worked example of the `UpdateRequestProcessor` scripting pipeline for derived field calculation.

<div class="callout callout-info">
<span class="callout-title">These features are Solr 9 only</span>
All features on this page require the Solr 9 stack (<code>docker-compose_2025.yml</code>). They are not available in the Solr 8 stack used by DSpace 2024. See the <a href="{{ '/ops/solr9-upgrade/' | relative_url }}">Solr 9 Upgrade</a> page for migration steps.
</div>

---

## Feature Overview

| Feature | What it enables for DSpace |
|---|---|
| [Scripted UpdateRequestProcessor](#scripted-updaterequestprocessor) | Calculate derived fields (e.g. percentage sums) at index time via JavaScript |
| [StatelessScriptUpdateProcessorFactory](#statelessscriptupdateprocessorfactory) | Thread-safe variant for high-concurrency indexing |
| [Dense Vector Search (KNN)](#dense-vector-search-knn) | Semantic / embedding-based similarity search alongside keyword search |
| [Improved Highlighting](#improved-highlighting) | `UnifiedHighlighter` — faster, more accurate hit highlighting with `HighlightComponent` |
| [Nested / Child Documents](#nested--child-documents) | Index structured hierarchies (e.g. item → bundles → bitstreams) as nested Solr docs |
| [Payload-Scored Fields](#payload-scored-fields) | Embed numeric weights directly in the index for boost-at-query-time |
| [Improved Streaming Expressions](#streaming-expressions) | SQL-like aggregation and joins over Solr data without external tooling |

---

## Scripted UpdateRequestProcessor

### What it does

An `UpdateRequestProcessor` intercepts every document before it is written to the Solr index. A scripted variant lets you execute JavaScript (via Java's Nashorn/GraalJS engine) against each document during indexing — reading existing fields, performing calculations, and writing new derived fields. This happens transparently: the DSpace indexer sends documents as normal, and Solr enriches them before storage.

### Use case: `dc.subject.percentagecalculated`

DSpace CRIS items can carry percentage values in `dc.subject.percentage` (multi-valued — one value per contributor or subject). A common reporting requirement is to verify or flag items where the percentages do not sum to 100. Rather than computing this in every application layer, we calculate it once at index time and store the result in `dc.subject.percentagecalculated`.

```
dc.subject.percentage: ["40", "35", "25"]
                              ↓ JavaScript at index time
dc.subject.percentagecalculated: 100
```

---

### Step 1 — Schema fields (`schema.xml`)

Add two fields to the search core schema. If `dc.subject.percentage` is already indexed by DSpace as a dynamic field fallback, the explicit declaration here makes it typed and stored correctly. `dc.subject.percentagecalculated` is the new derived field.

```xml
<!-- Percentage values contributed by DSpace metadata (multi-valued) -->
<field name="dc.subject.percentage"
       type="string"
       indexed="true"
       stored="true"
       multiValued="true"
       omitNorms="true"
       docValues="true" />

<!-- Derived field: sum of all dc.subject.percentage values, calculated at index time -->
<field name="dc.subject.percentagecalculated"
       type="int"
       indexed="true"
       stored="true"
       omitNorms="true"
       docValues="true" />
```

<div class="callout callout-info">
<span class="callout-title">Why type="string" for the input field?</span>
DSpace metadata values are always strings. Using <code>type="string"</code> (exact, no tokenisation) for <code>dc.subject.percentage</code> preserves the raw values for the JavaScript processor to parse with <code>parseInt()</code>. The calculated result is stored as <code>type="int"</code> for efficient range queries.
</div>

---

### Step 2 — JavaScript processor (`dc-percentage-calculator.js`)

Place this file in the Solr `conf/` directory of the `search` core (alongside `schema.xml`):

```
/var/solr/data/search/conf/
├── schema.xml
├── solrconfig.xml
└── dc-percentage-calculator.js     ← here
```

```javascript
/**
 * dc-percentage-calculator.js
 *
 * Solr UpdateRequestProcessor script.
 * Calculates the sum of dc.subject.percentage values and stores the result
 * in dc.subject.percentagecalculated before the document is committed.
 *
 * Triggered on: document INSERT and UPDATE
 * Required Solr: 9.x (StatelessScriptUpdateProcessorFactory)
 */

function processAdd(cmd) {
    var doc = cmd.solrDoc;

    // Check if dc.subject.percentage is present on this document
    var percentages = doc.getFieldValues("dc.subject.percentage");
    if (!percentages || percentages.size() === 0) {
        return;  // Nothing to calculate — leave the field unset
    }

    var total = 0;
    var hasInvalid = false;

    for (var i = 0; i < percentages.size(); i++) {
        var val = parseInt(percentages.get(i), 10);
        if (isNaN(val)) {
            hasInvalid = true;
            continue;  // Skip non-numeric values
        }
        total += val;
    }

    // Store the calculated sum
    doc.setField("dc.subject.percentagecalculated", total);

    // Optional: flag documents with non-numeric percentage values
    if (hasInvalid) {
        doc.setField("dc.subject.percentage_valid", "false");
    }
}

// processDelete and processMergeIndexes can be left unimplemented —
// Solr calls them only if defined, and they are not needed here.
```

**File permissions:**

```bash
# Inside the container — Solr must be able to read and execute the script
docker compose -f docker-compose_2025.yml exec dspacesolr \
  chmod 644 /var/solr/data/search/conf/dc-percentage-calculator.js
```

---

### Step 3 — `solrconfig.xml` configuration

Add the processor chain and wire it to the `/update` handler. This replaces the existing `/update` handler declaration in `solrconfig.xml`.

<div class="callout callout-warn">
<span class="callout-title">Use StatelessScriptUpdateProcessorFactory</span>
The correct Solr 9 class is <code>solr.StatelessScriptUpdateProcessorFactory</code>. The earlier <code>solr.ScriptUpdateProcessorFactory</code> uses a shared, mutable script engine state and is not thread-safe under concurrent indexing. The <code>Stateless</code> variant creates a fresh script execution context per document, which is safe and correct for field calculation scripts.
</div>

```xml
<!-- ── Scripted UpdateRequestProcessor chain ─────────────────────────── -->
<!-- Processes documents through the JavaScript calculator before indexing -->
<updateRequestProcessorChain name="percentageCalculatorChain">

  <!-- 1. Run the JavaScript processor -->
  <processor class="solr.StatelessScriptUpdateProcessorFactory">
    <str name="script">dc-percentage-calculator.js</str>
    <!-- To use a path outside conf/, specify the full path:
    <str name="script">/opt/solr/custom-scripts/dc-percentage-calculator.js</str>
    -->
  </processor>

  <!-- 2. Run the standard Solr update pipeline after the script -->
  <processor class="solr.RunUpdateProcessorFactory" />

</updateRequestProcessorChain>

<!-- ── /update handler — wire to the processor chain ─────────────────── -->
<!-- Replace or extend the existing /update handler declaration -->
<requestHandler name="/update" class="solr.UpdateRequestHandler">
  <lst name="defaults">
    <str name="update.chain">percentageCalculatorChain</str>
  </lst>
</requestHandler>

<!-- ── /update/json — also apply the chain for JSON bulk updates ──────── -->
<requestHandler name="/update/json" class="solr.UpdateRequestHandler">
  <lst name="defaults">
    <str name="stream.contentType">application/json</str>
    <str name="update.chain">percentageCalculatorChain</str>
  </lst>
</requestHandler>
```

**Placement in `solrconfig.xml`:** Add the `<updateRequestProcessorChain>` block anywhere at the top level of `<config>`, before the closing `</config>` tag. The `/update` handler declaration replaces the existing one.

---

### Step 4 — Trigger a reindex

The script only runs when documents are (re-)indexed. After making the configuration changes and reloading the core, run a full reindex to populate `dc.subject.percentagecalculated` for all existing documents:

```bash
# 1. Copy your updated solrconfig.xml and dc-percentage-calculator.js into the container
docker cp dc-percentage-calculator.js \
  dspacesolr:/var/solr/data/search/conf/dc-percentage-calculator.js

# 2. Reload the search core (no restart needed for solrconfig.xml changes)
curl "http://localhost:8983/solr/admin/cores?action=RELOAD&core=search"

# 3. Verify the processor chain loaded correctly
curl "http://localhost:8983/solr/search/config/updateRequestProcessorChain?wt=json&indent=true"

# 4. Trigger a full reindex so the script runs on all existing documents
docker compose -f docker-compose_2025.yml exec dspace \
  /dspace/bin/dspace index-discovery -f

# 5. Verify — find documents with calculated percentages
curl "http://localhost:8983/solr/search/select?q=dc.subject.percentage:[*+TO+*]&fl=dc.title,dc.subject.percentage,dc.subject.percentagecalculated&rows=5&wt=json&indent=true"
```

---

### Querying the derived field

```bash
# All documents that have any percentage values
q=dc.subject.percentage:[* TO *]

# Documents where percentages sum exactly to 100 (complete allocation)
q=dc.subject.percentage:[* TO *] AND dc.subject.percentagecalculated:100

# Documents where percentages do NOT sum to 100 (data quality check)
q=dc.subject.percentage:[* TO *] AND -dc.subject.percentagecalculated:100

# Documents where total exceeds 100 (over-allocated)
q=dc.subject.percentage:[* TO *] AND dc.subject.percentagecalculated:[101 TO *]

# Documents where total is below 100 (under-allocated)
q=dc.subject.percentage:[* TO *] AND dc.subject.percentagecalculated:[* TO 99]

# Combine with entity type for targeted reporting
q=entityType:Project AND dc.subject.percentage:[* TO *] AND -dc.subject.percentagecalculated:100

# Group by calculated total (facet)
curl "http://localhost:8983/solr/search/select?q=dc.subject.percentage:[*+TO+*]&rows=0&facet=true&facet.field=dc.subject.percentagecalculated&facet.limit=20&wt=json&indent=true"
```

---

### `StatelessScriptUpdateProcessorFactory`

The stateless variant is the correct choice for production because it creates a new script engine context for each document batch. Key differences:

| | `ScriptUpdateProcessorFactory` | `StatelessScriptUpdateProcessorFactory` |
|---|---|---|
| Script engine state | Shared across threads | Fresh per batch — thread-safe |
| Global variable access | Supported | Not supported (stateless by design) |
| Performance overhead | Lower (shared state) | Slightly higher (new context per batch) |
| Thread safety | ❌ Not safe under concurrent indexing | ✅ Safe |
| Recommended for DSpace | ❌ | ✅ |

**Script placement options:**

```xml
<!-- Option 1: conf/ directory (default — relative path) -->
<str name="script">dc-percentage-calculator.js</str>

<!-- Option 2: Absolute path (for scripts managed outside the core) -->
<str name="script">/opt/solr/custom-scripts/dc-percentage-calculator.js</str>

<!-- Option 3: Multiple scripts in sequence -->
<arr name="script">
  <str>dc-percentage-calculator.js</str>
  <str>dc-validation.js</str>
</arr>
```

---

## Dense Vector Search (KNN)

Solr 9 introduces native approximate nearest-neighbour (KNN) vector search via the `DenseVectorField` type. This enables semantic similarity search alongside keyword search — a document can be retrieved based on embedding similarity rather than (or in addition to) term overlap.

### What it enables for DSpace CRIS

- Find publications similar to a given item based on abstract embeddings, not just shared keywords
- "More Like This" improvements — semantic rather than term-frequency-based
- Multilingual similarity search — embeddings from multilingual models bridge language gaps

### Schema setup

```xml
<!-- Dense vector field for storing text embeddings (e.g. from abstract) -->
<!-- 768 dimensions = typical BERT/sentence-transformers output -->
<fieldType name="knn_vector_768" class="solr.DenseVectorField"
           vectorDimension="768"
           similarityFunction="cosine"
           knnAlgorithm="hnsw"
           hnswMaxConnections="16"
           hnswBeamWidth="100"/>

<field name="abstract_embedding"
       type="knn_vector_768"
       indexed="true"
       stored="false"/>
```

### Indexing vectors

Vectors are indexed as JSON arrays. An external embedding service (e.g. a Python microservice using `sentence-transformers`) generates the vector from `dc.description.abstract` and POSTs it to Solr:

```bash
# Example: index a document with an embedding vector
curl "http://localhost:8983/solr/search/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '[{
    "search.uniqueid": "a1b2c3-2",
    "abstract_embedding": [0.021, -0.134, 0.087, ...]
  }]'
```

### KNN query

```bash
# Find the 10 most similar items to a given vector
curl "http://localhost:8983/solr/search/select" \
  -H "Content-Type: application/json" \
  -d '{
    "q": "{!knn f=abstract_embedding topK=10}[0.021, -0.134, 0.087, ...]",
    "fl": "dc.title,score",
    "rows": 10
  }'

# Combine KNN with keyword filter (hybrid search)
curl "http://localhost:8983/solr/search/select" \
  -H "Content-Type: application/json" \
  -d '{
    "q": "{!knn f=abstract_embedding topK=20}[0.021, -0.134, 0.087, ...]",
    "fq": "entityType:Publication AND archived:true",
    "fl": "dc.title,score",
    "rows": 10
  }'
```

<div class="callout callout-info">
<span class="callout-title">Embedding generation is external to Solr</span>
Solr stores and queries vectors but does not generate them. You need an embedding service — a Python script, a microservice, or a DSpace plugin — to convert text to vectors before indexing. The <a href="https://www.sbert.net/" target="_blank">sentence-transformers</a> library is the most common choice for research repository use cases.
</div>

---

## Improved Highlighting

Solr 9 ships with `UnifiedHighlighter` as the recommended highlighter, replacing the older `FastVectorHighlighter` and standard highlighter. The explicit `HighlightComponent` declaration added to the DSpace 2025 `solrconfig.xml` activates this.

### What changed

```xml
<!-- Solr 9 solrconfig.xml — new explicit declaration -->
<searchComponent class="solr.HighlightComponent" name="highlight">
    <highlighting>
        <encoder name="html" class="solr.highlight.HtmlEncoder"/>
    </highlighting>
</searchComponent>
```

The `HtmlEncoder` ensures highlighted snippets have HTML special characters escaped — critical when returning snippets to the frontend to prevent XSS.

### Using highlighting in queries

```bash
# Basic hit highlighting on dc.title and fulltext
curl "http://localhost:8983/solr/search/select?q=music+arts&hl=true&hl.fl=dc.title,fulltext_hl&hl.snippets=3&hl.fragsize=150&wt=json&indent=true"

# Response will include:
# "highlighting": {
#   "<uuid>-2": {
#     "dc.title": ["<em>Music</em> and Performing <em>Arts</em>"],
#     "fulltext_hl": ["...ensemble of <em>music</em> and performing <em>arts</em>..."]
#   }
# }

# Unified highlighter with passage scoring (better context selection)
curl "http://localhost:8983/solr/search/select?q=music&hl=true&hl.method=unified&hl.fl=fulltext_hl&hl.bs.type=SENTENCE&hl.snippets=2&wt=json&indent=true"
```

| Parameter | Description |
|---|---|
| `hl=true` | Enable highlighting |
| `hl.fl` | Fields to highlight (use `_hl` suffix variants for stored text) |
| `hl.method=unified` | Use `UnifiedHighlighter` explicitly |
| `hl.snippets` | Number of highlighted fragments per field |
| `hl.fragsize` | Character length of each fragment |
| `hl.bs.type` | Boundary scanner type: `SENTENCE`, `WORD`, `LINE` |
| `hl.tag.pre` / `hl.tag.post` | Custom highlight tags (default: `<em>` / `</em>`) |

---

## Nested / Child Documents

Solr 9 significantly improved support for nested (child) documents with `[child]` transformer and block join queries. For DSpace this could model the item → bundle → bitstream hierarchy within a single Solr document tree.

### Schema requirement

```xml
<!-- Required in schema.xml for nested documents -->
<field name="_root_" type="string" indexed="true" stored="false" docValues="false"/>
<field name="_nest_path_" type="_nest_path_" indexed="true" stored="true"/>
```

### Example: item with nested bitstreams

```bash
curl "http://localhost:8983/solr/search/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '[{
    "search.uniqueid": "item-uuid-2",
    "dc.title": "My Publication",
    "entityType": "Publication",
    "_childDocuments_": [
      {
        "search.uniqueid": "bitstream-uuid-0",
        "bundle": "ORIGINAL",
        "filename": "paper.pdf",
        "size": 1048576
      },
      {
        "search.uniqueid": "thumbnail-uuid-0",
        "bundle": "THUMBNAIL",
        "filename": "paper.jpg",
        "size": 8192
      }
    ]
  }]'
```

### Block join query

```bash
# Find items that have a bitstream in the ORIGINAL bundle larger than 1MB
q={!parent which="search.resourcetype:2"}(bundle:ORIGINAL AND size:[1048576 TO *])
```

<div class="callout callout-info">
<span class="callout-title">DSpace does not yet use nested documents natively</span>
The DSpace indexer currently creates flat documents. Nested documents would require a custom indexing plugin or a scripted UpdateRequestProcessor that restructures the document tree. This is shown here as a capability for custom DSpace extensions.
</div>

---

## Additional `UpdateRequestProcessor` Use Cases

The scripted processor pattern from the percentage calculator can be extended to other derived field calculations. Here are practical examples for a DSpace CRIS deployment:

### Data quality flag: completeness score

Flag items that are missing required metadata fields for reporting and display:

```javascript
// dc-completeness.js
function processAdd(cmd) {
    var doc = cmd.solrDoc;
    var score = 0;
    var required = [
        "dc.title", "dc.date.issued", "dc.contributor.author",
        "dc.type", "dc.description.abstract"
    ];
    for (var i = 0; i < required.length; i++) {
        if (doc.getFieldValue(required[i])) score++;
    }
    doc.setField("metadata_completeness_score", score);
    doc.setField("metadata_complete", score === required.length ? "true" : "false");
}
```

Schema additions:

```xml
<field name="metadata_completeness_score" type="int" indexed="true" stored="true" docValues="true"/>
<field name="metadata_complete" type="string" indexed="true" stored="true" docValues="true"/>
```

Query — items missing at least one required field:

```
q=archived:true AND metadata_complete:false AND entityType:Publication
```

### Normalised entity type label

Map internal DSpace entity type values to display-friendly labels in the index, avoiding any mapping logic in the frontend:

```javascript
// dc-entity-label.js
function processAdd(cmd) {
    var doc = cmd.solrDoc;
    var typeMap = {
        "Publication": "Research Output",
        "OrgUnit":     "Organisation",
        "Person":      "Researcher",
        "Project":     "Project",
        "Funding":     "Funding"
    };
    var entityType = doc.getFieldValue("dspace.entity.type");
    if (entityType) {
        var label = typeMap[entityType] || entityType;
        doc.setField("entityTypeLabel", label);
    }
}
```

### Derived date decade

Useful for decade-based faceting without application-layer date parsing:

```javascript
// dc-date-decade.js
function processAdd(cmd) {
    var doc = cmd.solrDoc;
    var year = doc.getFieldValue("dc.date.issued.year");
    if (year) {
        var y = parseInt(year, 10);
        if (!isNaN(y)) {
            var decade = Math.floor(y / 10) * 10;
            doc.setField("dc.date.issued.decade", decade);
        }
    }
}
```

Schema:

```xml
<field name="dc.date.issued.decade" type="int" indexed="true" stored="true"
       omitNorms="true" docValues="true"/>
```

Facet query: `&facet=true&facet.field=dc.date.issued.decade&facet.sort=index`

---

## Payload-Scored Fields

Payloads allow storing a per-token numeric weight directly in the inverted index at indexing time. At query time, a `payload` query function boosts the document score based on the stored weight. Useful for custom relevance models that incorporate metadata signals.

### Schema

```xml
<fieldType name="payloads" class="solr.TextField" stored="false" indexed="true">
  <analyzer>
    <tokenizer class="solr.WhitespaceTokenizerFactory"/>
    <filter class="solr.DelimitedPayloadTokenFilterFactory"
            encoder="float"
            delimiter="|"/>
  </analyzer>
</fieldType>

<field name="author_weight" type="payloads" indexed="true" stored="false" multiValued="true"/>
```

### Indexing with payloads

```bash
# Index author names with per-author citation weights as payloads
# Format: term|weight (the | delimiter is configured in the schema)
curl "http://localhost:8983/solr/search/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '[{
    "search.uniqueid": "item-uuid-2",
    "dc.title": "Music Theory and Practice",
    "author_weight": ["Szepe|0.85", "Mueller|0.62", "Weber|0.41"]
  }]'
```

### Payload query

```bash
# Boost results where "Szepe" has a high payload weight
q={!payload_score f=author_weight v=Szepe func=max}
```

---

## Streaming Expressions

Solr 9's Streaming Expressions provide a SQL-like declarative language for aggregations, joins, and data pipelines over Solr collections — without exporting data to an external system.

### Useful for DSpace reporting

```bash
# Count publications per entity type (equivalent to GROUP BY)
curl "http://localhost:8983/solr/search/stream" \
  -H "Content-Type: application/json" \
  -d '{
    "expr": "facet(search, q=\"archived:true\", buckets=\"entityType\", bucketSorts=\"count(*) desc\", bucketSizeLimit=20, count(*))"
  }'

# Items with no abstract (data quality)
curl "http://localhost:8983/solr/search/stream" \
  -H "Content-Type: application/json" \
  -d '{
    "expr": "search(search, q=\"archived:true AND entityType:Publication\", fq=\"-dc.description.abstract:[* TO *]\", fl=\"dc.title,handle\", rows=\"100\", sort=\"dc.date.issued desc\")"
  }'

# Top 20 authors by publication count (using rollup aggregation)
curl "http://localhost:8983/solr/search/stream" \
  -H "Content-Type: application/json" \
  -d '{
    "expr": "rollup(sort(search(search, q=\"archived:true AND entityType:Publication\", fl=\"dc.contributor.author_filter\", rows=\"10000\", sort=\"dc.contributor.author_filter asc\"), by=\"dc.contributor.author_filter\"), over=\"dc.contributor.author_filter\", count(*))"
  }'
```

---

## Full `solrconfig.xml` Addition

The complete block to add to the DSpace 2025 `solrconfig.xml` to enable all scripted processing features documented on this page:

```xml
<!-- ══════════════════════════════════════════════════════════════════════
     UpdateRequestProcessor Chains — DSpace CRIS custom processing
     Add these blocks before the closing </config> tag
     ═══════════════════════════════════════════════════════════════════ -->

<!-- ── Percentage calculator chain ──────────────────────────────────── -->
<updateRequestProcessorChain name="percentageCalculatorChain">
  <processor class="solr.StatelessScriptUpdateProcessorFactory">
    <str name="script">dc-percentage-calculator.js</str>
  </processor>
  <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>

<!-- ── Combined chain: percentage + completeness + decade ────────────── -->
<!-- Use this chain to run all custom scripts in sequence -->
<updateRequestProcessorChain name="dspaceCrisChain">
  <processor class="solr.StatelessScriptUpdateProcessorFactory">
    <!-- Run multiple scripts in order -->
    <arr name="script">
      <str>dc-percentage-calculator.js</str>
      <str>dc-completeness.js</str>
      <str>dc-date-decade.js</str>
    </arr>
  </processor>
  <!-- Log processing errors without failing the whole update -->
  <processor class="solr.LogUpdateProcessorFactory" />
  <processor class="solr.RunUpdateProcessorFactory" />
</updateRequestProcessorChain>

<!-- ── Wire /update and /update/json to the combined chain ───────────── -->
<requestHandler name="/update" class="solr.UpdateRequestHandler">
  <lst name="defaults">
    <str name="update.chain">dspaceCrisChain</str>
  </lst>
</requestHandler>

<requestHandler name="/update/json" class="solr.UpdateRequestHandler">
  <lst name="defaults">
    <str name="stream.contentType">application/json</str>
    <str name="update.chain">dspaceCrisChain</str>
  </lst>
</requestHandler>
```

---

## Reference

| Resource | Link |
|---|---|
| `StatelessScriptUpdateProcessorFactory` | [solr.apache.org/guide/solr/latest/configuration-guide/script-update-processor](https://solr.apache.org/guide/solr/latest/configuration-guide/script-update-processor.html) |
| `UpdateRequestProcessorChain` | [solr.apache.org/guide/solr/latest/configuration-guide/update-request-processors](https://solr.apache.org/guide/solr/latest/configuration-guide/update-request-processors.html) |
| Dense vector / KNN search | [solr.apache.org/guide/solr/latest/query-guide/dense-vector-search](https://solr.apache.org/guide/solr/latest/query-guide/dense-vector-search.html) |
| `UnifiedHighlighter` | [solr.apache.org/guide/solr/latest/query-guide/highlighting](https://solr.apache.org/guide/solr/latest/query-guide/highlighting.html) |
| Nested documents | [solr.apache.org/guide/solr/latest/indexing-guide/nested-documents](https://solr.apache.org/guide/solr/latest/indexing-guide/nested-documents.html) |
| Payload queries | [solr.apache.org/guide/solr/latest/query-guide/other-parsers#payload-score-parser](https://solr.apache.org/guide/solr/latest/query-guide/other-parsers.html#payload-score-parser) |
| Streaming Expressions | [solr.apache.org/guide/solr/latest/query-guide/streaming-expressions](https://solr.apache.org/guide/solr/latest/query-guide/streaming-expressions.html) |
| sentence-transformers (embedding generation) | [sbert.net](https://www.sbert.net/) |

<div class="page-nav">
  <a href="{{ '/ops/solr9-upgrade/' | relative_url }}">← Solr 9 Upgrade</a>
  <a href="{{ '/ops/kubernetes/' | relative_url }}">Kubernetes & OpenShift →</a>
</div>
