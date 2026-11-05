# Tutorial: Building a Local Documentation QA Tool with Python and ChromaDB

A step-by-step guide to building a command-line question-answering tool that indexes markdown files and answers queries using grounded retrieval.

---

## What You Will Build

You will create a lightweight CLI script that:
1. Parses markdown documentation files from a local directory.
2. Segments documents into manageable chunks with overlap using `RecursiveCharacterTextSplitter`.
3. Generates vector embeddings and stores them in an in-memory ChromaDB instance.
4. Executes similarity queries and constructs a prompt containing retrieved snippets.
5. Produces an answer citing the specific source file and section, refusing to guess if the answer is missing.

---

## System Requirements

- Python 3.9, 3.10, or 3.11.
- Terminal access (macOS, Linux, or WSL on Windows).
- An OpenAI API key (or any compatible endpoint supporting the OpenAI standard).

Estimated time: **15 minutes**.

---

## Step 1: Directory Setup and Virtual Environment

Create an isolated project workspace to prevent dependency conflicts with system packages:

```bash
mkdir doc-qa-tool && cd doc-qa-tool
python3 -m venv .venv
source .venv/bin/activate
```

Install the required libraries:

```bash
pip install langchain==0.2.16 langchain-community==0.2.16 langchain-openai==0.1.25 chromadb==0.5.5
```

Export your API key:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

---

## Step 2: Create Sample Documentation Data

Create a `docs/` folder to act as our local knowledge base:

```bash
mkdir docs
```

Write sample technical documentation to `docs/backup_runbook.md`:

```markdown
# Storage Engine Runbook: Immutability and Retention

## Immutability Policies
Retention locks prevent backup snapshots from being pruned or overwritten before a set duration.
When an operator activates an Enterprise Lock on a target bucket:
- The retention period cannot be shortened or cancelled by any role, including cluster root administrators.
- Object data cannot be modified, renamed, or truncated.
- The retention policy can only be extended, never decreased.

## Network Requirements
Inter-cluster replication runs over port 8443 using mutual TLS (mTLS).
- Minimum sustained uplink: 250 Mbps for secondary replication targets.
- Packet loss threshold: Must remain below 0.2%. If packet loss exceeds this, the replication worker throttles to prevent socket starvation.
```

---

## Step 3: Implement the Pipeline

Create `qa_tool.py`:

```python
"""
qa_tool.py
Indexes local markdown files and answers technical questions with source citations.
"""

import os
import sys
from langchain_community.document_loaders import DirectoryLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough


def verify_environment():
    """Ensures necessary environment variables are configured."""
    if not os.environ.get("OPENAI_API_KEY"):
        sys.stderr.write("ERROR: OPENAI_API_KEY environment variable is not set.\n")
        sys.stderr.write("Run: export OPENAI_API_KEY='sk-...'\n")
        sys.exit(1)


def build_index(doc_directory: str = "docs") -> Chroma:
    """Reads markdown files, splits into chunks, and populates ChromaDB."""
    print(f"[INFO] Scanning directory: {doc_directory}")
    loader = DirectoryLoader(
        doc_directory,
        glob="*.md",
        loader_cls=TextLoader,
        loader_kwargs={"autodetect_encoding": True}
    )
    raw_documents = loader.load()

    if not raw_documents:
        raise FileNotFoundError(f"No markdown documents found in '{doc_directory}'.")

    print(f"[INFO] Loaded {len(raw_documents)} file(s). Running recursive splitter...")

    # Chunking: 400 characters (~80-100 tokens) with 50-char overlap
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=400,
        chunk_overlap=50,
        separators=["\n\n", "\n", " ", ""]
    )
    chunks = splitter.split_documents(raw_documents)
    print(f"[INFO] Generated {len(chunks)} text chunks.")

    print("[INFO] Computing embeddings and populating ChromaDB index...")
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    vector_store = Chroma.from_documents(
        documents=chunks,
        embedding=embeddings,
        collection_name="docs_qa"
    )
    return vector_store


def format_context_with_citations(docs) -> str:
    """Formats retrieved document chunks with clear source attribution."""
    formatted_chunks = []
    for idx, doc in enumerate(docs, start=1):
        source_path = doc.metadata.get("source", "Unknown file")
        formatted_chunks.append(
            f"--- Chunk {idx} [Source: {source_path}] ---\n{doc.page_content}"
        )
    return "\n\n".join(formatted_chunks)


def main():
    verify_environment()

    # 1. Index documentation
    vector_store = build_index("docs")
    retriever = vector_store.as_retriever(
        search_type="similarity",
        search_kwargs={"k": 2}
    )

    # 2. Strict grounding prompt template
    system_prompt = (
        "You are an internal documentation assistant for engineering teams.\n"
        "Answer the user query based ONLY on the provided context below.\n\n"
        "Rules:\n"
        "1. If the provided context does not explicitly contain the answer, reply exactly:\n"
        "   'This information is not documented in the provided knowledge base.'\n"
        "2. Do not speculate, extrapolate, or use general outside knowledge.\n"
        "3. Always cite the file source and chunk index at the end of your answer.\n\n"
        "Context:\n"
        "{context}"
    )

    prompt_template = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{question}")
    ])

    llm = ChatOpenAI(
        model="gpt-4o-mini",
        temperature=0.0  # Zero temperature for deterministic responses
    )

    # 3. Assemble LCEL chain
    chain = (
        {
            "context": retriever | format_context_with_citations,
            "question": RunnablePassthrough()
        }
        | prompt_template
        | llm
        | StrOutputParser()
    )

    print("\n[READY] Documentation QA service initialized. Type 'exit' to quit.\n")

    while True:
        try:
            query = input("Query: ").strip()
            if not query:
                continue
            if query.lower() in ("exit", "quit", "q"):
                break

            response = chain.invoke(query)
            print(f"\nResponse:\n{response}\n")
            print("=" * 60)
        except (KeyboardInterrupt, EOFError):
            print("\nShutting down.")
            break


if __name__ == "__main__":
    main()
```

---

## Step 4: Verification and Test Runs

Run the application:

```bash
python3 qa_tool.py
```

### Test Case 1: In-Scope Operational Query

```text
Query: What is the maximum allowed packet loss for inter-cluster replication?
```

**Expected Output**:
```text
Response:
The packet loss threshold must remain below 0.2%. If packet loss exceeds this value, the replication worker throttles to prevent socket starvation.

Citation: docs/backup_runbook.md (Chunk 2)
============================================================
```

### Test Case 2: Negative Fallback Test (Verifying Grounding)

Test how the model handles a question not covered in `backup_runbook.md`:

```text
Query: How do I configure AWS S3 Glacier storage tiering?
```

**Expected Output**:
```text
Response:
This information is not documented in the provided knowledge base.
============================================================
```

The model adheres to rule #1 and halts instead of generating standard AWS documentation from its parametric training data.

---

## Step 5: Engineering Notes for Productionization

If you adapt this script into an internal microservice:

1. **Persistent Vector Storage**: Replace in-memory Chroma with a persistent directory (`persist_directory="./chroma_db"`) or an external service (Qdrant, pgvector) so the index does not re-embed on every process restart.
2. **Deterministic Embeddings Cache**: Wrap the embedding call with a Redis hash cache keyed by `sha256(chunk_content)` to avoid paying API costs on unchanged files during doc builds.
3. **CI Integration**: Hook the script into a GitHub Action that re-indexes whenever markdown files change on the `main` branch.

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
