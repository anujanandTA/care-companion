---
name: cypher-architect
description: 'Use when writing or reviewing Neo4j Cypher for GraphCare, especially graph schema design, node and relationship creation, deduplication, canonical entity lookup, and safe parameterized queries. Prevents made-up labels and relationships.'
argument-hint: 'Describe the graph change, query, or schema task'
---

# Cypher Architect

Specialized guidance for GraphCare Neo4j work. This skill exists to keep Cypher aligned with the project schema and to prevent hallucinated labels or relationship types.

## When To Use
- Writing Cypher queries for inserts, updates, or retrieval
- Designing or reviewing graph schema changes
- Mapping extracted triplets into Neo4j nodes and relationships
- Debugging duplicate entities or bad graph merges
- Converting normalized clinical facts into graph-ready writes

## Core Rules
- Only use node labels that are already part of the GraphCare schema: `Patient`, `Symptom`, `Medication`, `Condition`, `ClinicalEvent`.
- Only use relationship types that are already approved by the schema or explicitly requested in the task.
- Do not invent alternatives such as `SUFFERS_FROM` when the schema already uses `HAS_CONDITION`.
- Prefer `Condition` over `Diagnosis` unless the repository schema is explicitly updated.
- Always use parameterized Cypher queries. Never interpolate user or note content directly into query strings.
- Normalize synonyms to canonical entities before any write.
- Deduplicate at the graph layer by matching existing canonical nodes before creating new nodes.

## Baseline GraphCare Schema
- `(:Patient)-[:HAS_SYMPTOM]->(:Symptom)`
- `(:Patient)-[:HAS_CONDITION]->(:Condition)`
- `(:Patient)-[:TAKES_MEDICATION]->(:Medication)`
- `(:ClinicalEvent)-[:TRIGGERS]->(:ClinicalEvent)` only when event-to-event causality is explicitly modeled

If a requested edge does not fit the baseline schema, stop and propose a schema change instead of inventing a relationship.

## Procedure
1. Identify the canonical entities from the source note or task.
2. Map each entity to an approved node label.
3. Map each fact to an approved relationship type.
4. Match existing canonical nodes before creating any new nodes.
5. Use `MERGE` for canonical nodes and stable relationship creation.
6. Use parameters for all dynamic values.
7. Return the Cypher with a short explanation of how it matches the schema.

## Patterns

### Canonical Node Merge
```cypher
MERGE (symptom:Symptom {name: $canonical_symptom})
```

### Patient To Symptom Link
```cypher
MATCH (patient:Patient {patient_id: $patient_id})
MERGE (symptom:Symptom {name: $canonical_symptom})
MERGE (patient)-[:HAS_SYMPTOM]->(symptom)
```

### Patient To Condition Link
```cypher
MATCH (patient:Patient {patient_id: $patient_id})
MERGE (condition:Condition {name: $canonical_condition})
MERGE (patient)-[:HAS_CONDITION]->(condition)
```

## Anti-Patterns
- Creating both `HF` and `Heart Failure` as separate nodes
- Using `CREATE` where `MERGE` is required for canonical entities
- Mixing `Condition` and `Diagnosis` labels for the same concept
- Inventing new relationships because they sound natural
- Building Cypher strings with concatenated note text

## Output Expectations
- Return valid Cypher only if the schema mapping is clear.
- If the schema is underspecified, say what is missing and propose the smallest explicit schema update.
- When reviewing a query, identify any mismatched labels, relationships, or unsafe parameter handling first.