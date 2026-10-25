# 4-bit Format Selection

**CS611 · Assignment 1 · Group 7 · Dataset C**

Four computationally cheaper selectors for choosing among five 4-bit numerical formats, compared against full-matrix baseline evaluation and exhaustive exponent search.

> **Main result:** Every selector chooses **MXFP E2M1** on Dataset C, matching the exhaustive-search winner. The exhaustive-search calibration-output error is **12.031%**.

📄 **[Read the report](Report.pdf)**

## Project structure

```text
Group7_4bit_Dataset-C/
├── README.md
├── Report.pdf
├── W_C.npy
├── W_C.txt
├── X_cal.npy
├── X_cal.txt
├── Algo-1/
│   └── block-immportance-selector (1).ipynb
├── Algo-2/
│   ├── columnwise_k_vs_bruteforce_comparison_final (3).ipynb
│   ├── generic_vs_bruteforce_3.ipynb
│   ├── topk_vs_bruteforce_comparison.ipynb
│   ├── generic_vs_bruteforce_comparison.csv
│   └── topk_vs_bruteforce_results/
│       ├── agreement.csv
│       ├── cost.csv
│       ├── error_brute_force_pct.csv
│       ├── error_full_baseline_pct.csv
│       ├── error_topk_selected_pct.csv
│       ├── rank_brute_force.csv
│       ├── rank_full_baseline.csv
│       ├── rank_topk.csv
│       └── winner_table.csv
├── Algo-3/
│   └── generalized_histogram_selector_v2_no_bruteforce (1).ipynb
└── Algo-4/
    ├── algo-4.ipynb
    ├── W_quantized_fast.npy
    └── W_quantized_reference.npy
```

## Objective and data

Choose among **MXFP E2M1, MXFP E1M2, MXINT4, MXUINT4, and BFP4** so that the quantized matrix product stays close to the original calibration output.

```python
Y = W @ X_cal
Y_quantized = W_quantized @ X_cal

error_percent = 100 * np.linalg.norm(Y - Y_quantized) / np.linalg.norm(Y)
```

For these matrices, `np.linalg.norm` computes the Frobenius norm.

| File | Shape | Data type | Purpose |
|:---|:---:|:---:|:---|
| `W_C.npy` | `100 × 100` | `float32` | Dataset C weights |
| `X_cal.npy` | `100 × 256` | `float32` | Calibration inputs |

The `.txt` files are text copies. All notebooks use the `.npy` files.

## Setup

Requires **Python 3.9+**.

```bash
python -m pip install numpy pandas jupyter nbconvert
```

## Run the notebooks

### Interactive execution

From the project root:

```bash
jupyter notebook
```

Open a notebook inside its `Algo-*` folder and select **Run All**.

The notebooks load data from their parent folder using paths such as:

```python
W = np.load("../W_C.npy")
X_cal = np.load("../X_cal.npy")
```

### Command-line execution

Run the following commands from the project root. Outputs are saved inside the notebooks.

Filenames containing spaces and parentheses are quoted.

```bash
jupyter nbconvert --to notebook --execute --inplace "Algo-1/block-immportance-selector (1).ipynb"

jupyter nbconvert --to notebook --execute --inplace "Algo-2/columnwise_k_vs_bruteforce_comparison_final (3).ipynb"

jupyter nbconvert --to notebook --execute --inplace "Algo-2/topk_vs_bruteforce_comparison.ipynb"

jupyter nbconvert --to notebook --execute --inplace "Algo-2/generic_vs_bruteforce_3.ipynb"

jupyter nbconvert --to notebook --execute --inplace "Algo-3/generalized_histogram_selector_v2_no_bruteforce (1).ipynb"

jupyter nbconvert --to notebook --execute --inplace "Algo-4/algo-4.ipynb"
```

<details>
<summary><strong>Using data from another location</strong></summary>

Update the path configuration near the top of the relevant notebook:

| Notebook | Configuration |
|:---|:---|
| Most notebooks | `W_PATH`, `X_CAL_PATH` |
| Algorithm 4 | `WEIGHTS_PATH`, `CALIBRATION_PATH` |
| Algorithm 2: feature union | `WEIGHT_DATASETS["Dataset C"]` |

Re-run the configuration cell before loading the data.

</details>

## The four selectors

### Algorithm 1 — Block importance

[Open notebook](Algo-1/block-immportance-selector%20%281%29.ipynb)

Score weights using squared weight values multiplied by calibration-row energy. Rank the 32-weight blocks and retain enough blocks to capture **50% of total importance**.

- Evaluate candidate formats on the selected blocks.
- Set `THRESHOLD = 0.5`.
- Refine near-ties using `TIE_MODE = "refine"`.

### Algorithm 2 — Sampled product contributions

Three variants evaluate selected contributions to the matrix product.

| Variant | Selection rule | Notebook |
|:---|:---|:---|
| Column selection | Keep 28 weight columns with the largest absolute column sums; requested fraction: 27.5% | [Open](Algo-2/columnwise_k_vs_bruteforce_comparison_final%20%283%29.ipynb) |
| Entry masks | Mark the largest 20% of absolute entries in each matrix; retain product terms where either entry is marked | [Open](Algo-2/topk_vs_bruteforce_comparison.ipynb) |
| Feature union | Combine indices from the top 20% weight columns and top 20% calibration rows | [Open](Algo-2/generic_vs_bruteforce_3.ipynb) |

