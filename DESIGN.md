# Architecture Design: Next-Gen AI Data Orchestrator

This document outlines a "from-zero" architecture for a modern data orchestration platform tailored for AI workloads. It aims to combine the reliability of Airflow/Dagster, the streaming capabilities of Flink, and the AI-native features required for RAG/LLM pipelines, while addressing the performance bottlenecks identified in earlier architectures (like "JSON over FFI").

## Core Philosophy

1.  **Vectorized First:** Processing happens on batches (Columnar), not rows.
2.  **Zero-Copy Interop:** Data moves between Python (User Logic) and Rust (Engine) via pointers, not serialization.
3.  **Decoupled State:** State is a log, not a table. Storage is pluggable (Local/S3).
4.  **Logical/Physical Separation:** Python defines the *Intent* (Logical Plan); Rust handles the *Execution* (Physical Plan).

---

## 1. The Stack

| Layer | Component | Technology Choice | Rationale |
| :--- | :--- | :--- | :--- |
| **Interface** | **Python SDK** | Pydantic + Fluent API | Familiar DX. Builds a logical graph, doesn't execute it. |
| **Protocol** | **Logical Plan** | **Substrait** (Protobuf) | Standardized, cross-language representation of compute operations. |
| **Data Plane** | **Memory Format** | **Apache Arrow** | The gold standard for columnar in-memory data. Zero-copy passing. |
| **Engine** | **Executor** | **Rust** + **DataFusion** | State-of-the-art query engine. Native Arrow support. Async. |
| **State** | **State Store** | **LSM Tree** (RocksDB/Sled) or **Delta Log** | High write throughput for CDC/Stream state. No row-locking DBs. |

---

## 2. Architecture Components

### A. The Data Plane: Arrow IPC
Instead of serializing Python objects to JSON to pass to Rust:
1.  Python SDK uses `pyarrow` to wrap data in `RecordBatch`.
2.  Passes the memory address (pointer) of the `RecordBatch` to the Rust engine via the **Arrow C Data Interface**.
3.  Rust reads the data without copying or deserializing.
4.  Rust performs heavy compute (joins, filtering, aggregations).
5.  If a UDF (User Defined Function) in Python is needed (e.g., a LangChain call), Rust passes an Arrow Array back to Python (Zero-Copy).

**Result:** "Ultra Performant" isn't just marketing; it's physics. No serialization tax.

### B. The Execution Engine: Vectorized Streaming
We leverage **Apache DataFusion** (or Polars) as the core execution kernel.
*   **Micro-Batching:** Even for "real-time" streams, we accumulate small batches (e.g., 100ms or 1000 rows).
*   **Operator DAG:** The flow is compiled into a DAG of physical operators.
    *   `SourceOp`: Reads from Kafka/S3/Postgres CDC.
    *   `TransformOp`: vectorized `map`/`filter`.
    *   `ModelInferenceOp`: Specialized operator that batches inputs for GPU inference (e.g., sending a batch of 64 texts to an embedding model).
*   **Backpressure:** Native async Rust handling to prevent OOM when the AI model is slower than the ingestion.

### C. State Management: The "Incremental" Secret
Avoid hitting Postgres for every row.
*   **The Problem:** RAG pipelines need to know "Has this file changed?" or "What is the previous embedding ID?".
*   **The Solution:**
    *   **Hot State (Local):** Use an embedded LSM Tree (like **RocksDB** or **Sled**) for keeping track of file hashes/mod-times. Fast lookups, high write throughput.
    *   **Durable State (Cloud):** Checkpoint the state to S3 as Parquet files + a transaction log (inspired by **Delta Lake** or **DuckDB**).
    *   **CDC:** Instead of "polling and hashing", integrate with **Debezium** or native replication protocols for databases.

---

## 3. Workflow Example: "Embed and Upsert"

**User Python Code:**
```python
@flow
def rag_pipeline():
    # logical definition, no execution yet
    docs = source.s3("bucket/docs/*.pdf")
    chunks = docs.apply(split_text, chunk_size=500)
    embeddings = chunks.apply(generate_embeddings, model="openai/text-embedding-3")
    embeddings.sink(target.qdrant("my_collection"))
```

**Under the Hood:**

1.  **Compilation:** Python SDK generates a `Substrait` plan describing the DAG.
2.  **Submission:** Plan sent to Rust Runtime.
3.  **Execution (Rust):**
    *   **Reader:** Scans S3. Uses `RocksDB` to filter out files unmodified since last run.
    *   **Batching:** Reads valid PDFs into Arrow RecordBatches (binary column).
    *   **UDF Dispatch:** Calls `split_text` (Python) with a *batch* of PDF binaries. Python returns a *batch* of text strings (Arrow List<String>).
    *   **Inference:** Rust accumulates text strings. When batch size = 64, sends to GPU/API for embedding. Receives Float32Array (Arrow FixedSizeList).
    *   **Sink:** Flushes Arrow batches to Qdrant via gRPC. Update "Last Processed" state in RocksDB.

---

## 4. Comparison with Current Repo

| Feature | Current Repo (`cocoindex`) | Proposed Architecture |
| :--- | :--- | :--- |
| **Data Format** | JSON / Custom Enum | Apache Arrow (Columnar) |
| **Execution** | Row-by-row (Iterator) | Vectorized Batches |
| **Language Bridge** | FFI with serialization cost | C Data Interface (Zero Copy) |
| **State Store** | Postgres (Relational) | LSM Tree / Object Store Log |
| **Scalability** | Bound by DB IOPS & Python Loop | Bound by CPU/GPU & Network |

## 5. Conclusion

To build a "Modern Airflow for AI," you must treat **AI Models as I/O devices** and **Data as Tensors/Columns**.

The proposed architecture shifts the bottleneck from the "Python Loop" and "State DB" to the actual compute (Model Inference), which is where it should be.
