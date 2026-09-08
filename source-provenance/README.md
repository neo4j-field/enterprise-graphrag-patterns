# GraphRAG provenance patterns for Neo4j Aura

This demo answers a common GraphRAG lifecycle question:

> After extracting a domain graph from document chunks, how can we trace every entity and relationship back to its sources, update or delete a document safely, and still expose a deduplicated graph that is easy to query at scale?

The example starts with scientific publications split into chunks and extracts the claim “aspirin treats migraine”. Two documents independently support the same claim. `DOC-1` is then updated and deleted, while `DOC-2` and the information it supports must remain.

## Requirements

The target design must provide all of the following:

- Canonical entities such as one `Aspirin` node, even when many chunks mention it.
- Provenance for entity mentions and extracted relationships.
- Independent source metadata such as confidence, text spans, extractor version, and review status.
- Document-scoped update and deletion without corpus-wide scans.
- One canonical domain relationship for normal application queries.

The final query graph is deliberately conventional:

```cypher
MATCH (drug:Drug)-[r:TREATS]->(indication:Indication)
RETURN drug, r, indication;
```

Applications do not need to traverse `Fact` or `Evidence` nodes. Those nodes belong to the provenance and ingestion model. The single native `TREATS` relationship is a materialized projection of the currently supported facts.

## Common entity provenance

Every model uses the same document, chunk, and entity structure:

```text
(Chunk)-[:PART_OF]->(Document)
(Chunk)-[:MENTIONS {start, end, confidence}]->(Entity)
```

`MENTIONS` is preferable to saying that a canonical entity was `EXTRACTED_FROM` a chunk. The chunk contains an occurrence that was resolved to the shared entity; the entity itself may already exist because another document mentioned it.

`start` is the inclusive character offset and `end` is the exclusive character offset inside `Chunk.text`. For `Aspirin treats migraine.`, `{start: 0, end: 7}` identifies `Aspirin`. Production models may use clearer names such as `startOffset` and `endOffset`.

If individual mentions require their own identity, correction, or review lifecycle, reify them as `Mention` nodes. A relationship is sufficient for the simpler case.

## Four provenance models

### 1. Source IDs in one canonical relationship

```text
(Drug)-[:TREATS {sourceChunkIds: [...]}]->(Indication)
```

This produces the desired query graph with no duplicate `TREATS` relationships, but the provenance is an array on a shared relationship. Removing a document means finding and editing array values. Per-source confidence and extraction metadata are awkward to represent, and popular facts can become write hotspots.

This model is included as the tempting baseline, not as the recommendation.

### 2. Source relationships plus a canonical relationship

```text
(Drug)-[:TREATS_SOURCE {chunkId, confidence, ...}]->(Indication)
(Drug)-[:TREATS {supportCount}]->(Indication)
```

Each extraction occurrence is independently removable and the canonical `TREATS` relationship remains easy to query. However, provenance must be implemented again for every domain relationship type, and the graph contains two representations of the same predicate with different meanings.

There is also no structural graph path from a chunk to a `TREATS_SOURCE` relationship: relationships cannot be endpoints of other relationships. Document deletion must therefore find source relationships through stored IDs and a relationship-property index, with separate lifecycle logic for every predicate. This is a fundamental reason to prefer a shared Fact-based provenance model.

### 3. Canonical Fact nodes plus a native query projection

```text
(Chunk)-[:SUPPORTS {confidence, extractorVersion}]->(Fact)
(Fact)-[:SUBJECT]->(Drug)
(Fact)-[:OBJECT]->(Indication)
(Drug)-[:TREATS {factId, supportCount}]->(Indication)
```

One `Fact` represents one normalized claim. Many chunks can support it without duplicating the logical fact. The native `TREATS` relationship is created while the fact has at least one active support and removed when its last support disappears.

This is the recommended default. `Fact` is the provenance source of truth; `TREATS` is the query projection.

### 4. Fact and Evidence nodes plus a native query projection

```text
(Chunk)-[:HAS_EVIDENCE]->(Evidence)-[:SUPPORTS]->(Fact)
(Fact)-[:SUBJECT]->(Drug)
(Fact)-[:OBJECT]->(Indication)
(Drug)-[:TREATS {factId, supportCount}]->(Indication)
```

