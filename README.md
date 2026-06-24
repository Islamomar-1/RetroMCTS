# RetroMCTS: Retrosynthesis Planning with Monte Carlo Tree Search

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.9%2B-blue?logo=python&logoColor=white" alt="Python 3.9+">
  <img src="https://img.shields.io/badge/PyTorch-2.0%2B-EE4C2C?logo=pytorch&logoColor=white" alt="PyTorch">
  <img src="https://img.shields.io/badge/RDKit-2023%2B-007ACC" alt="RDKit">
  <img src="https://img.shields.io/badge/NetworkX-3.1%2B-green" alt="NetworkX">
  <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="MIT License">
  <img src="https://img.shields.io/badge/ICLR-2022-orange" alt="ICLR 2022">
  <img src="https://img.shields.io/badge/tests-pytest-success" alt="Tests">
</p>

<p align="center">
  A clean, modular PyTorch implementation of <strong>Monte Carlo Tree Search</strong>
  for computer-aided retrosynthetic planning, inspired by
  <a href="https://openreview.net/forum?id=FRplAwWhIO">Gao, Mercado & Coley (ICLR 2022)</a>.
</p>

---

## Motivation

Discovering viable synthetic routes to complex drug-like molecules is one of the
most time-consuming bottlenecks in medicinal chemistry. Computer-aided retrosynthesis
aims to automate this process by working backwards from a target molecule and
proposing sequences of known reactions that yield commercially available starting
materials.

**RetroMCTS** combines two powerful paradigms:

1. **One-step template prediction** — a neural network (Morgan fingerprint encoder + MLP)
   scores a library of reaction templates for any given molecule.
2. **Monte Carlo Tree Search (MCTS)** — an UCB1-guided tree search explores the
   combinatorial space of multi-step synthetic routes efficiently, balancing
   exploitation of high-scoring pathways with exploration of novel ones.

Together they provide a principled, scalable alternative to brute-force exhaustive
search, with a clear separation between the *model* (what reactions are plausible?)
and the *planner* (how do we search for a complete route?).

---

## Features

- **UCB1-guided MCTS** with configurable exploration constant, depth, and iteration budget
- **One-step retrosynthesis model** based on Morgan (ECFP4) fingerprints and a
  lightweight MLP template classifier
- **10 built-in reaction templates** (Suzuki, amide bond formation, ester hydrolysis,
  Diels-Alder, Wittig, Buchwald-Hartwig, …)
- **Purchasability oracle** backed by a mock building-block catalogue; easily swappable
  with real vendor APIs (Enamine, Sigma-Aldrich, Reaxys)
- **NetworkX integration** — export search trees as directed graphs for downstream
  visualisation and analysis
- **Rich CLI** for single-target planning (`plan.py`) and batch benchmarking (`evaluate.py`)
- **Full test suite** with `pytest`, covering MCTS phases, UCB1 scoring,
  purchasability lookup, and chemistry utilities
- **Modular architecture** — swap the model, oracle, or search algorithm independently

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/islamomar/RetroMCTS.git
cd RetroMCTS
```

### 2. Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

### 3. Install RDKit

RDKit is best installed via conda:

```bash
conda install -c conda-forge rdkit
```

Or via pip (Python ≥ 3.9):

```bash
pip install rdkit
```

### 4. Install RetroMCTS

```bash
pip install -e ".[dev]"
# or simply:
pip install -r requirements.txt
pip install -e .
```

---

## Quick Start

### Plan a single molecule

```bash
python plan.py --smiles "CC(=O)Oc1ccccc1C(=O)O"   # Aspirin
```

Example output:

```
╭──────────────────────────────────────────────────╮
│  RetroMCTS  ·  1 target(s)  ·  max_iter=200  depth=6  top_k=5  │
╰──────────────────────────────────────────────────╯
Device: cpu
Building-block catalogue: 80 molecules

