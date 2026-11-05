# Engineering Documentation Style Guide and Standards

Editorial principles, taxonomy standards, voice and tone rules, and automated linting standards for technical documentation.

---

## 1. Core Principles

Documentation is part of the product interface. The goal is to minimize time-to-first-success for developers, maintain architectural clarity, and reduce support escalations.

1. **Direct and unambiguous**: Use straightforward vocabulary. Avoid filler phrases, idioms, and undefined internal acronyms.
2. **Action before theory**: Place prerequisites, working code, and command snippets ahead of lengthy architectural background.
3. **Progressive disclosure**: Introduce the core workflow first, then cover edge cases, performance flags, and configuration overrides.
4. **Assume high intelligence, zero context**: Treat readers as competent engineers who simply have not seen this specific codebase, system architecture, or API design before.

---

## 2. Information Architecture: The Diataxis Framework

Documentation must be categorized into one of four distinct modes:

```
                  PRACTICAL (How-to)
                          |
         Tutorials        |       How-To Guides
    (Learning-oriented)   |    (Problem-oriented)
                          |
--------------------------+--------------------------
THEORY                    |                      WORK
(Understanding)           |               (Execution)
                          |
       Explanations       |        Reference
    (Concept-oriented)    |  (Information-oriented)
                          |
                  THEORETICAL (What it is)
```

| Mode | Purpose | Reader State | Tone |
| :--- | :--- | :--- | :--- |
| **Tutorial** | Teach a beginner by building a complete working example. | "I want to get started." | Instructive, guided, step-by-step. |
| **How-To Guide** | Guide an engineer through solving a specific operational problem. | "I have a specific task to solve." | Recipe-driven, direct, assumes baseline literacy. |
| **Reference** | Provide exhaustive technical facts and interfaces. | "I need exact specifications." | Encyclopedic, neutral, structured. |
| **Explanation** | Clarify architecture, constraints, and engineering decisions. | "I want to understand why." | Analytical, comparative, high-level. |

---

## 3. Voice, Tone, and Grammar Standards

### 3.1 Active Voice

Write instructions in the active voice. Active voice identifies who or what performs the action.

- **Avoid**: "The vector database is queried by the retrieval worker once the embedding has been computed."
- **Preferred**: "The retrieval worker computes the query embedding and searches the vector database."

### 3.2 Second Person ("You")

Address the user directly as "you". Do not use the royal "we" unless describing actions taken by the platform or engineering team.

- **Avoid**: "One must set the authorization token prior to making requests."
- **Avoid**: "Now we will configure our ChromaDB instance."
- **Preferred**: "Set your `OPENAI_API_KEY` before starting the script."

### 3.3 Present Tense

Document product behavior in the present tense.

- **Avoid**: "Clicking this link will launch the cluster manager."
- **Preferred**: "Clicking this link launches the cluster manager."

---

## 4. Code Snippet Standards

Every code example must be complete, tested against a real environment, and copy-pasteable.

1. **Always specify syntax highlighting**: Every fenced block must include its language tag (`python`, `bash`, `json`, `yaml`).
2. **Commands for one-click copying**:
   - For shell commands, do not prefix with `$`. Users should be able to click copy and run the line directly without editing out the prompt character:
     ```bash
     pip install langchain chromadb
     ```
   - Only include terminal prompts when contrasting shell input with program output in the same block.
3. **Environment placeholders**:
   - Never use fake credentials that resemble active keys.
   - Use uppercase descriptive tokens: `<YOUR_API_KEY>`, `<COLLECTION_ID>`, `<PORT>`.

---

## 5. Terminology and Glossary

| Concept | Preferred Usage | Avoid |
| :--- | :--- | :--- |
| Artificial Intelligence | Generative AI, GenAI | Gen-AI, generative AI (in headers) |
| Models | Large Language Model, LLM | large language model |
| Retrieval | Retrieval-Augmented Generation (RAG) | RAG architecture (redundant), rag |
| Vector Index | Vector store, Vector database | Vector DB (in formal reference specs) |
| System Boundaries | Allowlist / Denylist | Whitelist / Blacklist |
| Node Topologies | Primary / Secondary | Master / Slave |

---

## 6. Automated Linting and CI Standards

To keep documentation clean across multiple contributors, maintain these automated checks:

### Markdownlint Configuration (`.markdownlint.json`)

```json
{
  "MD013": { "line_length": 120, "code_blocks": false, "tables": false },
  "MD033": false,
  "MD024": { "siblings_only": true }
}
```

### Prose Linting with Vale (`.vale.ini`)

Use Vale in your local pre-commit hook to catch passive voice and jargon:

```ini
StylesPath = styles
MinAlertLevel = warning

[*.md]
BasedOnStyles = Vale, Google
Vale.Spelling = YES
Google.Passive = warning
Google.We = warning
```

---

## 7. Pre-Merge Documentation Checklist

Before submitting a documentation pull request, confirm:

- [ ] Fits cleanly into one Diataxis category (Tutorial, How-To, Reference, or Explanation).
- [ ] Code samples executed and validated against current release.
- [ ] No unformatted JSON or unhighlighted code blocks.
- [ ] All relative links verified and functional.
- [ ] Terminology aligns with the glossary standards.
- [ ] No decorative emojis used in headings or technical prose.

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