An `Evidence` node represents one extraction occurrence and can carry quotations, spans, confidence, model version, review status, and correction history. Use it when an occurrence needs a stable ID or independent lifecycle. Otherwise, metadata on `Chunk-[:SUPPORTS]->Fact` is smaller and simpler.

## What belongs on Fact, SUPPORTS, and Evidence

The placement rule is simple:

- Put information on `Fact` when it describes the normalized claim and is identical for every source.
- Put information on `SUPPORTS` when it describes one chunk’s support for that claim.
- Create an `Evidence` node when that source occurrence must be addressed, reviewed, corrected, versioned, or linked independently.

### Fact properties

A `Fact` stores the source-independent meaning and identity of a claim. Typical properties include:

| Property | Purpose |
|---|---|
| `factId` | Deterministic identifier derived from the normalized claim |
| `predicate` | Normalized predicate such as `TREATS` |
| `polarity` | Positive, negative, or another normalized assertion state |
| `modality` | Asserted, possible, hypothetical, conditional, and so on |
| `qualifiers` | Normalized dose, route, population, time, or experimental context when these participate in claim identity |
| `normalizationVersion` | Version of the rules or ontology used to construct the canonical claim |

Do not put `chunkId`, quotation, extraction confidence, or extractor version on `Fact`: different sources of the same Fact can have different values. Mutable aggregates such as support count or maximum confidence should normally be treated as derived caches rather than Fact identity.

### SUPPORTS relationship properties

In model 3, each `Chunk-[:SUPPORTS]->Fact` relationship represents one source occurrence. It can store compact source-specific information:

| Property | Purpose |
|---|---|
| `supportId` | Stable ID for idempotent ingestion when the same chunk can be processed more than once |
| `confidence` | Extractor confidence for this occurrence—not confidence in the Fact globally |
| `extractorVersion` | Model or pipeline version that produced the extraction |
| `extractionRunId` | Identifier of the ingestion or extraction run |
| `extractedAt` | Extraction timestamp when operational audit is needed |
| `quotation` | Optional short supporting text when an Evidence node would be excessive |
| `subjectStart`, `subjectEnd`, `objectStart`, `objectEnd` | Optional offsets within the chunk |
| `status` | Optional lightweight state such as active, rejected, or superseded |

The source document does not need to be repeated on the relationship because it is reached through `Chunk-[:PART_OF]->Document`. Avoid copying large chunk text onto every support.

If one chunk can produce the same Fact in several extraction runs, either give each `SUPPORTS` relationship a unique `supportId`, replace the previous run deterministically, or use explicit Evidence nodes. The choice depends on whether previous runs must remain queryable.

### Evidence node properties

An `Evidence` node promotes a support occurrence into a first-class record. Typical properties include:

| Property | Purpose |
|---|---|
| `evidenceId` | Stable identifier for this extraction occurrence |
| `quotation` | Exact text supporting the interpretation |
| `page`, `section`, and character offsets | Precise location inside the source |
| `confidence` | Confidence assigned by this extraction run |
| `extractorVersion` and `extractionRunId` | Extraction lineage |
| `reviewStatus` | Unreviewed, accepted, rejected, corrected, or superseded |
| `reviewedBy`, `reviewedAt`, `reviewComment` | Human-review audit information |
| `createdAt`, `validFrom`, `validTo` | Lifecycle or temporal history when required |

With explicit Evidence, rich occurrence metadata belongs on the node:

```text
(Chunk)-[:HAS_EVIDENCE]->(Evidence)-[:SUPPORTS]->(Fact)
```

The `Evidence-[:SUPPORTS]->Fact` relationship can remain property-free when an Evidence item supports exactly one Fact. If Evidence can be interpreted differently against several Facts, the relationship may carry only association-specific values such as `stance`, `entailmentScore`, or `mappingVersion`.

For contradictory scientific statements, either use association types such as `SUPPORTS` and `CONTRADICTS`, or store a controlled `stance` value. Keep the canonical claim’s own negation or polarity on `Fact` when it changes the claim identity; do not confuse claim polarity with whether a source supports that claim.

## Recommendation

For a corpus of millions of documents, start with model 3:

- `Document`, `Chunk`, and `MENTIONS` provide entity provenance.
- `Fact` deduplicates normalized claims.
- `SUPPORTS` represents source-specific provenance.
- Native domain relationships such as `TREATS` form the application-facing graph.
- Explicit `Evidence` nodes are added only where their additional lifecycle justifies their storage cost.

