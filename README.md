# QuantumGraphColoring

Reference implementation and benchmarking suite for **qubit-efficient and gate-efficient encodings of graph partitioning problems**, applied to *Minimum Graph Coloring* on the D-Wave *Advantage2* quantum annealer.

This repository accompanies the paper:

> T. Zaborniak, P. P. Angara, V. Mulligan, H. Müller, and U. Stege, *"Qubit-efficient and gate-efficient encodings of graph partitioning problems for quantum optimization."*

The three notebooks build, reduce, embed, sample, aggregate, and plot results for two competing encodings of Minimum Graph Coloring on the same set of random graphs, producing the timing, qubit-count, and chain-length data reported in the paper.

---

## What's implemented

| Encoding | Type | Qubits per vertex | Paper section |
|---|---|---|---|
| **One-hot** | QUBO | $k$ | § III-A |
| **Logarithmic (binary)** | HUBO, reduced to QUBO via Rosenberg | $\lceil \log_2 k \rceil$ | § III-C |

Both formulations use the provably sufficient penalty conditions derived in the paper (Theorems 3.1, 3.3, and 3.5). All penalty coefficients — including the Rosenberg quadratization strengths — are set automatically from graph size and structure; no manual tuning is required.

---

## Repository contents

The pipeline is split across three notebooks, each handling one stage:

### 1. `QuantumGraphColoring_Execution.ipynb`

End-to-end benchmarking on a set of input graphs. For each graph, it:

1. Computes the Brooks' theorem upper bound on $\chi(G)$ to size both encodings.
2. Builds the one-hot QUBO and the logarithmic HUBO (with penalties per Corollaries 3.2 and 3.4).
3. Reduces the HUBO to QUBO via `dimod.make_quadratic` with uniform Rosenberg strength per Corollary 3.6.
4. Submits both QUBOs to the D-Wave *Advantage2* QPU (or a simulated annealer if no token is provided), with optional per-qubit **inhomogeneous driving**.
5. Decodes samples, validates adjacency, computes Time-to-Solution, and records logical/physical qubit counts and chain-break fractions.

Writes `execution_results*.csv` into the chosen output directory.

### 2. `QuantumGraphColoring_Embedding.ipynb`

Focuses on the minor-embedding step alone. For each graph, it:

1. Constructs the one-hot QUBO and the quadratized logarithmic QUBO.
2. Finds minor embeddings for each onto the Zephyr-20 topology via `minorminer`, and serializes each embedding to JSON.
3. Aggregates per-encoding chain-length statistics (total physical qubits, mean, variance) across all graphs.

Writes per-graph embedding JSONs (`<graph>_<encoding>.json`) and a summary `embedding_results.csv` into the chosen output directory. The CSV is taken in by the plotting notebook to produce chain-length and physical-qubit figures.

### 3. `QuantumGraphColoring_Plotting.ipynb`

Consumes the benchmark CSV and the embedding CSV and produces the figures in the paper:

- **Time-to-Solution vs graph properties**, aggregated by **Kaplan–Meier survival-time analysis** (Section V-C of the paper). Observed TTS values enter as uncensored events; instances that never found the best energy within the read budget enter as right-censored at the experiment-wide wall-clock horizon. The survival function $\hat{S}(t)$ is estimated via the product-limit estimator; the median is read off at $\hat{S}(t) = 0.5$, and 95 % CIs on the median follow from Greenwood's variance inverted from $S$-space to time-space. Groups whose curve never crosses 0.5 are plotted as upward-arrow lower bounds.
- **Logical and physical qubit counts**, both as one-hot-vs-logarithmic scatters and as means grouped by graph properties.
- **Chain-length statistics** (average and variance) across encodings, grouped by graph size.

All figures are written to a single multi-page PDF.

---

## Requirements

- Python 3.9+
- `dimod`, `dwave-system`, `dwave-neal`, `minorminer`, `dwave-networkx` (Ocean SDK)
- `networkx`, `numpy`, `pandas`, `matplotlib`

Install via:

```bash
pip install dwave-ocean-sdk networkx numpy pandas matplotlib
```

A D-Wave Leap account and API token are required for hardware runs. Without a token, `QuantumGraphColoring_Execution.ipynb` falls back to `neal.SimulatedAnnealingSampler`; `QuantumGraphColoring_Embedding.ipynb` requires the Ocean SDK for the hardware topology.

---

## Usage

### Input format

Each `.pkl` file should contain either a single `networkx.Graph` or a list/tuple of `networkx.Graph` objects. The 350 random connected graphs used in the paper are available at <https://github.com/pangara/random-connected-graphs/>.

### Running the pipeline

Each notebook has a single configuration cell near the top or bottom. Set the paths, token, and sampler options there, then run the cells top to bottom.

**Execution notebook:**

```python
graph_dir            = "path/to/graphs"
output_dir           = "path/to/outputs"
use_dwave            = True
dwave_token          = "your-dwave-token"
dwave_solver         = "Advantage2_system1"
num_reads            = 1000
anneal_time          = 20.0                   # microseconds
max_nodes            = 100

# Inhomogeneous driving (optional)
use_inhomogeneous_driving = False
delta_max                 = 0.1
effective_field_method    = 'iterative'       # 'iterative' or 'exact'
greedy_postprocess        = False             # SteepestDescent refinement
```

**Embedding notebook** uses the same configuration block (without the read/anneal options), and produces per-graph embedding JSONs plus `results.csv`.

**Plotting notebook** reads paths via three top-level constants:

```python
BENCHMARK_CSV = "path/to/outputs/execution_results.csv"
EMBEDDING_CSV = "path/to/outputs/embedding_results.csv"
PDF_OUT       = "path/to/outputs/plotting_results.pdf"
```

---

## Output

`QuantumGraphColoring_Execution.ipynb` writes one row per graph to `execution_results.csv`, with columns including:

| Column | Meaning |
|---|---|
| `num_nodes`, `num_edges`, `max_degree`, `brooks_bound` | Graph statistics |
| `oh_num_logical`, `bin_pre_quad`, `bin_post_quad`, `bin_aux` | Qubit counts per encoding |
| `oh_num_physical`, `bin_num_physical` | Physical qubits after minor embedding |
| `oh_best_ncolors`, `bin_best_ncolors` | Fewest colors achieved |
| `oh_tts_us`, `bin_tts_us` | Time-to-Solution (50 % target probability) |
| `oh_cbf`, `bin_cbf` | Mean chain break fraction |

`QuantumGraphColoring_Embedding.ipynb` writes `embedding_results.csv` with columns:

| Column | Meaning |
|---|---|
| `name`, `nodes`, `edges`, `avg_deg`, `max_deg` | Graph identifier and statistics |
| `onehot_total`, `logarithmic_total` | Total physical qubits used per encoding |
| `onehot_average`, `logarithmic_average` | Mean chain length per encoding |
| `onehot_variance`, `logarithmic_variance` | Chain-length variance per encoding |

`QuantumGraphColoring_Plotting.ipynb` writes a single multi-page PDF containing the figures reported in the paper.

---

## Citation

If you use this code, please cite the paper:

```bibtex
@inproceedings{zaborniak2025qubit,
  title     = {Qubit-efficient and gate-efficient encodings of graph partitioning problems for quantum optimization},
  author    = {Zaborniak, Tristan and Angara, Prashanti Priya and Mulligan, Vikram and M\"uller, Hausi and Stege, Ulrike},
  booktitle = {To appear},
  year      = {2025}
}
```

---
