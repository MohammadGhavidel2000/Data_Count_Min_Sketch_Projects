# Network Pulse — Count–Min Sketch: Final Report

*Project 3, Mining of Massive Datasets.*

## 1. Problem

Given a high-volume data stream, estimate the frequency of every distinct item
using memory that does **not** grow with the number of distinct items. Exact
counting with a hash map is accurate but its memory scales with the vocabulary,
which is unbounded and unpredictable at web scale. We use word frequencies over
several Project Gutenberg books as a concrete, reproducible stand-in for such a
stream.

## 2. Method

**Count–Min Sketch (from scratch).** A `d × w` table of `int64` counters
(all zero initially) plus `d` independent hash functions. On each stream event
`x` we increment `C[i, h_i(x)]` for every row `i`; the estimate is
`f̂(x) = min_i C[i, h_i(x)]`. Because collisions only ever add counts, each row
over-estimates and the row-minimum is the tightest over-estimate — so the sketch
**never under-counts**.

**Universal hashing.** The hash functions are the Carter–Wegman family
`h_i(x) = ((a_i·x + b_i) mod p) mod w` with the Mersenne prime `p = 2^61 − 1`.
The coefficients `a_i, b_i` are drawn from a seeded RNG, making the whole sketch
reproducible. The modular arithmetic is done with Python big integers so the
`a_i·x` product (≈2^122) does not overflow.

**Token → integer.** Strings are converted to integers with a deterministic
polynomial rolling hash, `x = (Σ_k byte_k · B^k) mod p`, used *instead of*
Python's built-in `hash()` (which is salted per process and not reproducible).

**Streaming.** A generator reads each file line by line and yields one token at
a time (lowercased, alphabetic-only). The full token list is never materialised.
Exact counts are maintained in the same pass **only** as a validation/memory
baseline, never as the counting mechanism.

**Top-*k*.** Since a CMS cannot enumerate its items, a bounded-capacity
min-heap tracks heavy-hitter *candidates* during the stream; at the end every
candidate is re-queried and the true top-*k* is taken.

**Parameters.** `epsilon = 0.001`, `delta = 0.01` ⇒ `w = ⌈e/ε⌉ = 2719`,
`d = ⌈ln(1/δ)⌉ = 5`; `top_k = 20`; `seed = 42`.

## 3. Results (seed 42, five books)

| Metric | Value |
|--------|-------|
| Tokens streamed (N) | 556,237 |
| Distinct words | 22,521 |
| Throughput | ~199k tokens/s |
| CMS top-20 vs exact top-20 | 20/20 (100% overlap) |
| CMS memory (fixed) | ~106 KB (5 × 2719 × 8 B) |
| Exact-dictionary memory | ~2,623 KB |
| Memory reduction | ~24.7× |
| Under-counted tokens (sample of 200) | 0 |
| Max over-count error | 149 |
| Mean over-count error | 22.6 |
| Error bound `ε·N` | 556.2 |
| Sample within bound | 200/200 (100%) |

**Plots produced:** (1) estimated vs exact counts (log-log; all points on or
above `y = x`), (2) over-count error distribution (all ≥ 0, bounded by `ε·N`),
(3) memory comparison bar chart, (4) parameter sweep of error vs memory.

**Parameter sweep.** Reducing `ε` from 0.01 to 0.0005 widens the table (more
memory) and drives the maximum over-count error down roughly in proportion to
`1/w`, empirically confirming the `error ≤ ε·N` guarantee and the memory/accuracy
trade-off.

## 4. Conclusions

- The from-scratch CMS behaves exactly as theory predicts: **one-sided error**
  (no under-count) and over-counts comfortably within `ε·N`.
- It recovers the true most-frequent words with 100% top-20 overlap while using
  a small, **fixed** memory footprint — ~25× less than the exact dictionary,
  and constant regardless of how large the vocabulary grows.
- Limitations: the sketch answers point queries only (a helper structure is
  needed for top-*k*), and it over-counts most for rare items sharing buckets
  with frequent ones — mitigated by the row-minimum and a sufficiently large `w`.
- Fully reproducible: one seed drives all randomness, the token→integer map is
  deterministic, and the notebook runs top-to-bottom from a clean state.
