# Technical Documentation Style Guide & Standards

> **Editorial principles, taxonomy standards, voice & tone rules, and formatting conventions for high-velocity engineering documentation.**

---

## 1. Core Principles

Documentation is an essential user interface of software products. Our goal is to minimize cognitive friction, accelerate developer time-to-first-success, and maintain authoritative clarity across all written artifacts.

1. **Clarity Over Cleverness**: Use straightforward vocabulary. Avoid idioms, jargon without definitions, or unnecessary literary flourishes.
2. **Actionable First**: Show, then explain. Place runnable code samples and quick commands ahead of lengthy theoretical exposition.
3. **Progressive Disclosure**: Structure content from the simplest mental model to deeper architectural nuances.
4. **Assume High Intelligence, Low Context**: Treat readers as capable engineers who are simply encountering this specific architecture or API for the first time.

---

## 2. Information Architecture: The Diátaxis Framework

All documentation must fit strictly into one of four distinct modes. Do not mix modes within a single document:

```
                  PRACTICAL (How to do)
                            ▲
                            │
         Tutorials          │         How-To Guides
   (Learning-oriented)      │      (Problem-oriented)
                            │
◄───────────────────────────┼───────────────────────────►
THEORY                      │                       WORK
(Understanding)             │                 (Execution)
                            │
        Explanations        │           Reference
   (Concept-oriented)       │     (Information-oriented)
                            │
                            ▼
                 THEORETICAL (What it is)
```

| Mode | Purpose | Reader Intent | Voice & Focus |
| :--- | :--- | :--- | :--- |
| **Tutorial** | Teach beginners by walking through a complete project from scratch. | *"Help me learn the fundamentals."* | Friendly, prescriptive, hand-holding, step-by-step. |
| **How-To Guide** | Solve a specific, real-world task or operational problem. | *"Show me how to fix or configure X."* | Goal-focused, concise, assumes existing foundational knowledge. |
| **Reference** | Provide authoritative, complete, factual specifications. | *"What parameters does this endpoint take?"* | Neutral, systematic, exact, encyclopedic. |
| **Explanation** | Clarify architecture, design decisions, and trade-offs. | *"Why does the system behave this way?"* | Analytical, contextual, visual, high-level. |

---

## 3. Voice, Tone, and Grammar Standards

### 3.1 Active Voice vs. Passive Voice
Always write in the **active voice**. Active voice makes instructions direct and clarifies who or what is performing the action.

- ❌ **Passive**: *"The vector store is queried by the retrieval service after the query string has been transformed."*
- ✅ **Active**: *"The retrieval service converts the query into a vector and searches the vector store."*

### 3.2 Second Person ("You")
Address the reader directly using **"you"**. Avoid the royal "we" unless explicitly referring to the engineering organization or company.

- ❌ **Third person / Impersonal**: *"One must configure their API key prior to initialization."*
- ❌ **Royal We**: *"Now we will create the vector index."*
- ✅ **Second Person**: *"You must set your `OPENAI_API_KEY` before running the assistant."*

### 3.3 Present Tense
Document software state and actions in the **simple present tense**.

- ❌ **Future tense**: *"Clicking this button will open the settings modal."*
- ✅ **Present tense**: *"Clicking this button opens the settings modal."*

---

## 4. Code Block & Snippet Standards

Every code example must be complete, tested, and copy-pasteable.

1. **Always Specify Syntax Highlighting**: Fenced code blocks must declare their language (e.g., ` ```python `, ` ```bash `, ` ```json `).
2. **Differentiate Shell Input from Output**:
   - Use `$` or clear prompts only when demonstrating input vs. output in the same block.
   - For commands intended to be copied in one click, omit the leading `$` so users don't copy the prompt symbol:
     ```bash
     # ✅ Recommended (easy 1-click copy):
     pip install langchain chromadb
     ```
3. **Use Environment Placeholders**:
   - Never hardcode fake secrets that look like real API keys.
   - Use descriptive placeholders: `<YOUR_API_KEY>`, `<COLLECTION_ID>`.

---

## 5. Callouts and Alerts

Use GitHub-style alert callouts sparingly. If a document has five callouts on one page, the reader will ignore all of them:

> [!NOTE]
> Use for useful background context or related documentation links.

> [!TIP]
> Use for performance optimizations, keyboard shortcuts, or time-saving recommendations.

> [!IMPORTANT]
> Use for critical prerequisites or configuration steps that cannot be skipped.

> [!WARNING]
> Use for breaking changes, potential service interruptions, or irreversible actions (e.g. data deletion).

---

## 6. Terminology & Glossary

| Term | Preferred Usage | Avoid |
| :--- | :--- | :--- |
| **Generative AI / GenAI** | Capitalized as "Generative AI" or "GenAI". | "gen-ai", "generative ai" |
| **Large Language Model** | Capitalized as "Large Language Model" or "LLM". | "llm", "large language model" |
| **RAG** | "Retrieval-Augmented Generation (RAG)" on first mention, then "RAG". | "rag", "retrieval augmented generation" |
| **Vector Store / Vector Database** | "Vector store" for embedded in-memory engines; "Vector database" for standalone enterprise clusters. | "Vector DB" (in formal reference docs) |
| **Allowlist / Denylist** | Use inclusive terminology. | "Whitelist / Blacklist" |
| **Primary / Secondary** | Use inclusive architectural terms. | "Master / Slave" |

---

## 7. Pre-Publish Documentation Pull Request Checklist

Before submitting any documentation PR, verify:

- [ ] Does the document fit strictly into one Diátaxis category?
- [ ] Are all code snippets tested and verified against current software releases?
- [ ] Are all headings written in sentence case (e.g. *"Understanding vector embeddings"* rather than *"Understanding Vector Embeddings"*)?
- [ ] Are all internal links relative (`../explanations/...`) and working?
- [ ] Are tables formatted with proper alignment and column headers?
- [ ] Has the document been proofread for passive voice and unnecessary jargon?

---

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
