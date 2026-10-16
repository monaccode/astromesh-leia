# RAGPipeline Manifest (`kind: RAGPipeline`)

Source: astromesh core 0.70.0 (`config/rag/*.rag.yaml`, `astromesh/rag/loader.py`,
`vscode-extension/schemas/rag.schema.json`). An agent uses a pipeline through
`spec.knowledge.pipeline: <metadata.name>`; this file covers the pipeline itself.

```yaml
apiVersion: astromesh/v1
kind: RAGPipeline
metadata:
  name: product-knowledge          # referenced by spec.knowledge.pipeline
  version: "1.0.0"
spec:
  chunking:
    strategy: recursive            # recursive | fixed | sentence | semantic
    chunk_size: 512
    overlap: 50
    separators: ["\n\n", "\n", ". ", " "]
  embeddings:
    provider: ollama               # ollama | hf | sentence_transformer
    model: nomic-embed-text
    endpoint: http://localhost:11434
    dimension: 768
  vector_store:
    backend: pgvector              # pgvector | qdrant | chroma | faiss
    collection: product_docs
    connection: {host: postgres, port: 5432, database: astromesh, user: astromesh, password: astromesh}
  reranking:
    enabled: true
    provider: cross_encoder        # cross_encoder | cohere
    model: cross-encoder/ms-marco-MiniLM-L-6-v2
    top_k: 5
  retrieval:
    top_k: 20
    similarity_threshold: 0.7
```

Allowed `spec` keys: `description`, `chunking`, `embeddings`, `vector_store`, `reranking`, `retrieval`
(unknown keys are rejected by the editor schema). Other `vector_store` keys: `dimensions`, `url`, `host`,
`port`, `api_key`, `dsn`, `persist_directory`. `chunking.similarity_threshold` applies to `semantic`.

## Placement and API

- File: `config/rag/<name>.rag.yaml` on the node; the store is seeded from that directory, and
  `AgentRuntime.reload()` re-reads `rag/` and `agents/` together.
- CRUD: `/v1/rag/pipelines` (GET list, GET/PUT/DELETE by name, POST create; `PUT` needs `metadata.name`
  equal to the path). Ingest and query are separate: `/v1/rag/ingest`, `/v1/rag/query`.
- Keep credentials out of the file where you can (the sample uses a literal password for local dev).
- An agent that names an unknown pipeline only warns and runs without retrieval.

## Library

The open-source `astromesh-rag` library (v0.1.0) builds the same stages (ingest, retrieve, rank, answer)
in Python or from `kind: RAGPipeline` YAML, with patterns `naive`, `hybrid`, `multi_query`, `hyde`.
The core manifest above has no `pattern` key (see the allowed `spec` keys); do not add one.
Use pipeline (similarity over chunks) for large unstructured corpora, bundle for curated knowledge with trust signals.
