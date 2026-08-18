# V2V 6G Highway Resource Allocation — Sweep-Line + Greedy SINR + Edge-AI

A Computer Networks coursework simulation (VIT Vellore, Dept. of CSE — Shaurya Sorayan,
22BCE2674, with Pratik Kumar Jaishwal, 23BCE0300) modeling Vehicle-to-Vehicle (V2V)
communication on a 3-lane highway using next-generation mmWave (60 GHz) and THz (140 GHz)
frequency bands. A computational-geometry **sweep-line algorithm** finds which vehicles are
close enough to communicate, a **greedy SINR-based channel allocator** assigns radio channels
to minimize interference, and a small **Random Forest "Edge-AI" model** predicts link
throughput from channel conditions.

The project's naming and general approach ("greedy algorithm + sweep line model" for V2V/6G
resource allocation) is adapted from a real published paper — Mande, S. & Ramachandran, N.,
*"A novel approach for efficient resource allocation in 6G V2V networks using neighbor-aware
greedy algorithm and sweep line model,"* Egyptian Informatics Journal, 2025 — which is
reviewed slide-by-slide in `_source_docs/Paper_Review_Slides.pptx`. The actual simulation
code in this repo (parameters, mmWave/THz band modeling, the Edge-AI throughput predictor) is
this project's own implementation, not a reproduction of that paper's NS3/SUMO setup — see
[A note on the submitted report](#a-note-on-the-submitted-report-vs-this-repo) below.

---

## What's built

| Part | What it does |
|---|---|
| **Part 1 — Setup** | Global simulation parameters (40 vehicles, 3 lanes, 2000m road, 12 channels), path-loss models for mmWave and THz bands, adaptive band selection by SNR |
| **Part 2 — Core simulation** | Vehicle mobility model, a **sweep-line algorithm** to efficiently find all vehicle pairs within 250m communication range, a **greedy SINR-maximizing channel allocator** across 12 channels, static plots (top-down highway snapshot, channel utilization, SINR/throughput histograms) |
| **Part 3 — Animation** | A `matplotlib.animation.FuncAnimation` top-down highway visualization — see [Known limitations](#known-limitations), this part does **not** re-run the real algorithm per frame |
| **Part 4 — Edge-AI prediction** | A `RandomForestRegressor` trained on the Part 2 allocation results to predict per-link throughput from distance/band/SINR, plus Jain's Fairness Index and estimated latency as network-level summary metrics |

Open `architecture/interactive_architecture.html` in any browser for a full component-by-
component breakdown, including the honest data-flow story of what's real simulation vs.
illustrative animation.

---

## Core algorithms

**Sweep-line link discovery** (`sweep_line_find_links`): sorts all 40 vehicles by their
position along the road, then sweeps a moving window left-to-right, maintaining only the
vehicles that could still be within communication range (250m) of the current vehicle. This
avoids the naive O(n²) all-pairs distance check by discarding vehicles once they fall out of
range of the sweep front — a classic computational-geometry technique applied to a networking
problem. On the recorded run: **132 potential V2V proximity pairs** found among 40 vehicles.

**Greedy SINR-based channel allocation**: for each candidate link, sorted by descending
estimated achievable rate, the algorithm tries all 12 channels and picks whichever one gives
the highest SINR (Signal-to-Interference-plus-Noise Ratio) once existing allocations on that
channel are accounted for as interference — a greedy, not globally optimal, assignment.
Result on the recorded run: **183 allocated links, 417.32 Mbps total system throughput,
0.89 dB average SINR.**

**Adaptive band selection**: for each link, both a 60 GHz mmWave path-loss model (3GPP-style)
and a 140 GHz THz path-loss model (free-space spreading + atmospheric absorption) are
computed, and the band giving the higher received power is chosen per-link — modeling the
real 6G-era idea that different frequency bands suit different link distances.

---

## A note on the submitted report vs. this repo

The submitted `_source_docs/Submission_Report.pdf` describes a simulation with **different
parameters** than the actual notebook: 100-300 vehicles (code: 40), a 2-lane road (code: 3
lanes), and 10 channels (code: 12) — and reports a results table (spectrum utilization
78%→94%, throughput 8.2→11.5 Mbps, latency 4.3→2.6 ms, "Random vs. Proposed") that does not
correspond to any output this notebook produces. The report also never mentions mmWave/THz
band modeling or the Edge-AI throughput predictor, even though both are substantial parts of
the actual notebook (Parts 1 and 4). The report's "Sample Python Simulation" code listing is
a simplified toy snippet that doesn't match the notebook's real sweep-line/greedy
implementation either.

Separately, the report presents the "Greedy Algorithm + Sweep Line" hybrid as this project's
own proposed method, citing Mande & Ramachandran's 2025 paper as one of sixteen numbered
references among others — but `_source_docs/Paper_Review_Slides.pptx` makes clear that paper
is in fact the specific template this project's architecture and terminology were adapted
from (its own NS3/SUMO/20-vehicle simulation setup is quite different from what's implemented
here). The report itself does not disclose this relationship.

This README and the architecture explorer describe only what `notebooks/v2v_sweepline_simulation.ipynb`
actually implements and actually output on its recorded run (132 potential pairs, 183
allocated links, 417.32 Mbps total throughput, 0.89 dB average SINR, R²=0.999 on the Edge-AI
model) — not the report's separate claimed figures.

---

## Known limitations

- **The Part 3 animation does not run the real algorithm.** Its per-frame `update()` function
  explicitly replaces the channel allocation with **randomly generated fake assignments**
  (`channels[ch] = [(random.choice(veh_list), random.choice(["THz","mmWave"]), ...)]`), and
  the on-screen "live" throughput/SINR readouts are generated from `sin()`/`cos()` functions
  with a code comment literally reading `# fake live metrics (for illustration)` — not
  recomputed from the vehicles' real positions. The animation is visually representative of
  what a live system might look like, but its numbers are not connected to Part 2's real
  simulation once playback starts. This is documented here rather than silently presented as
  a live re-run of the real allocator.
- **The Edge-AI model's R²=0.999 is very likely overfit**, not a genuinely strong result. It's
  trained on ~137 samples (train split of ~183 allocated links) and evaluated on a ~46-sample
  test split, using features (`distance`, `band`, `sinr_db`) that are deterministic
  transformations of each other via the same path-loss formulas used to generate the training
  labels in the first place (`throughput_mbps` is itself a direct function of `sinr_db` via
  Shannon capacity) — the model is closer to learning the closed-form relationship it was
  generated from than predicting genuinely independent outcomes.
- **No real-world validation.** All results come from a self-contained synthetic simulation
  (`np.random.seed(42)`) — there's no comparison against real V2V measurement data or a
  published baseline's reported numbers.
- **Interference model is simplified.** The SINR calculation only accounts for interference
  from transmitters already assigned to the same channel at allocation time — it doesn't
  model fading, shadowing, or mobility-induced SINR changes between allocation and actual
  transmission.
- **Jain's Fairness Index of 0.400** indicates a fairly unequal throughput distribution across
  links (1.0 would be perfectly fair) — worth noting as an actual finding, not glossed over.

---

## Running it

```bash
pip install numpy matplotlib scipy scikit-learn pandas
jupyter notebook notebooks/v2v_sweepline_simulation.ipynb
```

Run all cells top to bottom — Part 1 sets up parameters, Part 2 runs the core sweep-line +
greedy simulation (takes a few seconds for 40 vehicles × 12 channels), Part 3 renders the
animation (may take a minute to build all 200 frames), and Part 4 trains the Edge-AI model.

---

## Project structure

```
v2v-6g-sweepline-resource-allocation/
├── README.md
├── architecture/
│   └── interactive_architecture.html   # click-through system + algorithm explainer
├── notebooks/
│   └── v2v_sweepline_simulation.ipynb  # all 4 parts, single notebook
└── _source_docs/
    ├── Submission_Report.pdf            # original coursework submission report
    └── Paper_Review_Slides.pptx         # presentation slides
```
