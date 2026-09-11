# Enterprise GraphRAG patterns with Neo4j

Practical Neo4j reference patterns and runnable demos for recurring lifecycle and modeling questions in enterprise GraphRAG projects.

The repository is intended for Neo4j field teams, partners, and customers designing graphs extracted from documents and other enterprise sources. Each pattern starts with a concrete customer question, compares a small number of graph models, recommends a default, and includes Cypher that can be explored visually.

## Available patterns

| Pattern | Customer question | Recommended default |
|---|---|---|
| [Source provenance](source-provenance/) | How can extracted entities and relationships be traced to their document chunks, updated or deleted safely, and exposed as a deduplicated query graph? | Canonical `Fact` nodes with source-specific `SUPPORTS` and a native relationship projection |
| [Entity resolution](entity-resolution/) | What happens to mentions, Facts, relationships, and provenance when duplicate entities are merged, and how can that decision be corrected? | Explicit `EntityMention` nodes resolved to canonical entities, with decision history added when required |

## Shared design principles

- Keep the application-facing graph conventional. Normal queries use canonical entities and native domain relationships rather than ingestion or governance structures.
- Treat native domain relationships as materialized projections when canonical Facts or resolution records are the source of truth.
- Preserve source observations separately when they need provenance, correction, review, or independent lifecycle.
- Use deterministic IDs, uniqueness constraints, and idempotent `MERGE`-based ingestion.
- Capture affected IDs before destructive lifecycle changes and clean only those candidates; do not scan the complete graph after each update, deletion, merge, or split.
- Explain alternatives primarily by **when to use them** and **what capability they add**.
- Use readable identifiers in demos, while documenting stronger canonical identity requirements for production.

## Repository structure

```text
source-provenance/
  README.md
  graphrag-provenance-aura-saved-queries.csv

entity-resolution/
  README.md
  graphrag-entity-resolution-aura-saved-queries.csv
```

Each pattern README explains its customer problem, graph models, recommendation, lifecycle, and pattern-specific execution sequence. The CSV file imports the complete walkthrough into Neo4j Aura saved queries.

## Requirements

- A disposable Neo4j database. Neo4j Aura is the primary walkthrough environment, but the Cypher can also be run against a compatible local Neo4j database.
- Access to the saved queries area of the Neo4j Aura console when importing a CSV.
- Neo4j features used by a particular pattern, such as relationship-property uniqueness constraints, as documented in that pattern's README.

The suites were most recently verified with Neo4j Enterprise 2026.08.0 using the Cypher 25 default language.

## Import and run

1. Choose a pattern and read its README before executing the Cypher.
2. Import that pattern's `*-aura-saved-queries.csv` file into the saved queries area of the Neo4j Aura console.
3. Open the imported top-level folder and select one model folder.
4. Run the model's queries in step order. Model folders are self-contained; do not mix steps from different models.
5. Use the graph-returning steps to inspect lifecycle changes and the citation steps to trace native relationships back to source documents.

For a local Neo4j database, run the same Cypher queries in the saved folder order using Neo4j Browser, Query, cypher-shell, or another Cypher client.

## Safety warning

Every model begins with:

```cypher
MATCH (n) DETACH DELETE n;
```

That query deletes every node and relationship in the selected database. Use these walkthroughs only with a disposable database, verify the target database before starting, and never run a reset step against data you need.

Constraints created by one model may remain after its data is reset. This is harmless when running the supplied demos, but remove demo constraints separately if the database must be returned to its original empty schema.

## How to read the demos

The Cypher favors clarity and visible lifecycle transitions over bulk-ingestion performance. Short comments explain important operations, while the READMEs contain the design rationale and production considerations.

The examples deliberately show before-and-after states such as document deletion, entity merge, and entity split. These transitions explain why richer provenance and resolution structures exist more clearly than a final graph alone.

## License

Copyright 2026 Neo4j Sweden AB. Licensed under the [Apache License 2.0](LICENSE).
