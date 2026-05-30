# Nextflow & nf-core RAG Assistant

A Retrieval-Augmented Generation (RAG) Q&A web app that answers natural-language
questions about **Nextflow** and the **nf-core** bioinformatics-pipeline ecosystem.
It embeds a corpus of official documentation into a Chroma vector database, retrieves
the most relevant chunks for each question, and has `gpt-4o-mini` answer **using only
that retrieved context**. The interface is built with Gradio and includes an
interactive 3-D visualization of the embedding space.

**File to run for the Gradio app:** [`nextflow_rag.ipynb`](nextflow_rag.ipynb) — open
it, select the project kernel, and **Run All**. The final cell launches the Gradio app
and prints a local URL (e.g. `http://127.0.0.1:7860`).

---

## Authors & contributions

| Name | Contributions |
|------|---------------|
| _Your name here_ | _e.g. data sourcing, chunking strategy, retrieval & prompt design, Gradio UI_ |
| _Teammate (if any)_ | _e.g. embedding/indexing, 3-D visualization, evaluation/testing_ |

> _Fill in the names of everyone in your group (up to 2) and what each person did._

---

## What it does (pipeline)

1. **Document loading & preprocessing** — reads 11 Markdown docs from `data/`, strips
   frontmatter / MyST markup, and captures each document's title as metadata.
2. **Structure-aware chunking** — splits each document along its **Markdown headers**
   (tracking a `Section > Subsection` breadcrumb), then subdivides any oversized section
   to a ~500-token budget while keeping an **80-token overlap** between adjacent chunks so
   ideas that cross a boundary stay intact. (Not a blind split on token counts.)
3. **Embedding & indexing** — embeds chunks with OpenAI `text-embedding-3-small` and
   stores each chunk's **id + text + metadata** (`source`, `doc_title`, `section`,
   `chunk_idx`) in a persistent Chroma collection. Indexing is idempotent.
4. **Retrieval** — Chroma similarity search returns the top-k chunks (with distances).
5. **Safety check** — screens each query; unsafe requests are declined before retrieval.
6. **Generation with memory** — sends `[system prompt] + recent conversation +
   [retrieved context + question]` to `gpt-4o-mini`. The system prompt restricts answers
   to the provided context and to Nextflow/nf-core topics. Prior turns are included as
   **memory** so follow-up questions work.
7. **Gradio UI** — a chat box (with memory), a panel showing the retrieved **source
   chunks** + distances, and an interactive **3-D Plotly** plot of the embedding space
   that highlights the chunks retrieved for the current question (hover for a preview).

## Data

`data/` contains 11 open-licensed Markdown documents (~19 pages total), balanced between
core Nextflow and the nf-core community:

- **Nextflow** (Apache-2.0): overview, your first script, workflows, configuration
- **nf-core** (MIT): what is nf-core, running pipelines (overview + run-pipelines),
  run your first pipeline, configuration options, reference genomes, nf-core tools

Sources: the [Nextflow documentation](https://www.nextflow.io/docs/latest/) and the
[nf-core website](https://nf-co.re/docs).

---

## Setup

### 1. Create a virtual environment and install dependencies
```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

> `requirements.txt` is a tested, minimal set of exactly what this notebook needs.
> The full class-provided environment in `instructions/requirements.txt` also works —
> if you use that one instead, additionally run `pip install tiktoken` (used for
> token-based chunk sizing; it is the only extra package this project adds).

### 2. Add your API keys to a `.env` file in the project root
```env
OPENAI_API_KEY=sk-...            # required: embeddings (and generation by default)
OPENROUTER_API_KEY=sk-or-...     # optional: only if you set LLM_PROVIDER="openrouter"
```
> The OpenAI API is billed separately from a ChatGPT subscription — add a little credit at
> <https://platform.openai.com/settings/organization/billing>. Embedding this corpus costs
> about one cent.

By default the app calls `gpt-4o-mini` **directly through OpenAI** (`LLM_PROVIDER =
"openai"` in the config cell). To route generation through OpenRouter instead, set
`LLM_PROVIDER = "openrouter"` and provide a valid `OPENROUTER_API_KEY`.

### 3. Run the app
```bash
# Register the venv as a Jupyter kernel (once):
python -m ipykernel install --user --name llmsrag-venv --display-name "Python (LLMs_RAG .venv)"
```
Open `nextflow_rag.ipynb` in Jupyter / VS Code, choose the **Python (LLMs_RAG .venv)**
kernel, and **Run All**. The last cell launches the Gradio interface; click the printed
local URL.

## Example questions to try
- "What is a process in Nextflow and how do I define its inputs and outputs?"
- "How do I run an nf-core pipeline on my own data?"
- "What's the difference between an entry workflow and a named workflow?"
- "How do I configure the resources for an nf-core pipeline?" — then follow up with
  "Show that as a config file example." (tests memory)

## Notes
- **Re-indexing:** delete the `chroma_db/` folder to rebuild embeddings from scratch
  (e.g. after changing chunk size/overlap), then Run All again.
- **Project structure:**
  ```
  data/                 # source Markdown documents
  nextflow_rag.ipynb    # the RAG app (run this)
  requirements.txt      # dependencies
  .env                  # your API keys (not committed)
  chroma_db/            # persistent vector store (auto-created)
  ```