The native relationship is derived data. It should be reproducible from active facts and evidence, and ingestion or deletion must update it in the same transaction or through a reliable projection pipeline.

## Query first, cite when needed

Normal application queries use only the native domain graph:

```cypher
MATCH (drug:Drug)-[:TREATS]->(indication:Indication)
RETURN drug, indication;
```

When a GraphRAG answer needs citations, the `factId` on the materialized relationship joins that query result to provenance:

```cypher
MATCH (drug:Drug)-[r:TREATS]->(indication:Indication)
MATCH (f:Fact {factId: r.factId})<-[s:SUPPORTS]-
      (c:Chunk)-[:PART_OF]->(d:Document)
RETURN drug.name AS subject, type(r) AS predicate,
       indication.name AS object, d.title AS source,
       c.text AS excerpt, s.confidence AS confidence;
```

The Aura demo includes equivalent citation queries for both the compact `SUPPORTS` model and the explicit `Evidence` model. Running them after document deletion also verifies that the native projection and its provenance remain consistent.

## Fact identity

The demo uses a readable ID such as:

```text
DRUG-ASPIRIN|TREATS|IND-MIGRAINE
```

Production IDs should be generated from a canonical representation and normally hashed outside Neo4j. Include every field that changes the meaning of a scientific claim, potentially including negation, dose, route, population, temporality, modality, and experimental context. Claims with materially different qualifiers must not collapse into the same `Fact`.

Canonical entity resolution is a separate concern. Removing a document must remove its mentions and support without deleting an entity that is still mentioned or used by another supported fact.

## Update and deletion lifecycle

For one document:

1. Start from its indexed `documentId`.
2. Traverse to its chunks and capture only the fact and entity IDs touched by those chunks.
3. Delete the document-owned chunks, mentions, supports, and evidence.
4. Recheck only the captured facts.
5. For facts with no remaining support, delete the `Fact` and its materialized native relationship.
6. Remove only touched entities that have no remaining mentions or supported facts.
7. For an update, ingest the replacement document version and materialize its supported domain relationships.

Do not scan every `Fact`, `Entity`, or relationship after each deletion. Large maintenance jobs should also be executed in bounded transactions.

If audit history is required, model document versions or extraction runs explicitly and mark old evidence inactive instead of physically deleting it.

## Scale considerations

“Four million documents” is not enough to estimate graph size. Capacity depends on chunks per document, entity mentions per chunk, extracted claims per chunk, repeated extraction runs, and how often claims are shared.

Other practical considerations:

- Generate deterministic document, chunk, entity, fact, and evidence IDs before ingestion.
- Back those IDs with uniqueness constraints.
- Use idempotent `MERGE`-based ingestion so retrying a batch does not duplicate sources, Facts, Evidence, or native relationships.
- Avoid updating `supportCount` for every event if highly popular facts create contention; maintain the projection in batches or asynchronously when appropriate.
- Rebuild or reconcile materialized domain relationships periodically because they are derived data.
- Benchmark the customer’s real read and write workloads before deciding whether projection metadata such as `supportCount` is worth maintaining.

Relationship-property uniqueness constraints used by the demo are available from Neo4j 5.7 onward.

## Optional extension: contradictory publications

A useful follow-on experiment is a third document stating that aspirin is not effective for migraine. It should create a separate normalized Fact with negative polarity rather than modify the positive Fact. Both Facts can retain their own sources and materialized domain semantics. This extension is intentionally outside the main walkthrough so the provenance and deletion story stays short.

## Import and run

Import `graphrag-provenance-aura-saved-queries.csv` into the saved queries area of the Neo4j Aura console. It creates one top-level folder containing four model folders.

Run one model folder in step order. Each model is self-contained and starts by deleting all data in the current database, so use only a disposable demonstration database.

The saved queries favor visual explanation over bulk-ingestion performance. Graph-display steps return nodes and relationships so the effect of adding, updating, and deleting sources can be inspected directly in Aura.

## Comparison

| Model | Deduplicated query graph | Source lifecycle | Main drawback |
|---|---:|---:|---|
| Relationship with source array | Yes | Weak | Arrays are difficult to query and update per source |
| Source + canonical relationships | Yes | Good | Predicate-specific schema and no structural path from chunks to source relationships |
| Fact + native projection | Yes | Good | Projection consistency must be maintained |
| Fact + Evidence + native projection | Yes | Richest | Highest storage and operational cost |
