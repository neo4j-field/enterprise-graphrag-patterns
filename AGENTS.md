# Project context

## Purpose

This repository contains small, customer-facing experiments about GraphRAG with Neo4j. Each experiment should answer a concrete customer question with a short explanation and a runnable demo. More experiments will be added over time.

## Current experiment: source provenance

The first experiment is in `source-provenance/`. It addresses provenance and lifecycle management when an LLM extracts a domain graph from scientific-publication chunks:

```text
(Chunk)-[:PART_OF]->(Document)
(Chunk)-[:MENTIONS]->(Entity)
```

The example domain claim is:

```text
(:Drug)-[:TREATS]->(:Indication)
```

The customer needs to trace extracted entities and relationships to their chunks, update or delete documents safely, support a corpus of roughly four million documents, and expose a deduplicated graph for normal queries.

## Modeling decisions

- The application-facing graph uses canonical native relationships such as `(:Drug)-[:TREATS]->(:Indication)`. Normal queries must not require traversing `Fact` nodes.
- `Fact` and `Evidence` are provenance and ingestion structures. A native domain relationship is a materialized projection of a supported Fact.
- Model 3—canonical `Fact` nodes, `Chunk-[:SUPPORTS]->Fact`, and native domain relationships—is the recommended default.
- Add explicit `Evidence` nodes only when an extraction occurrence needs its own identity, review state, correction history, or lifecycle.
- Use `Chunk-[:MENTIONS]->Entity` for entity provenance. `start` is an inclusive character offset and `end` is an exclusive offset in `Chunk.text`.
- Keep source-specific values such as confidence and extractor version on `SUPPORTS` or `Evidence`, not on the canonical Fact.
- Fact identity includes every normalized qualifier that changes claim meaning. The readable demo ID is simplified; production IDs should come from a canonical representation.
- Document deletion must capture touched Fact and Entity IDs before deleting chunks, then clean up only those candidates. Do not scan the complete graph after each deletion.
- Ingestion should be idempotent, and `supportCount` is derived projection metadata rather than Fact identity.

## Demo structure

`source-provenance/README.md` must be sufficient to understand the customer problem, alternatives, recommendation, and lifecycle without running Neo4j.

`source-provenance/graphrag-provenance-aura-saved-queries.csv` is imported into Neo4j Aura saved queries. It contains four progressively richer models:

1. Source chunk IDs stored as an array on one canonical domain relationship.
2. Source-specific native relationships plus a canonical native relationship.
3. Canonical Fact nodes plus a native query projection.
4. Fact and Evidence nodes plus a native query projection.

Keep explanations primarily in the README and Cypher comments minimal. Include graph-returning queries for visual understanding and citation queries that connect a native domain relationship back to its supporting chunks and documents.

## Verification

- Validate the saved-query CSV structure after editing it.
- When requested, test every saved query in folder order against a disposable local Neo4j database.
- Every model begins with `MATCH (n) DETACH DELETE n`; explicitly confirm before running it against a database containing data.
- The suite was last run successfully on Neo4j Enterprise 2026.07.1 with no errors or warnings.

## Repository

- GitHub: `https://github.com/neo4j-field/graphrag-experiments`
- Default branch: `main`
- Do not commit `.DS_Store`, credentials, connection files, or secrets.
