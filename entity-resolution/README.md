# Entity resolution and post-processing patterns for Neo4j GraphRAG

[← All enterprise GraphRAG patterns](../README.md)

This pattern answers a common enterprise GraphRAG question:

> What happens to mentions, facts, relationships, and provenance when two entities are resolved as the same real-world thing—and how can that decision later be corrected?

The example begins with two scientific-publication chunks:

```text
DOC-1: "Paracetamol treats fever."
DOC-2: "Acetaminophen treats fever."
```

The extraction pipeline initially creates two `Drug` entities and two supported `Fact` nodes. The demo then decides that `Paracetamol` and `Acetaminophen` identify the same drug. After resolution, the application graph exposes one canonical relationship:

```text
(:Drug {name: "Acetaminophen"})-[:TREATS]->(:Indication {name: "Fever"})
```

Both publications remain available as provenance. The walkthrough then reverses the resolution to show what information each model retained.

The example mapping is deliberately supplied by the demo. A production system should obtain equivalence evidence from trusted identifiers, reference data, deterministic rules, statistical models, human review, or a combination of those sources.

## Requirements

An enterprise design commonly needs to:

- Preserve the original surface form, offsets, source chunk, extractor confidence, and extractor version.
- Expose canonical entities and native domain relationships for normal application queries.
- Represent unresolved and ambiguous mentions without forcing an early merge.
- Apply entity merges incrementally without scanning the complete corpus.
- Recompute affected Fact identities and deduplicate native relationships after an entity resolution changes.
- Explain important decisions and retain rejected or superseded alternatives when governance requires it.
- Reverse an incorrect merge without re-extracting every document.
- Reconcile document mentions with independently managed structured records.

The central separation is:

```text
source observations -> identity resolution -> canonical entities -> query projection
```

Entity resolution determines which observations refer to the same thing. Fact resolution separately determines whether claims become the same normalized Fact after their subjects or objects are resolved.

## Shared query and provenance graph

Every model produces the same application-facing query shape:

```cypher
MATCH (drug:Drug)-[r:TREATS]->(indication:Indication)
RETURN drug, r, indication;
```

The native `TREATS` relationship is derived from supported canonical Facts:

```text
(Chunk)-[:SUPPORTS]->(Fact)
(Fact)-[:SUBJECT]->(Drug)
(Fact)-[:OBJECT]->(Indication)
(Drug)-[:TREATS]->(Indication)
```

The compact demo also stores `subjectMentionId` and `objectMentionId` on each `SUPPORTS` occurrence. Those role bindings identify which extracted observations produced a claim, which is essential when a later split must route support back to different canonical entities. When an extraction occurrence needs its own identity or richer role structure, use an explicit Evidence or extraction node instead:

```text
                                +--[:SUBJECT_MENTION]->(EntityMention)
(Chunk)-[:HAS_EVIDENCE]->(Evidence)+--[:OBJECT_MENTION]->(EntityMention)
                                +--[:SUPPORTS]->(Fact)
```

When two Drug entities are consolidated, Facts whose identity includes the Drug ID may also become duplicates. Their support must be consolidated before the native query projection is refreshed. Entity resolution therefore cannot safely stop at changing one node or relationship.

The demo keeps readable IDs such as `DRUG-PARACETAMOL|TREATS|IND-FEVER`. Production IDs should be generated from a canonical representation that includes all meaning-changing qualifiers, then normally hashed outside Neo4j.

## Four identity models

The models are organized by when to use them and what capability they add. They are not a maturity ladder that every project must climb. In particular, identity clusters and decision history solve different problems and can be used independently or together.

### 1. Physical canonical merge

```text
(Chunk)-[:MENTIONS]->(Entity)
```

Two duplicate canonical nodes are physically consolidated. Mentions and Fact support are moved to a selected survivor, duplicate Facts are consolidated, and the losing entity is deleted.

**Use when:** identity is certain, upstream source records do not require their own graph identity, and restoring the exact pre-merge state is not an operational requirement.

**What it adds:** the smallest graph and the simplest application queries.

**What it gives up:** the graph no longer records that two canonical entities once existed, which one was selected, or why. Original names can survive on mention relationships, but a reliable split requires external history or reconstruction.

This is the tempting baseline, not the recommended enterprise default.

### 2. Mentions resolve to canonical entities

```text
(Chunk)-[:HAS_MENTION]->(EntityMention)-[:RESOLVED_TO]->(Entity)
```

