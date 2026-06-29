# HugeGraph Source Connector — Technical Design

> Based on [hugegraph/hugegraph#151](https://github.com/hugegraph/hugegraph/issues/151).  
> Target: SeaTunnel connector-v2, HugeGraph 1.8.0+ with `hugegraph-client`.

---

## 1. Architecture Overview

SeaTunnel connector-v2 uses a **split-enumerator-reader** model. The Source is responsible for discovering what data exists (splits), distributing work across parallelism (enumerator), and reading data into `SeaTunnelRow` (reader).

```
                    ┌──────────────────────┐
                    │  HugeGraphSource      │  Entry point, creates enumerator
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  HugeGraphSourceEnum  │  Discovers label splits
                    │  (enumerator)         │  Assigns to readers
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌──────▼──────┐ ┌───────▼────────┐
    │ HugeGraphSource│ │HugeGraphSrc │ │HugeGraphSource  │
    │ Split (vertex) │ │Split (edge) │ │Reader           │
    └────────────────┘ └─────────────┘ └───────┬─────────┘
                                               │ reads by label + page
                                        ┌──────▼──────┐
                                        │ HugeGraph   │
                                        │ Client      │
                                        └─────────────┘
```

### Design decision: single-reader model for v1

SeaTunnel supports both single-reader and multi-reader parallelism. For v1 we use **single-reader** because:

- HugeGraph's REST API is the data plane — parallel readers hit the same server, adding contention without throughput gain in standalone mode.
- Full label scans are I/O-bound on the server side, not CPU-bound on the connector side.
- Simpler error handling: no reader coordination, no checkpoint synchronization.
- Can be upgraded to multi-reader later by splitting on vertex/edge ID ranges.

---

## 2. Class Structure

### 2.1 HugeGraphSource

```java
@AutoService(Factory.class)
public class HugeGraphSourceFactory
    implements TableSourceFactory, SupportParallelism {

    @Override
    public String factoryIdentifier() { return "HugeGraph"; }

    @Override
    public OptionRule optionRule() {
        return OptionRule.builder()
            .required(HugeGraphSourceConfig.URL)
            .required(HugeGraphSourceConfig.GRAPH)
            .optional(HugeGraphSourceConfig.LABEL)
            .optional(HugeGraphSourceConfig.LABEL_TYPE)
            .optional(HugeGraphSourceConfig.FIELDS)
            .optional(HugeGraphSourceConfig.PAGE_SIZE)
            .conditional(
                HugeGraphSourceConfig.LABEL_TYPE, "edge",
                HugeGraphSourceConfig.EDGE_FIELDS
            )
            .build();
    }

    @Override
    public SeaTunnelDataType<SeaTunnelRow> produceDataType(
            ReadonlyConfig config) {
        // Schema is user-declared (not auto-discovered for v1)
        // Build RowType from config.get(FIELDS) + reserved fields
        return buildRowType(config);
    }
}
```

### 2.2 HugeGraphSourceConfig

Centralize all option definitions. Prevents option name drift between Factory, Reader, and docs.

```java
public class HugeGraphSourceConfig {
    public static final Option<String> URL =
        Options.key("url").stringType()
               .noDefaultValue()
               .withDescription("HugeGraph Server URL");

    public static final Option<String> GRAPH =
        Options.key("graph").stringType()
               .defaultValue("hugegraph")
               .withDescription("Graph name / graph space");

    public static final Option<String> LABEL =
        Options.key("label").stringType()
               .noDefaultValue()
               .withDescription("Vertex or edge label to read");

    public static final Option<LabelType> LABEL_TYPE =
        Options.key("label_type").enumType(LabelType.class)
               .defaultValue(LabelType.VERTEX)
               .withDescription("vertex or edge");

    public static final Option<List<String>> FIELDS =
        Options.key("fields").listType()
               .noDefaultValue()
               .withDescription("Properties to read. " +
                   "If absent, all properties returned.");

    public static final Option<Integer> PAGE_SIZE =
        Options.key("page_size").intType()
               .defaultValue(1000)
               .withDescription("Batch size for label scan. " +
                   "Larger = fewer network round-trips, more memory.");
}
```

### 2.3 HugeGraphSourceSplit

A split represents a unit of work: one (label, labelType) pair.

```java
public class HugeGraphSourceSplit implements SourceSplit {
    private final String splitId;       // "vertex:person" or "edge:knows"
    private final String label;          // "person"
    private final LabelType labelType;   // VERTEX or EDGE
    private final List<String> fields;   // selected properties
    private final int pageSize;
}
```

### 2.4 HugeGraphSourceEnumerator

Discovers splits from config. For v1: **exactly one split per configured label**.

```java
public class HugeGraphSourceEnumerator
    implements Enumerator<HugeGraphSourceSplit, HugeGraphSourceState> {

    @Override
    public void run() {
        // v1: single split per label
        HugeGraphSourceSplit split = new HugeGraphSourceSplit(
            "vertex:" + config.getLabel(),
            config.getLabel(),
            config.getLabelType(),
            config.getFields(),
            config.getPageSize()
        );
        // Assign to reader 0 (single-reader model)
        context.assignSplit(0, List.of(split));
    }
}
```

**Design decision: no snapshot/checkpoint state in v1.** Full label scans are stateless — if the job restarts, it re-scans from the beginning. This is acceptable for batch mode and avoids the complexity of tracking scan cursors.

### 2.5 HugeGraphSourceReader

The core of the Source connector. Reads vertices or edges by label with pagination.

```java
public class HugeGraphSourceReader
    implements SourceReader<SeaTunnelRow, HugeGraphSourceSplit> {

    private final HugeClient client;
    private final SingleThreadFetcher fetcher;

    @Override
    public void pollNext(Collector<SeaTunnelRow> output) {
        for (HugeGraphSourceSplit split : pendingSplits) {
            readSplit(split, output);
        }
    }

    private void readSplit(
            HugeGraphSourceSplit split,
            Collector<SeaTunnelRow> output) {

        if (split.labelType() == LabelType.VERTEX) {
            readVertices(split, output);
        } else {
            readEdges(split, output);
        }
    }
}
```

---

## 3. Data Mapping: HugeGraph → SeaTunnelRow

This is the core design challenge. SeaTunnelRow is a flat tuple; HugeGraph data is a property graph with rich semantics.

### 3.1 Vertex output schema (canonical)

| Column index | Column name | HugeGraph source | Type |
|---|---|---|---|
| 0 | `~id` | `vertex.id()` | STRING |
| 1 | `~label` | `vertex.label()` | STRING |
| 2..N | user properties | `vertex.property(name)` | determined by property type |

```
SeaTunnelRow: ["person:1", "person", "Alice", 30, ...]
               ──┬──      ──┬──    ───┬──  ─┬─
               ~id        ~label   name   age
```

### 3.2 Edge output schema (canonical)

| Column index | Column name | HugeGraph source | Type |
|---|---|---|---|
| 0 | `~id` | `edge.id()` | STRING |
| 1 | `~label` | `edge.label()` | STRING |
| 2 | `~source_id` | `edge.sourceId()` | STRING |
| 3 | `~source_label` | `edge.sourceLabel()` | STRING |
| 4 | `~target_id` | `edge.targetId()` | STRING |
| 5 | `~target_label` | `edge.targetLabel()` | STRING |
| 6..N | user properties | `edge.property(name)` | as vertex |

**Design decision: prefix reserved fields with `~`.** SeaTunnel uses backtick-quoting for special column names. The `~` prefix:
- Avoids collision with user property names (HugeGraph property keys cannot start with `~`).
- Makes reserved fields visually distinct in config and debug output.
- Follows HugeGraph's own convention (`~task_result`, `~hidden` prefix).

### 3.3 Type mapping

| HugeGraph type | SeaTunnel type | Notes |
|---|---|---|
| `STRING`, `TEXT` | `STRING` | Direct |
| `INT`, `LONG` | `BIGINT` (INT64) | `INT` widened to avoid overflow |
| `FLOAT`, `DOUBLE` | `DOUBLE` (FLOAT64) | Same rationale |
| `BOOLEAN` | `BOOLEAN` | Direct |
| `DATE` | `TIMESTAMP` | Convert to `java.time.Instant` |
| `UUID` | `STRING` | ToString |
| `BLOB` | `BYTES` | Pass through |
| `LIST<T>` | `ARRAY<T>` | Element-by-element conversion |

**Design decision: widen integers and floats.** HugeGraph schemas can change. Widening at the connector boundary prevents job failures when a property changes from `INT` to `LONG` between pipeline runs. Users can downcast in transforms if needed.

### 3.4 Null handling

```
Missing property  → SeaTunnelRow column = null
Vertex deleted    → skip, log at DEBUG
Property is null  → SeaTunnelRow column = null  (distinguishable from missing)
```

---

## 4. Read Path: Full Label Scan

```
HugeGraphSourceReader.readVertices(split)
        │
        ▼
client.listVertices(label, page, pageSize)   // REST: GET /graphs/{g}/vertices?label=X&page=N
        │
        ▼
for each vertex:
  buildVertexRow(vertex, fields)
        │
        ▼
output.collect(row)
        │
        ▼
page++  →  loop until empty page
```

### Pagination

```java
private void readVertices(HugeGraphSourceSplit split,
                          Collector<SeaTunnelRow> output) {
    String label = split.label();
    int pageSize = split.pageSize();
    String page = null;  // null = first page

    while (true) {
        List<Vertex> batch = client.listVertices(
            label, page, pageSize);
        if (batch.isEmpty()) break;

        for (Vertex v : batch) {
            output.collect(buildVertexRow(v, split.fields()));
        }

        // HugeGraph REST API returns next page marker in response
        page = getNextPage(batch);
        if (page == null) break;
    }
}
```

---

## 5. Code Reuse with Existing Sink

PR #10002 already has `HugeGraphSink` with:

```
HugeGraphSinkConfig   → shared options (URL, GRAPH, auth)
HugeGraphClient       → connection management
```

We extract shared code into a **common package**:

```
connector-hugegraph/
├── HugeGraphSinkFactory.java     (existing, updated)
├── HugeGraphSinkWriter.java      (existing)
├── HugeGraphSinkConfig.java      (existing)
├── HugeGraphSourceFactory.java   (NEW)
├── HugeGraphSourceReader.java    (NEW)
├── HugeGraphSourceSplit.java     (NEW)
├── HugeGraphSourceEnumerator.java(NEW)
├── common/
│   ├── HugeGraphClient.java      (extracted, shared)
│   ├── HugeGraphOptions.java     (shared URL/GRAPH/auth)
│   ├── HugeGraphTypeMapper.java  (NEW: type conversion)
│   └── HugeGraphRowConverter.java(NEW: vertex/edge → SeaTunnelRow)
```

**Design decision: shared module within the same connector artifact.** A separate `connector-hugegraph-common` would add Maven complexity for minimal benefit. Co-locating in the same module with a `common/` package keeps the build simple and the reuse obvious.

---

## 6. Option Validation (OptionRule)

```
url:       REQUIRED, must be valid URI
graph:     OPTIONAL, defaults to "hugegraph"
label:     REQUIRED, non-empty string
label_type:OPTIONAL, "vertex" (default) or "edge"
fields:    OPTIONAL, list of property names. If absent → read all.
page_size: OPTIONAL, 100–10000, default 1000
```

**Conditional rule:**

```
IF label_type == "edge" THEN ~source_id, ~source_label, ~target_id, ~target_label are auto-included
```

Users do NOT need to list `~id` or `~label` in `fields` — these are always present.

**Validation errors (user-friendly):**

```
"label 'person' not found in graph 'hugegraph'"
  → Check schema with GET /graphs/hugegraph/schema/vertexlabels

"field 'email' not in vertex label 'person'. Available: [name, age, city]"
  → Check property list or omit 'fields' to include all

"page_size must be between 100 and 10000, got 5"
  → Small page sizes cause excessive network round-trips
```

---

## 7. Key Design Decisions (with rationale)

| Decision | Rationale |
|---|---|
| Single-reader for v1 | HugeGraph REST is I/O-bound; parallel readers add contention without gain |
| Full label scan only | Incremental/CDC requires server-side change-log, which doesn't exist yet |
| User-declared schema | Schema auto-discovery requires extra API calls and introduces ambiguity when schemas differ across pages |
| `~` prefix for reserved fields | Prevents collision with user properties; follows HugeGraph conventions |
| Widen numeric types | Schema evolution safety; users can narrow in transforms |
| Shared module, not separate artifact | Avoids Maven complexity; Sink code is small enough to co-locate |
| No checkpoint state in v1 | Batch mode is stateless; re-scan on restart is acceptable and simple |
| Config validation at connection time | Fail-fast: validate label, fields, auth before any data is read |

---

## 8. Error Handling Strategy

```
Connection failure       → SeaTunnel retry (3 attempts, exponential backoff)
Auth failure (401/403)   → Fail immediately, surface credentials error
Label not found          → Fail at config validation (before reading)
Property not in label    → Fail at config validation
Empty label              → Complete successfully (0 rows), not an error
Mid-scan network error   → Retry current page, then fail if exhausted
Type conversion failure  → Skip row, log WARN with vertex ID and field name
```

**Why skip on type conversion failure rather than fail:** A schema migration that adds a new property type shouldn't break existing pipelines reading old data. Skipping with a log is defensive; users can tighten behavior in a transform.

---

## 9. E2E Validation Scenarios

```
1. HugeGraph → File
   Read vertices by label → write CSV → verify row count and field values

2. HugeGraph → JDBC
   Read edges by label → write PostgreSQL → verify JOIN across tables

3. HugeGraph A → HugeGraph B (graph clone)
   Source: read vertices + edges from graph A
   Transform: remap label names if needed
   Sink: write to graph B
   Verify: vertex count(A) == vertex count(B), edge count(A) == edge count(B)

4. Empty label
   Read label with 0 vertices → verify pipeline completes successfully

5. Large label (100K+ vertices)
   Read with page_size=1000 → verify all rows retrieved, no duplicates, no gaps
```

---

## 10. Open Design Questions

These are intentional deferrals, with rationale:

| Question | V1 decision | Rationale |
|---|---|---|
| Condition-based reads? | No — full label scan only | Requires Gremlin query injection, which opens security and complexity |
| Schema auto-discovery? | No — user declares fields | Schema can differ across vertices within same label in HugeGraph |
| Incremental/CDC reads? | No — follow-up | Requires server-side change feed, not yet in HugeGraph |
| Multi-reader parallelism? | No — follow-up | Requires ID-range splitting, needs server-side support |
| Gremlin query source? | No — separate sub-issue | Different security model, different pagination semantics |

---

## References

- [hugegraph/hugegraph#151](https://github.com/hugegraph/hugegraph/issues/151) — Task specification
- [apache/seatunnel#10001](https://github.com/apache/seatunnel/issues/10001) — Sink connector issue
- [apache/seatunnel#10002](https://github.com/apache/seatunnel/pull/10002) — Sink connector PR
- [SeaTunnel Source API](https://seatunnel.apache.org/docs/connector-v2/source/) — Connector v2 documentation
- [HugeGraph Client](https://github.com/apache/hugegraph-toolchain/tree/master/hugegraph-client) — Java client library