── CC(=O)Oc1ccccc1C(=O)O ──────────────────────────
✔ SOLVED  |  43 iterations  |  0.31s  |  1 route(s)

╭─────────────────────────────── Route 1 ───────────────────────────────╮
│ Step │ Target              │ Precursors              │ Template │ Score │
│  1   │ CC(=O)Oc1ccccc1C=O │ c1ccccc1C(=O)O + CC=O  │ T001     │ 0.312 │
╰───────────────────────────────────────────────────────────────────────╯
```

### Plan from a file

```bash
python plan.py --input data/raw/test_targets.smi --output results.json --iterations 500
```

### Benchmark on a dataset

```bash
python evaluate.py --dataset data/raw/test_targets.smi --iterations 300 --output eval.json
```

### Use the Python API

```python
from retromcts import MCTS, TemplateModel, PurchasabilityOracle

# Initialise components
model  = TemplateModel(device="cpu")
oracle = PurchasabilityOracle()
mcts   = MCTS(model=model, oracle=oracle, max_iterations=300, max_depth=6)

# Search
result = mcts.search("CC(=O)Oc1ccccc1C(=O)O")

print(f"Solved: {result.solved}")
print(f"Routes found: {len(result.routes)}")
print(f"Iterations: {result.iterations}")
print(f"Time: {result.elapsed_seconds:.2f}s")

# Inspect the first route
for step in result.routes[0]:
    print(step["target"], "→", " + ".join(step["precursors"]))

# Export the search tree as a NetworkX graph
graph = mcts.to_networkx(result.root_node)
print(f"Nodes: {graph.number_of_nodes()}, Edges: {graph.number_of_edges()}")
```

### Run tests

```bash
pytest tests/ -v --tb=short
# With coverage:
pytest tests/ --cov=retromcts --cov-report=term-missing
```

---

## Technology Stack

| Component | Library | Role |
|---|---|---|
| Deep learning | **PyTorch 2.0+** | Template-scoring MLP |
| Chemistry | **RDKit 2023+** | Fingerprints, reaction SMARTS |
| Graph search | **NetworkX 3.1+** | Search-tree representation |
| CLI | **Click + Rich** | Colourful terminal interface |
| Testing | **pytest** | Unit & integration tests |
| Logging | **loguru** | Structured logging |

---

## Repository Structure

```
RetroMCTS/
├── retromcts/                  # Core library
│   ├── __init__.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── fingerprint.py      # MorganFingerprintEncoder (MLP)
│   │   └── template_model.py   # TemplateModel: predict & apply templates
│   ├── search/
│   │   ├── __init__.py
│   │   └── mcts.py             # MCTSNode, MCTS, SearchResult, UCB1
│   └── utils/
│       ├── __init__.py
│       ├── chemistry.py        # RDKit helpers (canonicalise, MW, …)
│       └── purchasability.py   # PurchasabilityOracle + mock catalogue
│
├── plan.py                     # CLI: plan synthetic routes
├── evaluate.py                 # CLI: benchmark on a dataset
│
├── data/
│   ├── raw/
│   │   └── test_targets.smi    # Demo benchmark targets
│   ├── processed/              # Pre-processed intermediate data
│   └── building_blocks/
│       └── mock_catalogue.json # Mock purchasable building blocks
│
├── models/
│   ├── checkpoints/            # Training checkpoints (.pt)
│   └── pretrained/             # Released model weights
│
├── notebooks/                  # Jupyter exploration notebooks
├── tests/
│   ├── __init__.py
│   ├── test_mcts.py            # MCTS unit & integration tests
│   └── test_chemistry.py       # Chemistry utility tests
│
├── requirements.txt
├── setup.py
├── pyproject.toml
├── .gitignore
└── LICENSE
```

---

## How It Works

### MCTS overview

```
                        Target molecule
                             │
                    ┌────────┴────────┐
                    │    Selection    │  UCB1 tree policy
                    └────────┬────────┘
                             │ (leaf node)
                    ┌────────┴────────┐
                    │   Expansion     │  Apply top-K templates via RDKit
                    └────────┬────────┘
                             │ (child nodes = precursor sets)
                    ┌────────┴────────┐
                    │   Evaluation    │  Fraction of purchasable precursors
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │ Backpropagation │  Update Q and N up the tree
                    └─────────────────┘