An `EntityMention` is one extraction occurrence. It retains the surface form, character offsets, proposed type, extracted identifiers, confidence, and extraction lineage. Canonical entities remain separate from those observations.

**Use when:** the project primarily resolves entities extracted from documents and needs incremental processing, ambiguity, or correction.

**What it adds:** a non-destructive boundary between source observations and canonical identity. Changing a resolution does not erase the original extraction.

An unresolved mention has no `RESOLVED_TO` relationship. Candidate relationships may be stored temporarily when several targets are plausible:

```text
(EntityMention)-[:CANDIDATE_FOR {score, method}]->(Entity)
```

This is the recommended default for document-centric GraphRAG. The materialized `RESOLVED_TO` relationship represents current state, not a complete audit history.

### 3. Heterogeneous records form identity clusters

```text
(EntityMention) -------\
(CRMRecord) ------------> (IdentityCluster)-[:MATERIALIZES]->(Entity)
(ReferenceDataRecord) --/
```

An identity cluster groups independently meaningful records believed to represent the same real-world entity. The canonical `Entity` is the current application representation of that cluster.

**Use when:** document extraction is reconciled with structured sources such as CRM, MDM, product catalogs, regulatory data, or external reference datasets, and those source records must remain independently addressable.

**What it adds:** a stable identity container that is separate from both source-system records and the current golden representation. Cluster membership can change while source records retain their original identifiers and values. Canonical properties can be recomputed using explicit survivorship rules.

If one imported structured source is already authoritative, an identity cluster may be unnecessary. Mentions can resolve directly to that authoritative entity:

```text
(EntityMention)-[:RESOLVED_TO]->(AuthoritativeEntity)
```

The demo includes a terminology record, a regulatory record, and document mentions. Initially they occupy two clusters. Merging the clusters produces one canonical Drug; splitting them restores the two identities.

### 4. Audited resolution decisions

```text
(ResolutionDecision)-[:SOURCE]->(Entity)
(ResolutionDecision)-[:SELECTED]->(Entity)
(ResolutionDecision)-[:CONSIDERED {score, signals...}]->(Entity)
(NewDecision)-[:SUPERSEDES]->(PreviousDecision)

(EntityMention)-[:RESOLVED_TO]->(Entity)       // current state
(Entity)-[:MERGED_INTO]->(Entity)              // current redirect
```

A `ResolutionDecision` is the durable record of proposing, accepting, rejecting, merging, splitting, or superseding an identity assignment. Fast application and ingestion queries still use materialized current-state relationships.

**Use when:** merges need human approval, explanations, correction history, rejected alternatives, resolver-version lineage, or regulatory audit. Tracking disagreement between several resolvers is one use case, but even one resolver benefits when consequential decisions must be reviewed and reversed.

**What it adds:** an immutable explanation of who or what decided, when, using which version and signals, and which earlier decision was superseded.

Decision history and identity clusters are orthogonal. A decision can select a canonical Entity in model 2 or assign a record to an IdentityCluster in model 3.

## The merge lifecycle

The saved queries implement the same lifecycle in each model:

1. Create two documents, chunks, Drug entities, Facts, and native `TREATS` relationships.
2. Display the duplicate identities and their provenance.
3. Apply a resolution that selects `DRUG-ACETAMINOPHEN` as the survivor.
4. Reassign source observations according to the model.
5. Move the Paracetamol chunk's support to the surviving canonical Fact.
6. Remove the obsolete Fact and rebuild one native `TREATS` relationship with `supportCount = 2`.
7. Query the application graph and cite both documents.
8. Split the identities again, where the model has enough retained state to do so.

The Cypher is intentionally explicit. Production pipelines should parameterize it, process affected IDs in bounded transactions, and maintain the projection in the same transaction or through a reliable asynchronous process.

### Idempotent writes

The setup and lifecycle queries use `MERGE` with deterministic IDs so retrying an ingestion or materialization step does not create duplicate nodes, support occurrences, decisions, or native relationships. Mutable values are applied with `SET` after the identity pattern has matched. `MERGE` makes writes repeatable; it does not decide that two different entity IDs represent the same real-world entity—that remains the responsibility of the resolution process demonstrated here.

The only `CREATE` statements in the saved-query file create uniqueness constraints with `IF NOT EXISTS`.

## Why a merge is more than a node operation

Before resolution, the demo contains:

```text
Paracetamol   -[:TREATS]-> Fever   supported by DOC-1
Acetaminophen-[:TREATS]-> Fever   supported by DOC-2
```

After resolving both names to one Drug, the two claims have the same normalized subject, predicate, and object. Post-processing must therefore:

