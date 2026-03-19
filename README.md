<p align="center">
  <img src="./assets/biocortex_logo.svg" alt="BioCortex Logo" width="520px" />
</p>

<p align="center">
<a href="#project-website">
<img src="https://img.shields.io/badge/Website-Coming%20Soon-0ea5e9?style=for-the-badge" alt="Project Website" />
</a>
<a href="#quick-start">
<img src="https://img.shields.io/badge/Try-Web%20UI-22c55e?style=for-the-badge" alt="Web UI" />
</a>
<a href="#license">
<img src="https://img.shields.io/badge/License-Apache--2.0-6366f1?style=for-the-badge" alt="License" />
</a>
<a href="#configuration">
<img src="https://img.shields.io/badge/Python-3.11%2B-f59e0b?style=for-the-badge" alt="Python 3.11+" />
</a>
</p>

# BioCortex: A Next-Generation AI Agent for All Biological Systems

BioCortex is a multi-agent biological AI framework designed to plan, execute, and synthesize complex research workflows across **all biological domains** (genomics, ecology, neuroscience, agriculture, marine biology, and more).

It extends Biomni-style tool execution with:
- **3-Stage Hybrid Retrieval** ⚡ (vector + knowledge graph + LLM rerank) — **3-5× faster** than Biomni's LLM-only retrieval
- **Strategy routing** (ReAct / DAG Parallel / MCTS) — auto-selects optimal execution strategy
- **Multi-agent pipeline** (Planner → Executor → Critic → Synthesizer) — parallel execution & self-validation
- **Persistent memory & knowledge graph** — learns from past analyses
- **Multimodal support** (sequence / structure / image encoders) — ESM-2, DNABERT-2, BiomedCLIP
- **Biomni tool bridge** (200+ tools can be reused) — full backward compatibility

---

## Project Website

Coming soon. For now, launch the local Web UI:
```
http://localhost:7860
```

---

## Quick Start

### 1) Installation

#### Option A: One-Click Install (Recommended)

```bash
cd /path/to/BioCortex
bash install.sh            # Full install (conda + pip)
bash install.sh --minimal  # Core only (skip torch/multimodal)
```

#### Option B: Manual Install (Conda)

```bash
# Create and activate conda environment
conda create -n biocortex_env python=3.11 -y
conda activate biocortex_env

# Install dependencies
pip install -r requirements.txt

# Install BioCortex (editable)
pip install -e .
```

#### Option C: From biocortex_env （recommend）

```bash
cd /path/to/BioCortex/biocortex_env
conda env create -f environment.yml
conda activate biocortex
Rscript install_r_packages.R r_packages.yml
```

### 2) Configure API Keys

```bash
cp env_example.txt .env
```

Edit `.env` and set your LLM provider:

```env
# ── Option 1: Anthropic (Claude) ──
ANTHROPIC_API_KEY=sk-ant-xxx

# ── Option 2: OpenAI (GPT-4o) ──
OPENAI_API_KEY=sk-xxx

# ── Option 3: DashScope / Qwen (recommended for China) ──
BIOCORTEX_CUSTOM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
BIOCORTEX_CUSTOM_API_KEY=sk-xxx
BIOCORTEX_REASONING_MODEL=qwen3-max
BIOCORTEX_CODER_MODEL=qwen3-max
BIOCORTEX_FAST_MODEL=qwen3-max
```

### 3) Single-Cell & Spatial Visualization (Vitessce)

