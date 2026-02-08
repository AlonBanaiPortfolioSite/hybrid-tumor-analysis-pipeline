# System B: Invasion Segmentation

## Overview

Identify individual tumor cells invading healthy tissue. Uses high-quality healthy tissue and tumor masks from main segmentation pipeline.

**Status:** Work in progress

---

## Challenge

Invading cells are:
- Orders of magnitude smaller than primary tumor
- Embedded within healthy tissue (inside healthy tissue mask region)
- Variable intensity (some bright, some dim)

---

## Approach: Intensity-Based Detection with Hysteresis

### Step 1: Define Search Region
Use healthy tissue mask to identify regions where invading cells may be present.

### Step 2: Intensity Analysis
Extract tumor channel intensities within healthy tissue mask region.

**Expected distribution:**
- **Normal distribution:** Healthy cells (background signal, leakage)
- **High-intensity tail:** Invading tumor cells

### Step 3: Statistical Thresholding
Hysteresis thresholding using statistics from healthy cell distribution:
- **High threshold:** μ + const₁ × σ (seed high-confidence invading cells)
- **Low threshold:** μ + const₂ × σ (grow to capture full cell extent)

**Rationale:** Percentile-based approach adapts to per-sample intensity distributions. Two thresholds capture variable cell brightness.

### Step 4: Quality Control
Visual inspection + automated filtering to remove artifacts:
- Size filtering (invading cells have expected size range)

### Step 5: Deep Learning Refinement
Train nnU-Net on high-quality filtered results from Step 4.

**Why nnU-Net after classical:** Classical method generates training data, deep learning improves generalization and handles edge cases.

---

## Current Status

- ✅ Step 1-2: Search region definition and intensity extraction implemented
- 🔄 Step 3: Optimizing threshold constants (const₁, const₂)
- 🔄 Step 4: QC criteria development
- ⏳ Step 5: Pending sufficient training data from Steps 3-4

---

## Preliminary Results

[To be added: Example images showing detected invading cells]

---

## Next Steps

1. Finalize threshold selection (μ + const×σ) through cross-validation
2. Build filtered training dataset (~50-100 samples)
3. Train nnU-Net for robust invasion detection
4. Validate against clinical recurrence data (if available)