- Recompute or look up the canonical Fact identity.
- Move source support without losing occurrence metadata.
- Consolidate duplicate Facts only when all meaning-changing qualifiers also match.
- Refresh the native relationship and its derived support count.
- Retain distinct Facts when polarity, dose, population, time, or other identity qualifiers differ.

Do not blindly deduplicate all relationships between the surviving endpoints. Two relationships can have the same type and endpoints while representing meaningfully different qualified claims.

## Merge, redirect, and split policy

Selecting a surviving entity does not require immediately deleting the losing entity. Models 2 and 4 retain it as a redirect:

```text
(losing:Entity)-[:MERGED_INTO]->(surviving:Entity)
```

This is useful for old IDs, caches, external links, and corrections. Avoid redirect chains: resolve them to the current survivor and periodically flatten them. Application queries should use the canonical projection rather than traversing redirects on every request.

A split is not always the exact inverse of a merge. The system must know which mentions, records, Facts, and property values belong on each resulting entity. Explicit mentions, cluster memberships, and resolution decisions supply that information; a destructive merge does not.

## Canonical property survivorship

The demo chooses `Acetaminophen` as the canonical name and retains `Paracetamol` as an alias. In production, canonical properties should be derived by documented policy, for example:

1. Prefer an approved reference source.
2. Otherwise prefer a reviewed structured record.
3. Otherwise select the highest-confidence active extraction.
4. Preserve alternative values and their sources rather than silently discarding them.

The simple demo stores the selected name and aliases on the canonical entity. If individual property values require provenance, review, or temporal history, represent value assertions explicitly rather than continually expanding arrays on the entity.

## Candidate generation and resolution

The demo begins after candidate generation and uses a supplied match. At enterprise scale, never compare every entity with every other entity. Generate a bounded candidate set using signals such as:

- Exact external identifiers or curated synonym sets.
- Normalized names and aliases.
- Entity-type, tenant, geography, and temporal compatibility.
- Full-text, vector, or phonetic similarity.
- Shared neighborhoods or domain relationships.
- Must-match and must-not-match business rules.

Store scores only when they are operationally useful. Model 4 records the signals that justified an important decision; low-value intermediate candidates can remain transient.

## Incremental processing and deletion

For a merge or split:

1. Capture the affected entity, mention or membership, Fact, and source IDs.
2. Change only those current resolution or membership relationships.
3. Recompute only Facts whose endpoints changed.
4. Consolidate or separate support for those Facts.
5. Refresh only the affected native domain relationships.
6. Remove an entity only when policy allows it and it has no active observations, records, Facts, or redirects requiring retention.

For document deletion, start from an indexed `documentId`, traverse to its chunks and mentions, and clean only the identities and Facts touched by those chunks. Do not scan the full graph after each source update.

At larger scale, deterministic IDs, uniqueness constraints, idempotent writes, bounded transactions, and projection reconciliation are essential. Highly connected canonical entities can become write hotspots, so mutable aggregates such as `supportCount` may be maintained asynchronously or in batches.

## Run this pattern

See the repository [import and run instructions](../README.md#import-and-run) for shared requirements and the database reset warning.

Import `graphrag-entity-resolution-aura-saved-queries.csv` into the saved queries area of the Neo4j Aura console. It creates one top-level folder containing four model folders.

Run one model folder in step order. Every model is self-contained and begins with:

```cypher
MATCH (n) DETACH DELETE n;
```

Use only a disposable demonstration database. Do not run a reset step against a database containing data you need.

Short Cypher comments call out the identity and post-processing operations. Graph-returning steps make the before, merged, and split states visible in Aura.

The complete suite was last run successfully on Neo4j Enterprise 2026.08.0 using the Cypher 25 default language, with no errors or warnings. Each setup step was also run twice and preserved the same node and relationship counts.

## Choosing a model

| Situation | Use | Capability added |
|---|---|---|
| Certain duplicates and no reversal requirement | Physical merge | Smallest canonical graph |
| Document extractions need correction or ambiguity | Mentions to canonical entities | Non-destructive current resolution |
| Structured records and document mentions must be reconciled | Identity clusters | Stable cross-system identity container |
| Decisions require explanation, approval, or history | Resolution decisions | Durable audit and supersession |

For most document-centric GraphRAG projects, begin with model 2. Add model 4 for consequential or reviewed decisions. Use model 3 when structured source records have their own lifecycle and must participate in identity resolution. Physically delete duplicate canonical nodes only when the retention and correction requirements genuinely permit it.
