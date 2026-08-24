# Computer Architecture Assignment — Labs 9, 10 & 11
## Branch Prediction and Cache Replacement in ChampSim

This repository contains the code, results, graphs, and report for the ChampSim
assignment covering branch prediction (Task 1) and last-level-cache replacement
(Task 2), including the two bonus questions.

The full write-up, with all tables, graphs, and analysis, is in **`report.pdf`**.

---

## What was changed, and where in the simulator

All new files were added to the standard ChampSim source tree. Nothing in the
baseline simulator was modified except by adding these new, self-contained files
that the build script selects by name.

### Branch predictors (Task 1c — bonus) → `branch/`

These are placed in ChampSim's `branch/` directory and selected as the
`${BRANCH}` argument to `build_champsim.sh`.

| File (in `code/`) | Goes in | New or baseline | What it is |
|---|---|---|---|
| `bullseye.bpred` | `branch/` | **New (mine)** | Modified Bullseye predictor: Hashed Perceptron base + a hard-to-predict (H2P) table with per-PC mini-perceptrons, fixed thresholds. |
| `bullseye_adaptive.bpred` | `branch/` | **New (mine)** | Adaptive-threshold version: admission thresholds scale with the number of H2P entries, plus adaptive eviction. |

Bimodal, GShare, Perceptron, and Hashed Perceptron are the predictors that ship
with ChampSim and were used unmodified as baselines.

### Cache replacement policies (Task 2c, 2e — bonus) → `replacement/`

These are placed in ChampSim's `replacement/` directory with the `.llc_repl`
extension and selected as the `${LLC_REPLACEMENT}` argument.

| File (in `code/`) | Goes in | New or baseline | What it is |
|---|---|---|---|
| `mru.llc_repl` | `replacement/` | **New (mine)** | Most-Recently-Used eviction (deliberate worst-case baseline). |
| `random.llc_repl` | `replacement/` | **New (mine)** | Random eviction with a fixed seed for reproducibility. |
| `hcit.llc_repl` | `replacement/` | **New (mine)** | Hard-to-Cache Identification Table: a Bullseye-inspired policy that tracks per-PC miss/reuse behaviour and protects only the small set of PCs responsible for most misses. |

LRU, SRRIP, DRRIP, and SHIP ship with ChampSim and were used unmodified.

### Multi-core trace selection (Task 1/2 — bonus requirement)

| File | Goes in | What it is |
|---|---|---|
| `random_trace_gen.py` | run from anywhere | Randomly selects 4 traces from the assignment's 15-trace pool using a fixed seed (42). Its output (`selected_traces.txt`, `selection_log.json`) is in `results/`. |

The seed-42 four-core mix is: 619.lbm, 403.gcc, 401.bzip2, 620.omnetpp.

---

## Repository layout

```
.
├── README.md                 # this file
├── report.pdf                # full write-up: tables, graphs, analysis
├── code/                     # all new source files
│   ├── bullseye.bpred
│   ├── bullseye_adaptive.bpred
│   ├── mru.llc_repl
│   ├── random.llc_repl
│   └── hcit.llc_repl
├── random_trace_gen.py       # multi-core trace selector
├── graphs/                   # all plots used in the report
└── results/
    ├── raw/                  # raw ChampSim output .txt files
    ├── tables/               # extracted CSV summary tables
    ├── selected_traces.txt   # trace-generator output
    └── selection_log.json
```

---

## How to reproduce

### 1. Install the new files into a ChampSim tree

Copy the branch predictors into `branch/` and the replacement policies into
`replacement/`:

```
cp code/bullseye.bpred          ChampSim-master/branch/
cp code/bullseye_adaptive.bpred ChampSim-master/branch/
cp code/mru.llc_repl            ChampSim-master/replacement/
cp code/random.llc_repl         ChampSim-master/replacement/
cp code/hcit.llc_repl           ChampSim-master/replacement/
```

### 2. Build

The build script takes:
`./build_champsim.sh ${BRANCH} ${L1I_PREF} ${L1D_PREF} ${L2C_PREF} ${LLC_PREF} ${LLC_REPLACEMENT} ${NUM_CORE}`

Examples:

```
# Branch predictor (Task 1), single core
./build_champsim.sh bullseye_adaptive no no no no lru 1

# LLC replacement policy (Task 2), single core
./build_champsim.sh hashed_perceptron no no no no hcit 1

# Four-core build
./build_champsim.sh hashed_perceptron no no no no hcit 4
```

### 3. Run

```
# Single core: ./run_champsim.sh [BINARY] [N_WARM] [N_SIM] [TRACE]
./run_champsim.sh hashed_perceptron-no-no-no-no-hcit-1core 1 10 605.mcf_s-665B.champsimtrace.xz

# Four core: ./run_4core.sh [BINARY] [N_WARM] [N_SIM] [N_MIX] [TRACE0..3]
./run_4core.sh hashed_perceptron-no-no-no-no-hcit-4core 1 10 0 \
  619.lbm_s-3766B.champsimtrace.xz 403.gcc-16B.champsimtrace.xz \
  401.bzip2-277B.champsimtrace.xz 620.omnetpp_s-874B.champsimtrace.xz
```

Configuration used throughout (except where a task varies it): single/4-core,
L1I/L1D 32 KB 8-way, L2 256 KB 8-way, LLC 4 MB 16-way, 64 B lines, LRU at L1/L2,
1 M warmup + 10 M simulated instructions.

---

## Summary of results

- **Branch prediction:** Hashed Perceptron is the strongest baseline; the adaptive
  Bullseye gives the lowest MPKI on the hardest trace (605.mcf, 17.19) and the best
  aggregate four-core throughput, while behaving like the base predictor on easy traces.
- **LLC:** a larger cache helps only memory-intensive workloads (605.mcf); associativity
  barely matters once the cache is capacity-limited. Among replacement policies SHIP is
  strongest overall, and the new HCIT policy approaches SHIP by protecting only the small
  set of hard-to-cache PCs.

Full numbers, graphs, and discussion are in `report.pdf`.

---

## Attribution

The Bullseye H2P concept is inspired by Behrendt, Pun & Nair, *"Taming Wild
Branches"* (CBP 2025), https://arxiv.org/pdf/2506.06773. The `bullseye`,
`bullseye_adaptive`, `mru`, `random`, and `hcit` implementations in this
repository are my own. LRU, SRRIP, DRRIP, SHIP, and the four baseline branch
predictors are part of the ChampSim distribution.
