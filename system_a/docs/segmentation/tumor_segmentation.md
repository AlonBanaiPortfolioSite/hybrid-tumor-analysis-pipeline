# System A: Segmentation Pipeline

## Overview

Segment tumor regions from the tumor-specific channel in 4-channel 3D microscopy stacks of zebrafish with implanted human tumor cells. The key insight enabling the classical approach: **the injection site tumor is always the largest object and represents the data distribution for all tumor cells in the volume.**

---

## Challenge

Variable intensity across z-slices creates segmentation difficulty:
- **Z-slice intensity drift**: Top slices significantly brighter than bottom slices in some acquisitions
- **Within-tumor variability**: Heterogeneous signal intensity within tumor regions
- **Root cause**: Imaging artifacts (photobleaching or acquisition settings)

Standard global thresholding fails when calibrated on bright regions → false positives in dimmer slices.

---

## Stage 1: Classical Bootstrapping

### Algorithm

**Input:** Tumor channel (purple/channel 2) from 4-channel stack

**Step 1: Preprocessing**  
Bilateral filtering (slice-by-slice) to denoise while preserving edges.

**Step 2: Coarse Localization**  
Global thresholding to identify the largest connected component (injection site tumor).

**Step 3: Statistics Extraction**  
Extract region around main tumor object; compute local mean (μ) and standard deviation (σ).

**Rationale:** The injection site tumor is always the largest object and shares the same intensity distribution as smaller metastatic tumors. Its statistics guide adaptive, image-specific thresholding.

**Step 4: Fine Segmentation**  
Hysteresis thresholding using computed statistics to segment all tumor regions.

### Performance

**Success rate:** Works robustly on the vast majority of samples where intensity is relatively uniform across z-axis.

**Failure mode:** Rare cases with severe z-slice intensity drift
- Thresholds calibrated on bright slices → false positives in dimmer slices
- Classical approach cannot adapt to systematic intensity gradients

[Placeholder: Add example images showing successful case vs. rare failure case due to z-drift]

---

## Stage 2: Deep Learning Refinement (Planned)

### Motivation

Classical pipeline successfully segments most samples, generating pseudo-labels for deep learning. nnU-Net is expected to:
1. Learn spatial context across z-slices to handle intensity drift
2. Generalize to rare challenging cases where classical methods fail

### Status

**Current:** Classical pipeline provides training data from successful segmentations.

**Next steps:** Train nnU-Net on pseudo-labels, prioritizing samples with z-slice intensity variations.

**Pivot:** Efforts redirected to System B due to patient data availability and higher clinical priority. System A segmentation remains functional using classical approach for current dataset.

---

## Pipeline Architecture
```
4-Channel 3D Stack → Extract Tumor Channel
    ↓
Bilateral Filtering (denoise)
    ↓
Global Thresholding (find injection site tumor)
    ↓
Extract Region → Compute μ, σ
    ↓
Adaptive Hysteresis Thresholding
    ↓
Tumor Segmentation
    ↓
[Future] nnU-Net Refinement (if needed)
```

[Placeholder: Add pipeline diagram from poster]

---

## Key Design Principle

**Leverage domain knowledge:** The biological constraint (largest object = injection site tumor = representative sample) enables robust classical segmentation without labels. This principle of using domain structure to bootstrap from unlabeled data applies across both systems.

## Lessons Learned

- **Population-level statistics are robust:** Using the injection site tumor as a reference works better than trying to segment all tumors simultaneously
- **Classical methods can bootstrap deep learning:** Successful classical segmentations provide high-quality training data
- **Rare failures are acceptable:** When failures are rare and identifiable, classical methods remain practical without deep learning
- **Know when to pivot:** When System B patient data became available, resource allocation shifted to higher-impact work
