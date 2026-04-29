# ADR 0002: Use pgvector for Vector Search

Date: 2025-01-05

## Status
Accepted

## Context
We need to store embeddings for the feedback loop. We considered Pinecone, Qdrant, and pgvector.

## Decision
We chose **pgvector**.

## Consequences
+ Simplifies infrastructure (single database for relational + vector data).
+ Transactional consistency (ACID) between feedback logs and embeddings.
- Performance might be lower than specialized vector DBs at >10M vectors (acceptable for MVP).
