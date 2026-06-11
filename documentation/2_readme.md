# Day-2 Learnings

> Day-1 ended on a problem: `RecursiveCharacterTextSplitter` only *prefers* to split on
> `def`/`class`, so a long function still gets cut in half. Day-2 fixes the **chunking**
> with AST, then fixes **retrieval** itself by adding lexical (BM25) search alongside
> semantic search.

## 1. AST (Abstract Syntax Tree) — better chunk boundaries

- **Use case:** parse code into its real structure instead of treating it as text.
- AST-Based Chunking Flow:
    ```
    Source Code → AST Tree → Find Classes/Functions → Extract Source Code → Create Chunks → Embed & Store
    ```
- AST API in Python — the `ast` module:
    - `ast.parse()`: parse Python source into an AST.
    - `ast.dump()`: string representation of the AST.
    - `ast.walk()`: traverse and yield every node in the tree.
    - `ast.iter_child_nodes()`: iterate the direct children of a node.
    - `ast.get_source_segment()`: get the exact source text for a node.

- Hands-on demo: [rag/2_ast_demo.py](../rag/2_ast_demo.py) prints the AST dump for an
  expression (`x = 1 + 2`), a function definition, and a class with methods.

   **Why this approach:** the parser knows where a function/class *actually* starts and
   ends, so we can cut on those boundaries instead of guessing with a character count.

---

## 2. Semantic RAG using AST

- Instead of chunking by characters or lines, walk the AST and emit **one chunk per
  logical unit**: a whole top-level class (methods included) or a module-level function.
- Each chunk is tagged with `type` (`class`/`function`) and `name` metadata, so
  retrieval results are self-describing.

Implementation: [rag/3_semanticrag_ast.py](../rag/3_semanticrag_ast.py)

   **Why we moved here from character chunking:** a chunk is now a complete, coherent
   unit, so its embedding represents one real idea — which makes semantic retrieval far
   more accurate.

   | Pros | Cons |
   | --- | --- |
   | Chunks are never split mid-function/mid-class | Python-only (uses the `ast` module); other languages need another parser |
   | Carries structure metadata (`type`, `name`) for free | A single function bigger than the embedding model's context still needs sub-splitting |
   | Cleaner embeddings → more relevant retrieval | Fails on syntactically broken code (demo falls back to the raw document) |

   **Limitation that drives us forward:** AST fixed *chunking*, but retrieval is still
   pure semantic similarity. Ask for the exact string `"Payment failed: insufficient
   funds"` or a function named `_v2_legacy_handler` and embeddings may not rank it first,
   because nothing else is semantically *near* it. We need exact-token matching too.

---

## 3. BM25 — lexical (exact-keyword) retrieval

- BM25 retrieves by **exact keyword matching** over a corpus.
- BM25 (Best Matching 25) = **Inverted Index + Ranking Algorithm**.
- BM25 score = Term Frequency + Word Rarity (IDF) − Length Penalty
    - **Term Frequency** — more occurrences of the query term ⇒ higher score.
    - **IDF (rarity)** — rare terms are more informative than common ones (`def` is
      everywhere; `process_payment` is not).
    - **Length penalty** — stops long chunks from winning just by containing more words.
- The demo builds the index with `rank_bm25`'s `BM25Okapi` over whitespace-tokenized
  chunks.

   **Why this approach:** it's the exact opposite failure mode of embeddings — it nails
   identifiers, error strings, and rare tokens that semantic search blurs over.

   | Pros | Cons |
   | --- | --- |
   | Exact match on identifiers, error strings, rare tokens | No notion of meaning — `charge customer` won't find `process_payment` |
   | No embedding model: cheap, fast, fully local | Sensitive to tokenization (`get_user` vs `getUser` vs `get user`) |
   | Transparent, debuggable scores | Zero recall for paraphrases / synonyms |

---

## 4. Lexical RAG using BM25 → why Hybrid

- Wraps `BM25Okapi` in a custom `BM25Retriever(BaseRetriever)`, exposes it as the
  `search_codebase` tool, and applies model/tool call-limit middleware to the agent.

Implementation: [rag/4_BM25_demo.py](../rag/4_BM25_demo.py) (BM25 index + lexical RAG agent)

- **Key takeaway: neither exact match alone nor semantic match alone is sufficient.**
  Semantic search wins on *intent*, BM25 wins on *exact tokens* — and real questions
  need both.

- **Hybrid Retrieval** (the approach we land on) combines them:
    - BM25 for exact-match retrieval,
    - vector DB for semantic retrieval,
    - optionally other sources (SQL, Elasticsearch, graph DB, …),
    - then **combine and re-rank** the candidates to feed the best context to the model.

   | Approach | Strength | Blind spot |
   | --- | --- | --- |
   | Semantic only | intent, paraphrase, synonyms | exact identifiers / rare strings |
   | Lexical (BM25) only | exact tokens, error messages | meaning, paraphrase |
   | **Hybrid (chosen)** | covers both via re-ranking | more moving parts to build & tune |

   **Why we go with Hybrid:** each method's blind spot is the other's strength. Combining
   and re-ranking gives the recall of semantic search *and* the precision of lexical
   search — the best context for the LLM to generate from.

---

## Summary — the progression and *why* at each hop

| Step | Approach | Fixed | Remaining gap → next step |
| --- | --- | --- | --- |
| 1 | Char chunking + semantic | Get RAG working at all | Chunks split mid-function |
| 2 | **AST** chunking + semantic | Whole-unit chunks, better embeddings | Misses exact tokens |
| 3 | **BM25** lexical | Exact-token matching | Misses meaning / paraphrase |
| 4 | **Hybrid** (semantic + lexical + re-rank) | Recall *and* precision | Tuning & infra complexity |

---

## AST vs RecursiveCharacterTextSplitter

| AST RAG | RecursiveCharacterTextSplitter |
| --- | --- |
| Parses code structure and chunks on logical units (class/function) | Recursively chunks by separators (class → def → blank lines → newlines → chars) |
| Chunks are guaranteed-whole logical units → cleaner embeddings | A long unit is still cut once it exceeds `chunk_size` |
| Carries `type`/`name` metadata | No structural metadata |
| Python-only; breaks on invalid syntax | Language-agnostic; works on any/broken text |

---

## TreeSitter

- TreeSitter is a parser generator tool and an incremental parsing library.
- Unlike Python's `ast` (Python-only), TreeSitter supports **many languages** and
  **incremental re-parsing** — the natural next step if AST chunking needs to scale to a
  multi-language codebase. _(No local demo yet.)_

- Reference: https://github.com/kreuzberg-dev/tree-sitter-language-pack
