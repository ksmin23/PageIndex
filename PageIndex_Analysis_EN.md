# PageIndex Codebase Analysis Report

---

## 1. Architecture Overview

PageIndex is a **vectorless**, **reasoning-based RAG** framework that retrieves information from documents using LLM reasoning instead of vector similarity search. The core idea is to transform documents into a **hierarchical tree structure** (similar to a table of contents) and let LLMs navigate this tree to find relevant information.

### Project Structure

```
PageIndex/
├── run_pageindex.py              # CLI entry point
├── requirements.txt              # Dependencies (litellm, pymupdf, PyPDF2, etc.)
├── pageindex/
│   ├── __init__.py               # Exports page_index and page_index_md modules
│   ├── config.yaml               # Default settings (model, parameters)
│   ├── page_index.py             # Core: PDF → tree structure generation pipeline
│   ├── page_index_md.py          # Markdown → tree structure generation
│   └── utils.py                  # LLM calls, PDF parsing, utility functions
└── cookbook/
    ├── pageindex_RAG_simple.ipynb # Basic RAG example
    ├── agentic_retrieval.ipynb    # Agentic retrieval example
    └── vision_RAG_pageindex.ipynb # Vision-based RAG example
```

### Module Relationships

```
run_pageindex.py
    ├── pageindex.page_index.page_index_main()   ← PDF processing
    └── pageindex.page_index_md.md_to_tree()     ← Markdown processing
            │
            └── pageindex.utils
                    ├── llm_completion() / llm_acompletion()  ← LiteLLM-based LLM calls
                    ├── get_page_tokens()                     ← PDF text extraction
                    ├── ConfigLoader                          ← Configuration management
                    └── Various tree manipulation / JSON utilities
```

---

## 2. How Documents Are Structured

PageIndex transforms PDF/Markdown documents into a **hierarchical tree structure**. The key differentiator is that it uses **no vector embeddings or chunking** — it leverages the document's natural section structure.

### 2.1 PDF Structuring Pipeline (`pageindex/page_index.py`)

The overall flow proceeds as `page_index_main()` → `tree_parser()` → `meta_processor()`:

**Step 1: PDF Text Extraction** (`utils.py:382-405`)
```
PDF → Extract per-page text + token counts using PyPDF2/PyMuPDF
```

**Step 2: Table of Contents (TOC) Detection** (`page_index.py:341-366`)
- Send the first N pages (default: 20) to the LLM to determine: "Does this page contain a table of contents?"
- Three branching paths:
  - **TOC found + page numbers present** → `process_toc_with_page_numbers`
  - **TOC found + no page numbers** → `process_toc_no_page_numbers`
  - **No TOC found** → `process_no_toc` (LLM generates structure directly)

**Step 3: Tree Structure Generation** (`page_index.py:958-997`)
- With TOC: LLM converts the TOC into JSON format and matches each section to its physical page index
- Without TOC: Document is split into token-based groups, and the LLM generates the hierarchical structure directly

**Step 4: Verification and Correction** (`page_index.py:900-952`)
- For each node in the generated tree, the LLM asynchronously verifies: "Does this title actually appear on this page?"
- If accuracy > 60%, only incorrect items are fixed; if < 60%, the system falls back to an alternative processing mode

**Step 5: Recursive Large Node Splitting** (`page_index.py:1000-1027`)
- Nodes exceeding `max_page_num_each_node` (default: 10) AND `max_token_num_each_node` (default: 20,000) are recursively subdivided

**Final Tree Output Structure:**
```json
{
  "doc_name": "document.pdf",
  "structure": [
    {
      "title": "Section Title",
      "node_id": "0001",
      "start_index": 1,
      "end_index": 5,
      "summary": "Section summary",
      "nodes": [...]
    }
  ]
}
```

### 2.2 Markdown Structuring Pipeline (`pageindex/page_index_md.py`)

For Markdown files, basic structuring is possible **without any LLM calls**:

1. Parse heading levels (`#`, `##`, `###`, etc.) to extract hierarchy directly (no LLM needed)
2. Extract text content for each node
3. (Optional) **Tree Thinning**: Merge nodes below a token threshold into their parent
4. (Optional) LLM is only called when summary generation is requested

---

## 3. How User Prompts Are Answered

**Important: This open-source repository only handles "tree structure generation." The question-answering (Q&A) workflow must be implemented separately.**

The full RAG pipeline as demonstrated in `cookbook/pageindex_RAG_simple.ipynb`:

### Step 1: Tree Structure Generation (This Repository's Role)
```
PDF → PageIndex → Hierarchical tree (each node has title, page range, summary)
```

### Step 2: Tree Search (Reasoning-based Retrieval)
Provide the tree structure + question to an LLM, which **selects relevant nodes through reasoning**:
```python
search_prompt = f"""
You are given a question and a tree structure of a document.
Each node contains a node id, node title, and a corresponding summary.
Your task is to find all nodes that are likely to contain the answer to the question.
...
"""
# LLM returns a list of relevant node_ids
```

### Step 3: Context Extraction
Extract the `text` (actual page content) from the selected nodes.

### Step 4: Answer Generation
Provide the extracted context + question to the LLM to generate the final answer.

> Key differentiator: Instead of vector similarity, PageIndex uses LLM **reasoning** to find relevant sections — the same way a human expert navigates a table of contents to locate needed information.

---

## 4. How to Use Without a PageIndex API Key

This repository supports **fully self-hosted operation**. No PageIndex API Key is required.

### What You Need: Only an LLM API Key