```

**UCB1 formula** used for child selection:

```
UCB1(v) = Q(v)/N(v)  +  c · √( ln N(parent) / N(v) )
           ──────────────   ─────────────────────────
            Exploitation          Exploration
```

where `c = √2` by default.

A node is **solved** when all precursors in at least one template application
are either purchasable or recursively solved. This is an AND/OR tree:
- **OR** over different template choices for a given molecule
- **AND** over precursors within a single template application

### Template model

```
SMILES → Morgan FP (radius=2, 2048 bits)
       → Linear(2048 → 512) → BN → ReLU → Dropout
       → Linear(512  → 512) → BN → ReLU → Dropout
       → Linear(512  → 256)          [encoder]
       → Linear(256  → 128) → ReLU → Dropout
       → Linear(128  → |T|)          [classifier]
       → log-softmax → top-K templates
```

---

## Benchmark

Results on the built-in 8-molecule demo set (random model weights; train a real
model for meaningful numbers):

| Metric | Value |
|---|---|
| Targets | 8 |
| Success Rate | — (train weights for real results) |
| Avg Iterations | ≤ 200 |
| Avg Time / Target | < 1 s (CPU) |
| Max Depth | 6 |

> **Tip:** To reproduce results from Gao et al. (ICLR 2022), train the
> `TemplateModel` on the USPTO-50K dataset and replace the mock building-block
> catalogue with the Enamine catalogue (~17 M molecules).

---

## Contributing

Contributions are warmly welcomed! Here is the workflow:

1. **Fork** the repository and create a feature branch:
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Write code** following the existing style (Black + isort):
   ```bash
   black retromcts/ tests/
   isort retromcts/ tests/
   flake8 retromcts/ tests/
   ```

3. **Add tests** for new functionality:
   ```bash
   pytest tests/ -v
   ```

4. **Open a Pull Request** with a clear description of what you changed and why.

### Areas for contribution

- Training pipeline for the template model on USPTO datasets
- Integration with real vendor catalogues (Enamine, Sigma-Aldrich)
- Alternative search algorithms (A\*, beam search, Retro\*)
- Reaction condition prediction
- Synthesizability scoring (SA score, SCScore)
- Interactive visualisation of synthesis trees

---

## Citation

If you use RetroMCTS in your research, please cite the original paper that
inspired this implementation:

```bibtex
@inproceedings{gao2022amortized,
  title     = {Amortized Tree Generation for Bottom-up Synthesis Planning
               and Synthesizability Assessment},
  author    = {Gao, Wenhao and Mercado, Rocío and Coley, Connor W.},
  booktitle = {International Conference on Learning Representations (ICLR)},
  year      = {2022},
  url       = {https://openreview.net/forum?id=FRplAwWhIO}
}
```

And this repository:

```bibtex
@software{omar2026retromcts,
  author  = {Omar, Islam},
  title   = {{RetroMCTS}: Retrosynthesis Planning with Monte Carlo Tree Search},
  year    = {2026},
  url     = {https://github.com/islamomar/RetroMCTS},
  license = {MIT}
}
```

---

## Related Work

- **Retro\*** — Segler et al., *Nature* 2018 — Deep learning + best-first search
- **MCTS retrosynthesis** — Gao et al., ICLR 2022 — Amortised MCTS
- **GLN** — Shi et al., NeurIPS 2020 — Graph-based template prediction
- **LocalRetro** — Chen & Jung, ACS Central Science 2021 — Local reaction template approach

---


# RetroMCTS