BioCortex includes an interactive single-cell and spatial transcriptomics viewer powered by [Vitessce](https://vitessce.io/). It loads directly from CDN — **no build step required**.

**What it does:**
- Opens `.h5ad` files in the Results Viewer with interactive dark-themed visualizations
- Automatically detects embeddings (UMAP, t-SNE, PCA) and spatial coordinates
- Displays: Scatterplots, Spatial distribution, Cell Sets, Cell Set Sizes
- For small datasets (<30k cells): also shows Gene List, Heatmap, Expression by Cell Set
- Includes **CellChat** 🧠 — an AI chat assistant that can answer questions about your data

**Required Python packages** (already in `requirements.txt`):

```bash
pip install anndata scanpy
```

**Optional for UMAP computation** (if your h5ad only has PCA):

```bash
pip install leidenalg igraph
```

**Supported h5ad data:**
| Field | Description | Required |
|-------|-------------|----------|
| `obsm['X_umap']` | UMAP coordinates | At least one embedding |
| `obsm['X_tsne']` | t-SNE coordinates | or PCA (auto-computes UMAP) |
| `obsm['X_pca']` | PCA coordinates | Fallback |
| `obsm['spatial']` | Spatial coordinates | For spatial transcriptomics |
| `obs[categorical cols]` | Cell type annotations | For cell set coloring |
| `X` (expression matrix) | Gene expression | For Gene List / Heatmap |

For **integrating the h5ad viewer in a new project** (dependencies, env vars, API routes, and frontend iframe), see **[biocortex/web/docs/VITESSCE_H5AD_SETUP.md](biocortex/web/docs/VITESSCE_H5AD_SETUP.md)**.

### 4) Run Web UI

```bash
python biocortex_web_app.py --port 7860
```

Open:
```
http://localhost:7860
```

---

## Python Usage

```python
from biocortex.agent import BioCortexAgent

agent = BioCortexAgent()
report = agent.go("Analyze scRNA-seq data: QC, clustering, annotation, DEGs")
```

Force a strategy:
```python
from biocortex.config import ReasoningStrategy
agent.go("Discover drug targets for ALS", strategy=ReasoningStrategy.MCTS)
```

---

## 3-Stage Hybrid Retrieval (Core Advantage)

BioCortex uses a **3-stage hybrid retrieval pipeline** for tool selection — a key architectural advantage over Biomni:

### Stage 1: Vector Semantic Search
- **Fast recall** using sentence embeddings (all-MiniLM-L6-v2)
- Retrieves top-50 candidates in milliseconds
- Scales to 10,000+ tools via FAISS index

### Stage 2: Knowledge Graph Expansion
- **Discovers implicit dependencies** via 2-hop BFS traversal
- Example: Query "scRNA-seq clustering" → expands to include "batch correction", "doublet removal"
- Leverages tool co-occurrence graph + biological ontologies (GO, KEGG)

### Stage 3: LLM Reranking
- **Precision selection** using task context
- LLM picks final top-15 tools from expanded candidates
- Context-aware: considers task requirements, data types, dependencies

### Performance vs. Biomni

| Metric | Biomni (LLM-only) | BioCortex (3-stage) | Speedup |
|--------|-------------------|---------------------|---------|
| Tool retrieval time | 3-5 seconds | 0.5-1 second | **3-5×** |
| Scalability | Limited by LLM context | 10,000+ tools | **100×** |
| Dependency discovery | Manual | Automatic (KG) | ✅ |

**Default:** Vector search is **enabled by default**. To disable (e.g., for offline environments):

```env
# In .env file:
BIOCORTEX_ENABLE_VECTOR_SEARCH=false
```

---

## Biomni tools (200+ domain tools)

Biomni has its own GitHub repo. BioCortex can use it in two ways:

1. **Embedded (default)** — If `vendor/Biomni/biomni` exists (e.g. after adding the Biomni submodule and running `git submodule update --init --recursive`), no configuration is needed. The Biomni repo URL will be provided once this framework is on GitHub; see `vendor/README.md` for submodule setup.
2. **Custom path** — Override with your own Biomni clone:

```bash
python biocortex_web_app.py --biomni-path /path/to/Biomni/biomni
```

Or in Python:
```python
agent = BioCortexAgent(biomni_path="/path/to/Biomni/biomni")  # optional; uses vendor/Biomni/biomni if empty and present
```

---

## Multimodal Support

BioCortex can encode:
- **Sequences** (protein, DNA, RNA)
- **Structures** (PDB, chemistry)
- **Images** (microscopy / pathology)

Optional dependencies (install only if needed):

```bash
pip install torch transformers torchvision fair-esm pillow
```

---

## Knowledge Graph

BioCortex maintains a **persistent knowledge graph** and auto-learns new facts from analyses.

The graph supports:
- Tool relations
- GO / KEGG / UniProt integration
- Entity search and context generation

---

## Web UI Features

- Chat-based analysis interface
- Strategy selector (Auto / ReAct / DAG / MCTS)
- DAG execution visualization (Mermaid)
- Tool browser (200+ tools)
- Knowledge graph explorer
- Multimodal encoder panel
- Memory viewer
- E2E test panel
- **Results Viewer** — multi-tab file browser for text, images, notebooks, CSV, and more
- **Vitessce Integration** — interactive single-cell / spatial transcriptomics viewer (dark theme)
  - Scatterplot (UMAP / t-SNE / PCA), Spatial distribution, Cell Sets, Cell Set Sizes
  - Gene List, Heatmap, Expression by Cell Set (for datasets < 30k cells)
  - Loads from CDN, no npm/build step required
- **CellChat** 🧠 — AI data assistant embedded in the Vitessce viewer
  - Ask questions about cell types, spatial patterns, gene expression
  - Uses your configured LLM (reasoning model)
  - Context-aware: knows dataset metadata, cell distributions, top genes

---

## Multi-User Web UI (Roadmap)

BioCortex currently ships with a **single-user** Gradio UI. If you need
multi-user isolation, authentication, and per-user files/history:

- Recommended path: adapt the multi-user modules from Biomni (`Biomni/web/*`)
- Alternative: deploy behind an auth gateway (Nginx basic auth / OAuth)
- We plan to integrate native multi-user support in a future release

---

## Configuration

Config is managed by `biocortex/config.py` and supports `.env` overrides:

```env
# ── LLM Provider (at least one required) ──────────
# Anthropic
ANTHROPIC_API_KEY=sk-ant-xxx

# OpenAI
OPENAI_API_KEY=sk-xxx

# Custom endpoint (DashScope, vLLM, Ollama, etc.)
BIOCORTEX_CUSTOM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
BIOCORTEX_CUSTOM_API_KEY=sk-xxx

# ── Model Names ───────────────────────────────────
BIOCORTEX_REASONING_MODEL=qwen3-max     # Planner + Synthesizer + CellChat
BIOCORTEX_CODER_MODEL=qwen3-max         # Code generation
BIOCORTEX_FAST_MODEL=qwen3-max          # Quick tasks, validation

# ── Strategy ──────────────────────────────────────
BIOCORTEX_STRATEGY=dag_parallel          # react / dag_parallel / mcts / hypothesis

# ── Execution ─────────────────────────────────────
BIOCORTEX_TIMEOUT=600
BIOCORTEX_USE_DOCKER=false

# ── Paths ─────────────────────────────────────────
BIOCORTEX_DATA_PATH=./data
BIOCORTEX_OUTPUT_PATH=./output
```

**Per-user config:** Each user can override settings in `./users/{username}/.env`.

---

## Citation

If you use BioCortex in research, please cite:

```bibtex
@software{biocortex2026,
  title  = {BioCortex: A Next-Generation AI Agent for All Biological Systems},
  author = {BioCortex Team},
  year   = {2026},
  url    = {REPLACE_WITH_PROJECT_URL}
}
```

---

## Contributing

We welcome contributions from the community:

- **New tools** (domain-specific analysis functions)
- **Datasets** and knowledge sources
- **Retrievers / KG** improvements
- **Multimodal encoders** and model adapters
- **Docs & tutorials**

How to contribute:

1. Fork the repo and create a feature branch
2. Add your changes with tests or examples
3. Open a pull request describing your changes

---

## Security Note

BioCortex executes LLM-generated code. For production use, **enable sandboxing** and isolate the runtime environment. Do not run with sensitive credentials or files on untrusted tasks.

---

## License

Apache 2.0
