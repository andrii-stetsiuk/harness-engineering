# ADR-0001: Compare Qdrant retrieval embeddings

- **Status:** Evaluated; recommendation is provisional
- **Date:** 2026-10-10

## Context

The cluster has two Qdrant MCP implementations and two embedding spaces:

| Retriever | Embedding configuration | Collection |
|---|---|---|
| Project `qdrant-mcp` (`vector_find`) | `nomic-embed-text-v1.5`, served by the cluster's llama.cpp embedding endpoint; 768-dimensional cosine vectors | `abox-nomic` |
| Official `mcp-server-qdrant` (`qdrant-find`) | `sentence-transformers/all-MiniLM-L6-v2` through FastEmbed; 384-dimensional cosine vectors | `abox-official-memory` |

The collections cannot be mixed or searched with each other's query embeddings.
This evaluation asks whether the official server's default MiniLM embedding
performs better for this repository's Kubernetes release manifests.

## Method

- Reused the same 14 sanitized manifest documents and metadata already indexed
  in `abox-nomic`; indexed one document per point into each collection.
- Confirmed both collections contained 14 points after indexing. The default
  collection uses 768-dimensional vectors; the official collection uses
  384-dimensional vectors.
- Ran the same 10 fixed natural-language queries against each server. For the
  default server, requested five results. The official `qdrant-find` tool does
  not expose a result limit, so only its first five results were evaluated.
- Assigned one expected manifest file to each query before comparing ranks.
  A hit means that file appeared in the first five results.
- **Recall@5** is the fraction of queries with the expected file in the top
  five. **MRR@5** is the mean reciprocal rank of the expected file, counting
  misses as zero.

This is a tool-level retrieval benchmark, not an end-to-end evaluation of
agent planning, query reformulation, or answer quality.

## Results

| # | Query | Expected file | Project MCP rank | Official MCP rank |
|---:|---|---|---:|---:|
| 1 | Which manifest configures the Qdrant Helm chart version and namespace? | `qdrant.yaml` | 1 | 1 |
| 2 | Which file defines the official Qdrant MCP deployment, collection, environment variables, and transport? | `mcp-servers.yaml` | 1 | 4 |
| 3 | What does the retrieval agent instruct the model to do with the Qdrant vector search and store tools? | `agent-retrieval.yaml` | 1 | 1 |
| 4 | Which manifest configures the embeddings model and serving endpoint used by the cluster? | `llama-cpp-embeddings.yaml` | 2 | 1 |
| 5 | How is Kagent deployed, and what release configuration does it use? | `kagent.yaml` | 1 | 1 |
| 6 | Which manifest configures the Neo4j MCP connection and database settings? | `neo4j.yaml` | 1 | 3 |
| 7 | How is the Agentgateway controller installed and its external Gateway configured? | `agentgateway.yaml` | 1 | 1 |
| 8 | Which manifest defines the llm-d inference deployment and routing configuration? | `llmd.yaml` | 1 | 1 |
| 9 | Where are the Kubernetes Gateway API CRDs installed from, and what version is used? | `crds/gateway-api-crds.yaml` | 1 | 1 |
| 10 | Which release manifest configures Phoenix for tracing and observability? | `phoenix.yaml` | 1 | 1 |

| Retriever | Recall@5 | MRR@5 | Expected file at rank 1 |
|---|---:|---:|---:|
| Project `qdrant-mcp` / Nomic | 1.00 (10/10) | 0.950 | 9/10 |
| Official MCP / MiniLM | 1.00 (10/10) | 0.858 | 8/10 |

Both retrievers found the expected manifest within five results for every
query. The project retriever ranked the expected file first more often and
had the higher MRR@5 by 0.092. MiniLM ranked the embeddings manifest first
where the project retriever ranked it second; the project retriever ranked
the MCP configuration and Neo4j manifests higher.

## Decision

For the current manifest corpus, keep the project `qdrant-mcp` and its Nomic
collection as the retrieval-agent default. Keep the official MCP/MiniLM
collection as an independently indexed experiment; do not switch the agent
or combine the collections based on this small benchmark.

The result is a weak, corpus-specific signal rather than evidence that Nomic
is generally superior. A larger, independently judged set of questions and
an end-to-end agent evaluation are needed before making a general model
choice.

## Limitations and operational notes

- There are only 10 queries over 14 documents, with one exact expected file
  per query. Judgments are not graded and do not capture other useful results.
- Queries target facts present in this repository and may favor its wording.
- This isolates semantic retrieval; it does not measure agent planning,
  follow-up tool calls, answer correctness, latency, or user satisfaction.
- The official `qdrant-find` schema has no result-limit argument, unlike the
  project `vector_find` tool.
- The deployed `qdrant-official-mcp` resource was still being reconciled by
  Flux from an older published release containing unsupported CLI flags.
  The official server was therefore run in a temporary standalone pod for
  this evaluation; that pod was removed afterward. The source manifest now
  uses the supported `--transport stdio` argument, but the published release
  must be updated and reconciled before the deployed MCP server uses it.
- The embedding package invocation is not version-pinned in the deployment
  manifest. Pin the package version and rerun the benchmark for strict
  reproducibility.
