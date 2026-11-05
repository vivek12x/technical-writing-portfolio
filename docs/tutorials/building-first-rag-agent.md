# Tutorial: Building Your First Knowledge-Grounded AI Assistant

> **A step-by-step practical guide to building an interactive documentation assistant in Python that cites its sources and refuses to hallucinate.**

---

## Overview

In this tutorial, you will construct a fully functional, knowledge-grounded Question & Answering (Q&A) assistant. By the end of this guide, you will have built a Python-based pipeline that:

1. Ingests local markdown documentation files.
2. Splits documents into semantically coherent chunks with overlap.
3. Indexes text chunks into a vector store.
4. Retrieves relevant context when a user asks a question.
5. Synthesizes a verified, cited response using an LLM.

---

## Prerequisites

Before starting, ensure you have:

- **Python 3.9+** installed on your system.
- Basic familiarity with running commands in a terminal.
- An API key for an LLM provider (e.g., OpenAI or compatible API), or a local LLM setup.

Estimated completion time: **15–20 minutes**.

---

## Step 1: Set Up the Environment

Create an isolated directory and install the necessary dependencies:

```bash
mkdir doc-assistant && cd doc-assistant
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

pip install langchain langchain-community langchain-openai chromadb
```

Set your API key in your terminal session:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

---

## Step 2: Prepare Sample Knowledge Base Documents

Create a directory named `data/` and add a sample product manual markdown file:

```bash
mkdir data
```

Create `data/storage_policy_guide.md`:

```markdown
# Enterprise Cloud Storage Policies

## Immutability & Retention Locks
Retention locks protect backup snapshots from unauthorized modification or early deletion.
When an administrator enables a Compliance Lock on a storage view, the following rules apply:
- The retention period cannot be shortened by any user, including root administrators.
- Files cannot be renamed, overwritten, or truncated until the retention window expires.
- Legal holds can be placed independently, overriding default expiration timers.

## Replication Topology
Replication jobs run asynchronously across geographical regions. 
- Minimum network bandwidth requirement: 100 Mbps dedicated uplinks.
- Recovery Point Objective (RPO): Target 15 minutes for critical tier-1 databases.
- Failover testing can be conducted in sandbox mode without interrupting live production sync.
```

---

## Step 3: Write the Ingestion and Retrieval Pipeline

Create a file named `assistant.py` with the following implementation:

```python
"""
assistant.py
A minimal, production-pattern RAG pipeline for querying local technical documentation.
"""

import os
from langchain_community.document_loaders import DirectoryLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough


def build_vector_store(data_dir: str = "data") -> Chroma:
    """Loads documents, chunks text, and stores vectors in an in-memory Chroma database."""
    print("📂 Loading documentation files...")
    loader = DirectoryLoader(data_dir, glob="*.md", loader_cls=TextLoader)
    documents = loader.load()

    print(f"📄 Loaded {len(documents)} document(s). Chunking text...")
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=400,
        chunk_overlap=50,
        separators=["\n\n", "\n", " ", ""]
    )
    chunks = splitter.split_documents(documents)
    print(f"🧩 Created {len(chunks)} chunks.")

    print("🧠 Generating embeddings and populating vector store...")
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    vector_store = Chroma.from_documents(chunks, embeddings)
    return vector_store


def format_docs(docs) -> str:
    """Formats retrieved document chunks with clear source attribution."""
    formatted = []
    for i, doc in enumerate(docs, start=1):
        source = doc.metadata.get("source", "Unknown Source")
        formatted.append(f"[Chunk {i} | Source: {source}]\n{doc.page_content}")
    return "\n\n---\n\n".join(formatted)


def main():
    if not os.environ.get("OPENAI_API_KEY"):
        raise ValueError("Please export OPENAI_API_KEY before running this script.")

    # 1. Build Index
    vector_store = build_vector_store()
    retriever = vector_store.as_retriever(search_kwargs={"k": 2})

    # 2. Define System & User Prompt with Grounding Guardrails
    system_prompt = (
        "You are an expert technical documentation assistant.\n"
        "Answer the user's question STRICTLY based on the provided context below.\n"
        "Guidelines:\n"
        "1. If the context does not contain the answer, say: 'I cannot find that in the documentation.'\n"
        "2. Do not speculate or introduce facts outside the context.\n"
        "3. Cite the chunk number and source at the end of your explanation.\n\n"
        "Context:\n{context}"
    )

    prompt = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{question}")
    ])

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.0)

    # 3. Assemble RAG Chain (LCEL)
    rag_chain = (
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | prompt
        | llm
        | StrOutputParser()
    )

    print("\n✅ Knowledge Base Ready! Type 'exit' to quit.\n")

    # 4. Interactive Loop
    while True:
        try:
            query = input("❓ Ask a documentation question: ").strip()
            if not query:
                continue
            if query.lower() in ["exit", "quit", "q"]:
                break

            response = rag_chain.invoke(query)
            print(f"\n💡 Answer:\n{response}\n")
            print("-" * 60 + "\n")
        except KeyboardInterrupt:
            break


if __name__ == "__main__":
    main()
```

---

## Step 4: Run and Test the Assistant

Launch your script:

```bash
python3 assistant.py
```

### Test Case 1: Grounded Factual Query
**Prompt**:
```text
❓ Ask a documentation question: Can an admin reduce the retention period under Compliance Lock?
```

**Expected Output**:
```text
💡 Answer:
No, when an administrator enables a Compliance Lock, the retention period cannot be shortened by any user, including root administrators. Furthermore, files cannot be renamed, overwritten, or truncated until the retention window expires.

[Source: data/storage_policy_guide.md | Chunk 1]
```

### Test Case 2: Out-of-Scope Query (Verifying Hallucination Guardrail)
**Prompt**:
```text
❓ Ask a documentation question: What is the company's pricing tier for 50 terabytes?
```

**Expected Output**:
```text
💡 Answer:
I cannot find that in the documentation.
```

Notice how the zero-temperature setting combined with negative constraint prompting reliably halts ungrounded guessing.

---

## Step 5: Key Takeaways & Best Practices

1. **Deterministic Temperature (`0.0`)**: For technical documentation assistants, always set the LLM temperature to `0.0`. Creative variance is harmful when users need exact operational procedures.
2. **Chunk Attribution**: Always retain file paths and chunk identifiers in metadata. Citing exact sources builds user confidence and simplifies auditing.
3. **Negative Constraint Directives**: Explicit instructions like *"If the context does not contain the answer, say..."* reduce false confidence by over 90% in retrieval tasks.

---

## Next Steps

- Explore [Evaluating GenAI Outputs](../how-to-guides/evaluating-llm-outputs.md) to learn how to measure context precision and retrieval recall mathematically.
- Review our [Semantic Search REST API](../api-reference/semantic-search-api.md) for enterprise client integration patterns.

---

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
