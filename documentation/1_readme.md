### Day-1 Learnings:

## 1. RAG (Retrieval Augmented Generation)

`Docs -> Chunk -> Embed -> Store -> Retrieve -> Generate`

**Why RAG?** An LLM only knows what was in its training data and what fits in its
context window. For a private codebase neither is true — the model has never seen it,
and the whole repo won't fit in context. RAG fixes this by retrieving only the few
chunks relevant to a question and handing them to the model at generation time.

Every approach below is just a different answer to two questions: **how do we cut the
code into chunks?** and **how do we decide which chunks are relevant to a query?**

---

## 2. Semantic RAG (vector similarity)

   - Stores chunks as embedding vectors and retrieves them by **cosine similarity** to
     the embedded query — i.e. it matches on *meaning*, not exact words.
   - This demo loads `sample_project/`, chunks each `.py` file with
     `RecursiveCharacterTextSplitter`, embeds chunks with `all-MiniLM-L6-v2`
     (HuggingFace), stores them in Chroma, and exposes a `search_codebase` retriever
     tool to a `create_agent` agent.

   Implementation: [rag/1_semantic_rag.py](../rag/1_semantic_rag.py)

   **Why this approach:** a user rarely types the exact words that appear in the code.
   They ask *"where do we charge the customer?"* and the relevant function is called
   `process_payment`. Embeddings capture that semantic link.

   | Pros | Cons |
   | --- | --- |
   | Matches intent / paraphrases, not just keywords | Misses **exact tokens** — an error string, a rare API name, or a UUID may not embed close to the query |
   | Robust to synonyms and natural-language questions | Embedding + vector store adds cost, latency, and a model dependency |
   | Works across phrasing the author never anticipated | Quality is only as good as the **chunks** you feed it (see §3) |

   **Limitation that drives us forward:** semantic retrieval is only as good as its
   chunks. If a chunk is half of one function glued to half of another, the embedding
   represents nothing coherent. So the next question is *how* we chunk.

---

## 3. Chunking strategy — RecursiveCharacterTextSplitter

- Refer to [RecursiveCharacterTextSplitter](https://python.langchain.com/en/latest/modules/indexes/text_splitters/examples/recursive_character_text_splitter.html) for more details.

- It tries a prioritized list of separators and falls back to finer ones only when a
  chunk is still too big:

   ```
   Try splitting by "class"
      ↓ Still too big?
   Try splitting by "def"
      ↓ Still too big?
   Try splitting by blank lines
      ↓ Still too big?
   Try splitting by newlines
      ↓ Still too big?
   Split by characters
   ```

   **Why this approach:** it's language-aware enough to *prefer* class/def boundaries,
   it's simple, fast, and has zero dependency on parsing the code correctly.

   | Pros | Cons |
   | --- | --- |
   | Simple, fast, works on any text or broken/partial code | A fixed `chunk_size` still cuts a long function in half once `def` boundaries don't fit |
   | Language-aware separators reduce mid-statement splits | No guarantee a chunk = one logical unit; `chunk_overlap` papers over this by duplicating text |
   | No AST/parse step, so nothing to fail on | Loses structural metadata (which class? which function?) |

   **Limitation that drives us forward:** "prefer to split on `def`" is a *hint*, not a
   guarantee. A function longer than `chunk_size` still gets sliced. We want chunks that
   are guaranteed to be whole logical units → **AST-based chunking (Day-2)**.

---

## 4. Tracing in LangSmith

   - https://smith.langchain.com/

   Tips:
   - create workspace and project in langsmith.
   - add params (tracing, endpoint, api_key, project_name) to `.env` file.

   **Why:** RAG is multi-step (retrieve → reason → answer). When an answer is wrong you
   need to see *what was retrieved* to know whether the bug is in chunking, retrieval,
   or the model. Tracing makes each step inspectable instead of a black box.

---

## 5. Limiting model and tool calls of the agent

   - Use middlewares to cap how many times the agent calls the model and tools.
   - Live example: `build_agent()` in [rag/4_BM25_demo.py](../rag/4_BM25_demo.py) wires
     `ModelCallLimitMiddleware(run_limit=5)` and
     `ToolCallLimitMiddleware(tool_name="search_codebase", run_limit=2)` into
     `create_agent`.

   **Why:** an agent can loop — re-searching and re-reasoning indefinitely — which burns
   tokens and money. Call limits are a guardrail that forces it to answer with what it
   has after a bounded number of steps.

   - References:
    - https://reference.langchain.com/python/langchain/agents/middleware/model_call_limit/ModelCallLimitMiddleware
    - https://reference.langchain.com/python/langchain/agents/middleware/tool_call_limit/ToolCallLimitMiddleware
    - Other middlewares: https://reference.langchain.com/python/langchain/agents/middleware/