```bash
# 1. Install dependencies
pip3 install --upgrade -r requirements.txt

# 2. Set your LLM API key in a .env file
echo "CHATGPT_API_KEY=sk-your-openai-key" > .env
# Or set a different provider's key (see Section 6 below)

# 3. Run
python3 run_pageindex.py --pdf_path /path/to/document.pdf
```

### Direct Python API Usage:
```python
from pageindex import page_index

result = page_index(
    doc="path/to/document.pdf",
    model="gpt-4o-2024-11-20",
    if_add_node_summary="yes"
)
```

> **PageIndex API Key** (`PAGEINDEX_API_KEY`) is only required when using Vectify's **cloud services** (Chat Platform, MCP, hosted API). When running this GitHub repository's code directly, you only need an LLM provider's API key.

---

## 5. Where OpenAI Models Are Required vs. Not Required

### Parts That **Require an LLM** (PDF Processing)

| Feature | Location | Description |
|---------|----------|-------------|
| TOC Detection | `page_index.py:104-122` | LLM determines if each page contains a TOC |
| TOC Transformation | `page_index.py:273-336` | Converts raw TOC into JSON structure |
| Tree Structure Generation | `page_index.py:542-574` | LLM directly generates structure for TOC-less documents |
| Page Index Mapping | `page_index.py:243-269` | Matches section titles to physical pages |
| Verification/Correction | `page_index.py:13-45` | Verifies that generated mappings are correct |
| Node Summary Generation | `utils.py:573-581` | Summarizes each node's text (optional) |
| Document Description Generation | `utils.py:617-626` | Generates overall document description (optional) |

### Parts That **Do NOT Require an LLM**

| Feature | Location | Description |
|---------|----------|-------------|
| PDF Text Extraction | `utils.py:216-241` | Handled directly by PyPDF2/PyMuPDF |
| Token Counting | `utils.py:25-28` | Uses LiteLLM's `token_counter` (local processing) |
| Tree Data Structure Operations | `utils.py:127-214` | write_node_id, get_nodes, list_to_tree, etc. |
| Markdown Parsing | `page_index_md.py:32-87` | Heading-based structure extraction (regex) |
| Tree Thinning | `page_index_md.py:135-187` | Token-based node merging (local processing) |
| Configuration Loading | `utils.py:649-681` | YAML file reading |
| Result Saving | `run_pageindex.py:72-80` | JSON file writing |

> **Key Point**: OpenAI models are NOT mandatory. Through LiteLLM, **any LLM** can be used (see next section).

---

## 6. How to Use Different AI Models

PageIndex has integrated **LiteLLM for multi-provider support**.

### LiteLLM Integration (`utils.py:1-23`)

```python
import litellm

# All LLM calls use litellm.completion() / litellm.acompletion()
def llm_completion(model, prompt, ...):
    response = litellm.completion(model=model, messages=messages, temperature=0)
    ...

async def llm_acompletion(model, prompt):
    response = await litellm.acompletion(model=model, messages=messages, temperature=0)
    ...
```

### How to Use Different Models

**Method 1: CLI Parameter**
```bash
# Anthropic Claude
export ANTHROPIC_API_KEY=sk-ant-your-key
python3 run_pageindex.py --pdf_path doc.pdf --model anthropic/claude-sonnet-4-20250514

# Google Gemini
export GEMINI_API_KEY=your-key
python3 run_pageindex.py --pdf_path doc.pdf --model gemini/gemini-2.5-pro

# Azure OpenAI
export AZURE_API_KEY=your-key
export AZURE_API_BASE=https://your-resource.openai.azure.com
python3 run_pageindex.py --pdf_path doc.pdf --model azure/gpt-4o
```

**Method 2: Edit `config.yaml`**
```yaml
# config.yaml already includes a commented-out Claude example
# model: "gpt-4o-2024-11-20"
model: "anthropic/claude-sonnet-4-6"   # Change to this
```

**Method 3: Specify in Python API**
```python
from pageindex import page_index

result = page_index(
    doc="document.pdf",
    model="anthropic/claude-sonnet-4-20250514"
)
```

### Major Providers Supported by LiteLLM

| Provider | Model Format | Environment Variable |
|----------|-------------|---------------------|
| OpenAI | `gpt-4o`, `gpt-4.1` | `OPENAI_API_KEY` |
| Anthropic | `anthropic/claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |
| Google | `gemini/gemini-2.5-pro` | `GEMINI_API_KEY` |
| Azure | `azure/gpt-4o` | `AZURE_API_KEY` + `AZURE_API_BASE` |
| AWS Bedrock | `bedrock/anthropic.claude-v2` | AWS credentials |
| Ollama (local) | `ollama/llama3` | No key required |
| Together AI | `together_ai/...` | `TOGETHERAI_API_KEY` |

> **Note**: In `utils.py:19-21`, there is backward-compatibility code that automatically maps `CHATGPT_API_KEY` to `OPENAI_API_KEY`. When using other providers, you only need to set the respective provider's environment variable.

---

## Summary

| Topic | Description |
|-------|-------------|
| **Core Philosophy** | Retrieval via LLM reasoning instead of vector similarity (Similarity ≠ Relevance) |
| **Tree Generation** | LLM identifies document TOC/structure → Hierarchical JSON tree |
| **Question Answering** | Provide tree structure to LLM → Reasoning-based node selection → Context-based answer |
| **API Key** | No PageIndex API Key needed (self-hosted); only LLM API key required |
| **Model Dependency** | Not locked to OpenAI; multi-provider support via LiteLLM |
| **LLM-Free Areas** | PDF parsing, Markdown structure extraction, token counting, tree data structure operations |
