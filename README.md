# 4-bit Format Selection

**CS611 · Assignment 1 · Group 7 · Dataset C**

Compare four computationally cheaper selectors for choosing a numerical format for a weight matrix. Each selector is evaluated against full-matrix baseline scoring and exhaustive exponent search.

> **Main result:** All selectors choose **MXFP E2M1** on Dataset C, matching the exhaustive-search winner. Its optimized calibration-output error is **12.031%**.

See **[Report.pdf](Report.pdf)** for the methodology, detailed results, and discussion.

## Contents

- [Overview](#overview)
- [Project structure](#project-structure)
- [Setup and execution](#setup-and-execution)
- [Selectors](#selectors)
- [Results](#results)
- [Quantization conventions](#quantization-conventions)
- [Evaluation references](#evaluation-references)
- [Notes](#notes)
- [Team](#team)

## Overview

Choose among five formats:

**MXFP E2M1 · MXFP E1M2 · MXINT4 · MXUINT4 · BFP4**

The objective is to keep the quantized output close to the original calibration output:

```python
Y = W @ X_cal
Y_quantized = W_quantized @ X_cal

error_percent = 100 * np.linalg.norm(Y - Y_quantized) / np.linalg.norm(Y)
```

The selectors reduce the work required to compare formats by evaluating important blocks, sampled product terms, histogram estimates, or sampled calibration columns.

### Dataset

| File | Shape | Data type | Purpose |
|:---|:---:|:---:|:---|
| `W_C.npy` | `100 × 100` | `float32` | Weight matrix |
| `X_cal.npy` | `100 × 256` | `float32` | Calibration inputs |

Text copies are included as `W_C.txt` and `X_cal.txt`. All notebooks load the `.npy` files.

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
│   ├── algo2_columnwise_k.ipynb
│   ├── algo2_topk_masks.ipynb
│   └── algo2_generic_feature_union.ipynb
├── Algo-3/
│   └── algo3_histogram_selector.ipynb
└── Algo-4/
    └── algo4_calibration_sampling.ipynb
```

## Setup and execution

### Install dependencies

Requires **Python 3.9+**.

```bash
python -m pip install numpy pandas jupyter nbconvert
```

### Run interactively

From the project root:

```bash
cd Algo-1
jupyter notebook
```

Open the notebook and select **Run All**. Use the same procedure for the other algorithm folders.

Each notebook loads data from its parent folder:

```python
W = np.load("../W_C.npy")
X_cal = np.load("../X_cal.npy")
```

### Execute from the command line

Run these commands from the project root. Each notebook is executed with its own folder as the working directory, and outputs are saved in place.

```bash
jupyter nbconvert --to notebook --execute --inplace "Algo-1/block-immportance-selector (1).ipynb"
jupyter nbconvert --to notebook --execute --inplace Algo-2/algo2_columnwise_k.ipynb
jupyter nbconvert --to notebook --execute --inplace Algo-2/algo2_topk_masks.ipynb
jupyter nbconvert --to notebook --execute --inplace Algo-2/algo2_generic_feature_union.ipynb
jupyter nbconvert --to notebook --execute --inplace Algo-3/algo3_histogram_selector.ipynb
jupyter nbconvert --to notebook --execute --inplace Algo-4/algo4_calibration_sampling.ipynb
```

<details>
<summary><strong>Changing data paths</strong></summary>

If you move the data files, update the path configuration near the top of each notebook:

| Notebook | Path settings |
|:---|:---|
| Most notebooks | `W_PATH`, `X_CAL_PATH` |
| Algorithm 4 | `WEIGHTS_PATH`, `CALIBRATION_PATH` |
| Algorithm 2: feature union | `WEIGHT_DATASETS["Dataset C"]` |

Re-run the configuration cell before loading the data.

</details>

<details>
<summary><strong>Generated outputs</strong></summary>

| Location | Output |
|:---|:---|
| `Algo-4/` | `W_quantized_fast.npy` |
| `Algo-4/` | `W_quantized_reference.npy` |
| `Algo-2/` | `generic_vs_bruteforce_comparison.csv` |
| `Algo-2/topk_vs_bruteforce_results/` | Comparison CSVs |

</details>

## Selectors

### Algorithm 1 · Block importance

[Open notebook](Algo-1/block-immportance-selector%20%281%29.ipynb)

Assign importance using squared weights multiplied by calibration-row energy. Rank the 32-weight blocks and retain enough blocks to capture **50% of total importance**.

- Scores candidate formats on selected blocks.
- Uses `THRESHOLD = 0.5`.
- Refines near-ties with `TIE_MODE = "refine"`.

### Algorithm 2 · Sampled product contributions

Three variants estimate format quality from selected contributions to the matrix product.

| Variant | Selection rule | Notebook |
|:---|:---|:---|
| Column selection | Keep the 28 weight columns with the largest absolute column sums; requested fraction: 27.5% | [Columnwise](Algo-2/algo2_columnwise_k.ipynb) |
| Entry masks | Mark the largest 20% of absolute entries in each matrix; retain product terms where either entry is marked | [Top-k masks](Algo-2/algo2_topk_masks.ipynb) |
| Feature union | Combine the indices of the top 20% weight columns and top 20% calibration rows | [Feature union](Algo-2/algo2_generic_feature_union.ipynb) |

Column selection and feature union score the reduced product:

```python
W[:, S] @ X_cal[S, :]
```

### Algorithm 3 · Histogram-guided refinement

[Open notebook](Algo-3/algo3_histogram_selector.ipynb)

The **GHS-2** selector ranks formats using a weighted weight histogram.

1. Accept early when the leading format has a clear margin.
2. Otherwise, refine using sampled rows.
3. Refine surviving candidates using true output error and local exponent moves within ±3, including pairs of blocks.

The exhaustive search is used only as an evaluation reference. A notebook check verifies that the selector does not call it.

### Algorithm 4 · Calibration sampling

[Open notebook](Algo-4/algo4_calibration_sampling.ipynb)

Quantize the complete weight matrix, then score formats using **32 calibration columns**:

- The maximum-energy column.
- Another 31 columns selected randomly with a fixed seed.

Sampled contributions are rescaled. Near-ties are resolved using the full calibration set.

## Results

### Selected format

**Every selector chooses MXFP E2M1**, matching the exhaustive-search winner on Dataset C.

| Format | Exhaustive-search output error |
|:---|---:|
| **MXFP E2M1** | **12.031%** |
| MXINT4 | 13.452% |
| BFP4 | 13.452% |
| MXUINT4 | 13.452% |
| MXFP E1M2 | 14.004% |

### Computational work

| Selector | Reported work | Comparison reference |
|:---|---:|:---|
| Algorithm 1 | **39.0%** of estimated operations | Full baseline |
| Algorithm 4 | **13.9%** of estimated operations | Full baseline |
| Algorithm 3 | **52.7%** of rounding operations | Exhaustive search |

Algorithm 3 reaches **12.031% output error** using **185,264 roundings**, compared with **351,800** for the exhaustive reference.

> Operation counts and rounding counts measure different kinds of work. They are not runtime measurements. Sampled scoring errors also use different subsets, so compare the selected format and ranking rather than comparing sampled error percentages directly.

## Quantization conventions

All notebooks use the following shared settings:

| Setting | Convention |
|:---|:---|
| Element width | 4 bits |
| Block layout | Row-wise blocks of 32 weights |
| Shared exponent | One 8-bit power-of-two exponent per block |
| Exponent range | −127 to 127 |
| Effective storage | Approximately 4.32 bits per weight |
| Rounding | Round to nearest, ties to even |
| Overflow | Saturation |
| Input handling | Original FP32 values; no rescaling or shuffling |

### Format definitions

Here, `amax` denotes the maximum absolute weight in a block.

| Format | Representable values | Baseline exponent rule |
|:---|:---|:---|
| MXFP E2M1 | Signed magnitudes `{0, 0.5, 1, 1.5, 2, 3, 4, 6} × 2^e` | `floor(log2(amax)) − 2` |
| MXFP E1M2 | Signed magnitudes `{0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5} × 2^e` | E2M1 exponent minus 1 |
| MXINT4 | Integer codes `−8 … 7`, with step `0.25 × 2^e` | `floor(log2(amax))` |
| MXUINT4 | Codes `0 … 15`, decoded as `(q − 8) × 2^e` | Fit negative extent by 8 and positive extent by 7 |
| BFP4 | Integer codes `−8 … 7`, multiplied by `2^e` | `ceil(log2(amax)) − 3`, with `BFP_ALPHA = 1` |

**MXUINT4 convention:** This implementation uses a fixed midpoint zero point at code 8 because Dataset C contains negative weights (approximately 49.5%). The fixed offset requires no additional per-block storage. A plain unsigned representation would be ineligible for these weights.

## Evaluation references

| Reference | Procedure |
|:---|:---|
| **Full baseline** | Quantize every block in every format at its baseline exponent, then measure full calibration-output error |
| **Exhaustive exponent search** | Search block-exponent combinations within ±3 of baseline; widen to ±5 when the best exponent lies on the search boundary |

The exhaustive reference retains the tested combination with the smallest full calibration-output error. Its optimum is relative to the searched exponent windows.

## Notes

- `X_cal` contains calibration inputs. No held-out test set was supplied; `X_TEST_PATH` and `TEST_PATH` remain unset.
- Keep `W_C` and `X_cal` unchanged: do not rescale or shuffle them.
- Algorithm 3 includes an optional `RUN_TEST_SUITE` flag, disabled by default. It requires generator scripts that are not included in this package.
- Results reported here apply to Dataset C. Matching the winning format does not imply that every selector matches the complete exhaustive ranking or optimized error.

## Team

**Group 7**

| Member | Roll number |
|:---|:---|
| Naitikkumar H Patel | 261110039 |
| S Aditya | 261110057 |
| Jaimin Gajera | 261110028 |
| Krish Kalgude | See report |

See [Report.pdf](Report.pdf) for complete submission details.
