# System A: Immune Cell Segmentation

## Overview

Segment immune cells (microglia) from the immune cell-specific channel in 4-channel 3D microscopy stacks. The primary challenge is **extreme class imbalance** - foreground objects occupy <5% of image volume.

---

## Challenge

- **Extreme class imbalance:** Microglia occupy <5% of volume, background dominates
- **High intra-class variance:** Individual cells have bright centers and dim peripheries
- **Variable imaging conditions:** Intensity distributions vary significantly across acquisitions

Standard histogram-based and clustering methods fail due to the combination of class imbalance and intra-class intensity variation.

<!-- TODO: Add example image showing sparse microglia distribution -->

---

## Why Standard Methods Fail

### Histogram-Based Thresholding

Class imbalance distorts intensity distributions. Background dominates histograms, causing optimal thresholds to be biased toward background → systematic under-segmentation (misses dim cell regions).

### Clustering-Based Methods (GMM)

High intra-class variance within foreground objects (bright centers, dim peripheries). Single Gaussian insufficient to capture this variation. Even with class-weighted GMM to compensate for imbalance, the method fails to model the full intensity range within cells.

<!-- TODO: Add failure examples for histogram and GMM methods -->

---

## Stage 1: Classical Bootstrapping - Hysteresis Thresholding

### Algorithm

**Step 1: Preprocessing**  
Bilateral filtering to denoise while preserving edges.

**Step 2: Hysteresis Segmentation**  
Two-threshold approach with percentile-based values:
- **High threshold:** Seeds high-confidence regions (bright cell centers)
- **Low threshold:** Grows into adjacent moderate-confidence regions (dim cell peripheries)

**Rationale:**  
- Percentile-based thresholds adapt to per-image intensity distributions (handles variable imaging conditions)
- Two-threshold strategy exploits biological structure (bright centers + dim edges)
- Avoids background bias from histogram methods

### Performance

Generated **~200 images** with acceptable segmentation quality.

**Limitation:** Does not generalize across all imaging conditions. Used as **bootstrapping strategy** to generate training data for deep learning, not as final segmentation method.

<!-- TODO: Add example of successful hysteresis segmentation -->

---

## Stage 2: Deep Learning - nnU-Net

### Rationale

200 labeled images sufficient for training 2D segmentation network with aggressive data augmentation.

### Architecture Selection

**CNN over Transformer:**  
Transformers require significantly more training data. CNNs are proven effective in low-data medical imaging regimes.

**Why nnU-Net:**
- Automatic data augmentation pipeline optimized for limited medical imaging datasets
- Medical image preprocessing (intensity normalization, anisotropic spacing handling)
- Self-configuring architecture eliminates manual hyperparameter tuning
- Established performance on small-data medical segmentation tasks

### Training Strategy

**Data:** 2D slices extracted from 3D stacks (maximizes training samples given data constraints)

**Augmentation:** nnU-Net default pipeline (geometric and intensity transformations)

**Loss function:** Standard Dice + cross-entropy (default nnU-Net configuration handles class imbalance)

### Performance

**Dice score:** ~0.8

**Interpretation:**  
Sufficient for downstream analysis of immune response spatial patterns (e.g., microglia density near vs. far from tumor). May not be sufficient for detailed morphological analysis of individual cells.

<!-- TODO: Add comparison images showing classical vs. nnU-Net segmentation -->

---

## Pipeline Architecture
```
Immune Cell Channel (from 4-channel stack)
    ↓
Bilateral Filtering
    ↓
Hysteresis Thresholding (percentile-based)
    ↓
~200 Pseudo-labeled Images
    ↓
nnU-Net Training (2D slices)
    ↓
Segmentation Model (Dice ~0.8)
    ↓
Final Masks (spatial analysis quality)
```

---

## Key Design Principles

- **Percentile-based thresholds:** Adapt to per-image intensity distributions without manual tuning
- **Hysteresis exploits structure:** Two thresholds match biological reality (bright centers + dim peripheries)
- **Bootstrap with classical methods:** Generate training data from unlabeled images using domain knowledge
- **Match model complexity to data:** CNNs appropriate for ~200 samples; Transformers would overfit

---

## Lessons Learned

- **Class imbalance breaks standard methods:** Background-dominated histograms and GMMs systematically fail
- **Biological structure enables bootstrapping:** Two-threshold hysteresis matches cell morphology (bright center + dim edge)
- **Quality requirements matter:** Dice ~0.8 sufficient for spatial analysis, insufficient for morphology - match segmentation quality to downstream task
- **2D training on 3D data:** Extracting slices multiplies training samples when 3D annotation is unavailable

---

## Future Improvements

If detailed morphological analysis is needed:
- Increase training data (more manual corrections or semi-automated refinement)
