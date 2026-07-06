# Network Pulse — Count–Min Sketch for Data Streams

**Project 3, Mining of Massive Datasets.**

A single Jupyter notebook that implements the **Count–Min Sketch (CMS)** from
scratch and uses it to estimate word frequencies over a stream of several
Project Gutenberg books, read **line by line and token by token**. The notebook
demonstrates top-*k* frequent words, memory savings versus an exact dictionary,
and validation of the sketch's estimates against exact counts.

## Files

| File | Description |
|------|-------------|
| `network_pulse_count_min_sketch.ipynb` | The deliverable — runs top-to-bottom without errors. |
| `requirements.txt` | Python dependencies. |
| `final_report.md` | Short write-up of method, results, and conclusions. |
| `data/` | Auto-created cache of the downloaded Gutenberg books. |

## How to run

```bash
# 1. (recommended) create an isolated environment
python3 -m venv .venv
source .venv/bin/activate           # Windows: .venv\Scripts\activate

# 2. install dependencies
pip install -r requirements.txt

# 3a. run interactively
jupyter notebook network_pulse_count_min_sketch.ipynb
#     then: Kernel -> Restart & Run All

# 3b. or run headlessly end-to-end
jupyter nbconvert --to notebook --execute --inplace \
    --ExecutePreprocessor.timeout=600 \
    network_pulse_count_min_sketch.ipynb
```

The notebook downloads five books on first run (Pride and Prejudice,
Frankenstein, Sherlock Holmes, Moby Dick, Alice in Wonderland) and caches them
under `data/`. Later runs reuse the cache and need no network.

### Offline use

If the machine has no internet, download the five text files on any connected
machine and place them in `data/` named `<id>_raw.txt` (e.g. `data/1342_raw.txt`).
The URLs and exact filenames are listed in the notebook's **offline-fallback**
Markdown cell (section 4b). Any plain-text files placed in `data/` and listed in
`book_paths` will work — the pipeline is source-agnostic.

## What the algorithm does

- **From scratch:** a `CountMinSketch` class with a `numpy` `d × w` counter
  table, **Carter–Wegman universal hashing** `h_i(x) = ((a_i·x + b_i) mod p) mod w`
  (Mersenne prime `p = 2^61 − 1`), and a deterministic **polynomial** token→integer
  hash used *instead of* Python's non-reproducible built-in `hash()`.
- **Streaming:** a generator yields one token at a time; the full token list is
  never held in memory.
- **Parameters** (single config cell): `epsilon=0.001`, `delta=0.01`,
  `top_k=20`, `seed=42`, giving `w = ⌈e/ε⌉` and `d = ⌈ln(1/δ)⌉`.
- **No external sketch/count-min libraries** and no scikit-learn.

## Headline results (seed 42, five books)

- ~556k tokens streamed, ~22.5k distinct words.
- CMS top-20 matches the exact top-20 **100%**.
- Memory: **~106 KB** (fixed) vs **~2.6 MB** for the exact dictionary — ~25× smaller.
- Validation on 200 sampled tokens: **zero under-counts**, all errors within the
  `ε·N` bound.
