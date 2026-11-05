# Vector Embeddings and Similarity Metrics in Search Systems

A technical explainer on how embedding models map language into vector spaces, distance metric math, and memory considerations in production search indices.

---

## The Limitations of Keyword Search

Traditional search engines rely on inverted indexes and term frequency algorithms like BM25:

$$BM25(D, Q) = \sum_{i=1}^{n} IDF(q_i) \cdot \frac{f(q_i, D) \cdot (k_1 + 1)}{f(q_i, D) + k_1 \cdot \left(1 - b + b \cdot \frac{|D|}{\text{avgdl}}\right)}$$

This approach is fast and works well when users know exact function names, CLI parameters, or error codes. However, it fails in two common documentation scenarios:

1. **Vocabulary Mismatch**: An engineer searching for `"how to recover wiped volume"` will not match a runbook titled `"Restoring Purged Snapshots"`, even though the underlying action is identical.
2. **Contextual Ambiguity**: The term `"driver"` means a kernel module in hardware docs, a database connector in backend code, or a golf club in sports. Lexical search scores them identically based solely on token occurrence.

**Vector embeddings** resolve this by converting text into dense mathematical coordinates where position represents semantic meaning rather than literal spelling.

---

## What an Embedding Actually Is

An embedding model (such as `text-embedding-3-small` or `bge-base-en-v1.5`) is a neural network that processes a string of tokens and outputs an array of fixed-length 32-bit floating-point numbers:

```python
# A 1536-dimensional vector representation
embedding = [-0.0182, 0.0431, -0.0094, 0.1205, ..., -0.0311]
```

Think of this array as a point in a high-dimensional coordinate system. During training across billions of text pairs, the network learns to place sentences with similar meanings close together in this coordinate space.

---

## Comparing Vectors: The Three Distance Metrics

Once documents and queries are represented as vectors, finding the most relevant documentation chunk becomes a geometry problem.

```
       Cosine Similarity (Angle θ)                Euclidean Distance (L2)
             ▲                                            ▲
             │       / Vector B                           │       / Vector B
             │      /                                     │      /
             │     /                                      │     /  |
             │    / θ                                     │    /   | Distance (d)
             │   /                                        │   /    |
             └───┴────────────────► Vector A              └───┴────▼───────────► Vector A
```

### 1. Cosine Similarity

Measures the cosine of the angle $\theta$ between two vectors, regardless of their magnitude:

$$\text{Cosine Similarity} = \frac{\mathbf{A} \cdot \mathbf{B}}{\|\mathbf{A}\| \|\mathbf{B}\|} = \frac{\sum_{i=1}^{n} A_i B_i}{\sqrt{\sum_{i=1}^{n} A_i^2} \sqrt{\sum_{i=1}^{n} B_i^2}}$$

- **Output Range**: $[-1.0, 1.0]$. In practice, normalized text embeddings typically score between $0.4$ and $0.95$.
- **Why it is preferred for text**: A 30-word summary and a 300-word detailed guide on the same topic will point in nearly the same angular direction, even though the longer guide has a larger vector magnitude. Cosine similarity evaluates the topic rather than document length.

### 2. Dot Product (Inner Product)

Calculates the sum of products across all dimensions:

$$\mathbf{A} \cdot \mathbf{B} = \sum_{i=1}^{n} A_i B_i$$

- **Computational Advantage**: Dot product avoids the square root operations required by Cosine Similarity.
- **Unit Normalization**: If vectors are pre-normalized so their length is $1.0$ ($\|\mathbf{A}\| = 1$), the denominator of the Cosine Similarity equation equals $1$, making **Dot Product mathematically identical to Cosine Similarity**. Most modern vector stores (e.g., Pinecone, Qdrant, Milvus) expect pre-normalized vectors to leverage this performance gain.

### 3. Euclidean Distance (L2)

Measures the straight-line physical distance between coordinates:

$$d(\mathbf{A}, \mathbf{B}) = \sqrt{\sum_{i=1}^{n} (A_i - B_i)^2}$$

- **Output Range**: $[0, \infty)$, where $0$ indicates identical coordinates.
- **Limitation**: Highly sensitive to vector magnitude. Unless vectors are strictly normalized, two documents with identical themes but different lengths can show large Euclidean distance.

---

## Metric Comparison Matrix

| Metric | Computational Cost | Invariant to Length? | Typical Use Case |
| :--- | :--- | :--- | :--- |
| **Cosine Similarity** | Moderate (dot product + norms) | Yes | Text documentation search, semantic clustering. |
| **Dot Product** | Lowest (SIMD hardware-accelerated) | No (unless pre-normalized) | High-throughput vector search on pre-normalized vectors. |
| **Euclidean (L2)** | Moderate (differences, squares, sum, sqrt) | No | Image feature matching, spatial sensor clustering. |

---

## Systems and Memory Calculations

When deploying vector search at scale, index memory footprint is often the primary infrastructure constraint.

### Memory Footprint Formula

Each dimension in a standard float32 vector consumes 4 bytes of RAM:

$$\text{Memory per Vector} = \text{Dimensions} \times 4\text{ bytes}$$

For an index containing 250,000 document chunks using a 1536-dimensional model:

- $1536 \times 4\text{ bytes} \approx 6.144\text{ KB per vector}$
- $250,000 \times 6.144\text{ KB} \approx 1.536\text{ GB raw vector data}$
- With HNSW graph index overhead ($\approx 1.25\times$ to $1.5\times$), total RAM requirement reaches **~2.2 GB**.

### Optimization: Scalar Quantization (SQ8)
To reduce infrastructure cost, production systems apply **Scalar Quantization (SQ8)**, converting 32-bit floats into 8-bit signed integers. This reduces memory footprint by 75% (from 4 bytes to 1 byte per dimension) with less than a 1.5% drop in retrieval recall.

---

## Engineering Takeaways

1. Pre-normalize all embedding vectors at ingestion time so your database can use Dot Product rather than running costly norm calculations at query time.
2. Keep dimensions proportional to data size. For documentation sets under 50,000 pages, 384-dimensional or 768-dimensional models (`all-MiniLM-L6-v2` or `bge-base`) often yield identical top-5 recall to 3072-dimensional models while cutting RAM and search latency in half.
3. Always benchmark recall against a golden test set before deciding whether you need hybrid search or dense-only retrieval.

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