Column selection and feature union score the reduced product:

```python
W[:, S] @ X_cal[S, :]
```

### Algorithm 3 — Histogram-guided refinement

[Open notebook](Algo-3/generalized_histogram_selector_v2_no_bruteforce%20%281%29.ipynb)

The **GHS-2** selector ranks formats using a weighted weight histogram.

1. Accept early when the leading format has a clear margin.
2. Otherwise, refine using sampled rows.
3. Refine surviving candidates using true output error and local exponent moves within ±3, including pairs of blocks.

Exhaustive search is used only as an evaluation reference. A check cell verifies that the selector does not call it.

### Algorithm 4 — Calibration sampling

[Open notebook](Algo-4/algo-4.ipynb)

Quantize every weight, then evaluate candidate formats using **32 calibration columns**:

- The maximum-energy column.
- Another 31 columns selected randomly with a fixed seed.

Sampled contributions are rescaled. Near-ties are resolved using the complete calibration set.

## Results on Dataset C

All selectors choose **MXFP E2M1**, matching the exhaustive-search winner.

### Exhaustive-search output error

| Format | Error |
|:---|---:|
| **MXFP E2M1** | **12.031%** |
| MXINT4 | 13.452% |
| BFP4 | 13.452% |
| MXUINT4 | 13.452% |
| MXFP E1M2 | 14.004% |

### Computational work

| Selector | Work used | Reference |
|:---|---:|:---|
| Algorithm 1 | **39.0%** of estimated operations | Full baseline |
| Algorithm 4 | **13.9%** of estimated operations | Full baseline |
| Algorithm 3 | **52.7%** of rounding operations | Exhaustive search |

Algorithm 3 matches the exhaustive-search error of **12.031%**, using **185,264 roundings** compared with **351,800** for the reference.

> Estimated arithmetic operations and rounding counts measure different kinds of work; they are not measured runtimes. Sampled errors use different subsets, so compare the selected format and ranking rather than comparing sampled error percentages directly.

See [Report.pdf](Report.pdf) for complete results and discussion.

## Quantization conventions

| Setting | Convention |
|:---|:---|
| Element width | 4 bits |
| Block layout | Row-wise blocks of 32 weights |
| Shared exponent | One 8-bit power-of-two exponent per block |
| Exponent range | −127 to 127 |
| Effective storage | Approximately 4.32 bits per weight |
| Rounding | Round to nearest, ties to even |
| Overflow handling | Saturation |
| Data handling | Original FP32 values; no rescaling or shuffling |

### Format definitions

Here, `amax` is the maximum absolute weight in a block.

| Format | Representable values | Baseline exponent |
|:---|:---|:---|
| MXFP E2M1 | Signed magnitudes `{0, 0.5, 1, 1.5, 2, 3, 4, 6} × 2^e` | `floor(log2(amax)) − 2` |
| MXFP E1M2 | Signed magnitudes `{0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5} × 2^e` | E2M1 exponent minus 1 |
| MXINT4 | Signed codes `−8 … 7`, with step `0.25 × 2^e` | `floor(log2(amax))` |
| MXUINT4 | Codes `0 … 15`, decoded as `(q − 8) × 2^e` | Fit negative extent by 8 and positive extent by 7 |
| BFP4 | Signed codes `−8 … 7`, multiplied by `2^e` | `ceil(log2(amax)) − 3`, with `BFP_ALPHA = 1` |

**MXUINT4 convention:** This implementation uses a fixed midpoint zero point at code 8 because approximately 49.5% of Dataset C weights are negative. The fixed offset adds no per-block storage.

## Evaluation references

| Reference | Procedure |
|:---|:---|
| **Full baseline** | Quantize all blocks in all formats at baseline exponents and measure full calibration-output error |
| **Exhaustive exponent search** | Search block-exponent combinations within ±3 of baseline; widen to ±5 when the best exponent lies on the boundary |

The exhaustive reference retains the tested combination with the smallest full calibration-output error. Its optimum is relative to the searched exponent windows.

## Included outputs

| Location | Files |
|:---|:---|
| `Algo-2/` | `generic_vs_bruteforce_comparison.csv` |
| `Algo-2/topk_vs_bruteforce_results/` | Error, ranking, agreement, cost, and winner CSVs |
| `Algo-4/` | `W_quantized_fast.npy`, `W_quantized_reference.npy` |

## Notes

- `X_cal` is calibration data. No held-out test set was supplied; `X_TEST_PATH` and `TEST_PATH` remain unset.
- Do not rescale or shuffle `W_C` or `X_cal`.
- Algorithm 3 has an optional `RUN_TEST_SUITE` flag, disabled by default. It requires generator scripts that are not included.
- Results apply to Dataset C. Matching the winning format does not imply matching the complete exhaustive ranking or optimized error.

## Team

**Group 7**

| Member | Roll number |
|:---|:---|
| Naitikkumar H Patel | 261110039 |
| S Aditya | 261110057 |
| Jaimin Gajera | 261110028 |
| Krish Kalgude | See report |

See [Report.pdf](Report.pdf) for complete submission details.
