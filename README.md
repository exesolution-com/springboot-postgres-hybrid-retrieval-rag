# springboot-postgres-hybrid-retrieval-rag
Hybrid Retrieval RAG in PostgreSQL Keyword + pgvector + Rank Fusion (Spring Boot), this solution implements hybrid retrieval by combining two independent PostgreSQL-based retrieval paths.

# Hybrid Retrieval RAG in PostgreSQL: Keyword + pgvector + Rank Fusion (Spring Boot)


## 1) Problem

Production RAG systems frequently fail to retrieve the correct context due to over-reliance on a single retrieval strategy. Pure vector similarity often misses exact terms, identifiers, and domain-specific keywords. Pure keyword search lacks semantic recall and performs poorly on paraphrased queries. In production environments, teams require a deterministic, auditable hybrid retrieval approach that is fully runnable, observable, and cost-controlled.

## 2) Solution Overview (architecture described in text)

This solution implements hybrid retrieval by combining two independent PostgreSQL-based retrieval paths:

1. Keyword retrieval using PostgreSQL full-text search (`tsvector` + `tsquery`).
2. Semantic retrieval using `pgvector` cosine similarity over precomputed embeddings.

Each retrieval path executes independently with bounded limits. Results are merged using deterministic Reciprocal Rank Fusion (RRF). The fused ranking determines the final document set passed into the RAG generation layer via Spring AI abstractions. PostgreSQL serves as the sole persistence and retrieval engine.
View [Full runnable solution here](https://exesolution.com/solutions/springboot-postgres-hybrid-retrieval-rag)

## 3) Runnable Implementation

### Project structure

```
.
├── docker-compose.yml
├── Dockerfile
├── pom.xml
├── src/main/java
│   └── com/example/hybridrag
│       ├── HybridRagApplication.java
│       ├── config/
│       ├── ingestion/
│       ├── retrieval/
│       └── api/
├── src/main/resources
│   ├── application.yml
│   └── db/migration
│       ├── V1__init.sql
│       └── V2__indexes.sql
└── evidence/
```

### Configuration

Environment variables:

```
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=hybridrag
POSTGRES_USER=rag
POSTGRES_PASSWORD=ragpass
SPRING_AI_EMBEDDING_MODEL=text-embedding-3-large
```

Key `application.yml` settings:

```
rag:
  retrieval:
    keyword-limit: 20
    vector-limit: 20
    rrf:
      k: 60
  embedding:
    dimensions: 3072
```

### Commands to run

```
docker compose build
docker compose up
```

Query the API:

```
curl -X POST http://localhost:8080/api/query \
  -H "Content-Type: application/json" \
  -d '{"query":"postgres vector similarity indexing"}'
```

### Expected outputs

- Flyway migrations applied successfully.
- pgvector extension enabled.
- Deterministic ranked results returned.

## 4) Data Model / Storage

**documents**
- id (UUID, PK)
- content (text)
- tsv (tsvector)
- embedding (vector(3072))
- metadata (jsonb)

Indexes:
- GIN(tsv)
- ivfflat on embedding with cosine ops

Vector dimension is fixed to 3072 to match embedding model output and validated at ingestion time.

## 5) Performance & Cost Controls

- Two-phase bounded retrieval
- Deterministic RRF avoids score normalization drift
- PostgreSQL query plans validated via `EXPLAIN ANALYZE`
- No external search or vector infrastructure

## 6) Security & Compliance Notes

- No user prompts persisted by default
- Embeddings stored as numeric vectors only
- Environment-variable based secrets
- Ready for Spring Security integration

## 7) Operational Runbook

- Health endpoint: `/actuator/health`
- Monitor DB CPU, index hit ratio, query latency
- Rollback via Docker image pinning
- Reindex using `REINDEX INDEX CONCURRENTLY`

## 8) Verification & Evidence Pack

Included under `/evidence`:
- Startup logs
- Sample hybrid query logs
- SQL validation scripts

Sample log:

```
HybridRetrievalService : keywordHits=20 vectorHits=20 fusedResults=10 rrfK=60
```

## 9) Changelog

### 1.0.0
- Initial production release

## 10) FAQ

**Why PostgreSQL-only?**  
Reduces operational complexity and improves transactional consistency.

**Can ranking be customized?**  
Yes, the fusion layer is isolated.

**Is this production safe?**  
Yes, with standard access controls and monitoring enabled.

View [More runnable solutions here](https://exesolution.com)
