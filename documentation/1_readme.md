### Day-1 Learnings:

1. RAG (Retrieval Augmented Generation):
`Docs -> Chunk -> Embed -> Store -> Retrieve -> Generate`

---
2. Semantic RAG:
   - Uses vector databases to store and retrieve relevant information based on cosine similarity.
   - This demo loads `sample_project/`, chunks each `.py` file with `RecursiveCharacterTextSplitter`, embeds chunks with `all-MiniLM-L6-v2` (HuggingFace), stores them in Chroma, and exposes a `search_codebase` retriever tool to a `create_agent` agent.

   Semantic RAG Implementation: [rag/1_semantic_rag.py](../rag/1_semantic_rag.py)
---
3. RecursiveCharacterTextSplitter (Python)

- Refer to [RecursiveCharacterTextSplitter](https://python.langchain.com/en/latest/modules/indexes/text_splitters/examples/recursive_character_text_splitter.html) for more details.

- Flow:
   ```
   Try splitting by "class"
      ↓
   Still too big?
      ↓
   Try splitting by "def"
      ↓
   Still too big?
      ↓
   Try splitting by blank lines
      ↓
   Still too big?
      ↓
   Try splitting by newlines
      ↓
   Still too big?
      ↓
   Split by characters
   
   ```

---
4. Tracing in LangSmith:
   - https://smith.langchain.com/

   Tips:
   - create workspace and project in langsmith.
   - add params(tracing, endpoint, api_key, project_name) to .env file

---
5. Limiting model and tool calls of the agent:
   - Use middlewares to limit the number of calls to the model and tools.
   - Live example: `build_agent()` in [rag/4_BM25_demo.py](../rag/4_BM25_demo.py) wires `ModelCallLimitMiddleware(run_limit=5)` and `ToolCallLimitMiddleware(tool_name="search_codebase", run_limit=2)` into `create_agent`.

   - References:
    - https://reference.langchain.com/python/langchain/agents/middleware/model_call_limit/ModelCallLimitMiddleware

    - https://reference.langchain.com/python/langchain/agents/middleware/tool_call_limit/ToolCallLimitMiddleware

    - Other middlewares: https://reference.langchain.com/python/langchain/agents/middleware/