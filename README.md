# Technical Writing and Documentation Portfolio

Technical documentation, architecture write-ups, and developer guides by Vivek.

I am an engineering student with a focus on systems and Generative AI who enjoys writing documentation. Most developers treat documentation as an afterthought once the code is committed. I see documentation as part of the engineering surface area—if developers cannot integrate an API or debug a failure state from the docs, the implementation is incomplete.

This repository collects technical guides, API references, hands-on tutorials, and evaluation workflows I wrote while building and testing retrieval pipelines and developer tools.

For more background on my focus and engineering journey, read my [Background and Personal Statement](ABOUT.md).

---

## Documentation Index

The guides follow the Diataxis documentation structure (Tutorials, How-To Guides, Reference, Explanations):

| Document | Type | Focus | Target Reader |
| :--- | :--- | :--- | :--- |
| [Retrieval-Augmented Generation: Architecture and Failure Modes](docs/explanations/demystifying-rag.md) | Explanation | Chunking trade-offs, vector search mechanics, latency bottlenecks, and real failure modes. | Backend engineers, technical product managers |
| [Vector Embeddings and Similarity Metrics](docs/explanations/vector-embeddings.md) | Explanation | High-dimensional geometric representation, cosine similarity vs. dot product, and sparse vs. dense search. | Software developers, data engineers |
| [Building a Grounded Documentation Assistant](docs/tutorials/building-first-rag-agent.md) | Tutorial | End-to-end Python script using ChromaDB and LangChain with strict zero-hallucination prompt constraints. | Developers building their first RAG service |
| [Measuring Groundedness and Faithfulness in LLM Pipelines](docs/how-to-guides/evaluating-llm-outputs.md) | How-To Guide | Automated evaluation script measuring factual consistency, precision, and regression testing in CI. | MLOps and platform engineers |
| [REST API Reference: Document Ingestion and Query Service](docs/api-reference/semantic-search-api.md) | Reference | Formal HTTP endpoints, JSON schemas, headers, error states, and curl examples. | Integrators and API consumers |
| [Engineering Documentation Style Guide](docs/style-guide/technical-style-guide.md) | Standards | Voice and tone, active voice rules, code snippet standards, and pre-merge checklist. | Writers and contributing engineers |

---

## Technical Background and Tools

- **Core Engineering**: Python, REST APIs, JSON schema validation, Git, Bash shell scripting.
- **AI and Search**: Large Language Models, embeddings, vector indexing (HNSW, ChromaDB, FAISS), hybrid search (BM25 + dense vectors), RAGAS evaluation metrics.
- **Documentation Toolchain**: Markdown, GitHub Flavored Markdown (GFM), Mermaid diagrams, OpenAPI/Swagger specifications, Vale (prose linting), MkDocs.

---

## Structure and Philosophy

When writing technical documents, I follow four basic rules:

1. **Be direct and skip the fluff.** Avoid marketing buzzwords and filler phrases like "in today's fast-paced digital world." Start directly with the problem definition or prerequisite check.
2. **Include concrete numbers and trade-offs.** Theoretical discussions without latency figures, memory impacts, or cost comparisons rarely help an engineer make a decision.
3. **Make code examples runnable.** Every code snippet must run out of the box with realistic inputs, explicit import statements, and expected outputs.
4. **Design for scanning.** Engineers read documentation while debugging or implementing. Use clear headers, comparative tables, and callouts so the reader finds their answer in under 30 seconds.

---

## Author

- **Name**: Vivek
- **GitHub**: [github.com/vivek12x](https://github.com/vivek12x)
- **License**: MIT
