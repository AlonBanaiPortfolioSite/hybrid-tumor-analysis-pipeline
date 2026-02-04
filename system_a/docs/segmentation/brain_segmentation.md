# System A: Brain Tissue Segmentation

## Overview

Segment brain tissue from the brain-specific channel in 4-channel 3D microscopy stacks. The primary challenge is handling large intensity voids (hundreds of pixels) within brain regions caused by uneven staining and optical distortions.

---

## Challenge

- **Large intensity voids:** Hundreds of pixels of missing signal within brain tissue
- **Uneven staining:** Highly variable fluorescence intensity across tissue
- **Optical distortions:** Black spots and artifacts from imaging
- **Anatomical variability:** Different image stacks capture slightly different anatomical regions (cropping, field-of-view limitations)

Standard segmentation approaches fail because voids are too large for morphological filling without causing over-segmentation.

<!-- TODO: Add example image showing brain channel with large voids -->

---

## Approach 1: GMM with Morphological Postprocessing (Failed)

### Algorithm

1. **Preprocessing:** Bilateral filtering to reduce noise while preserving edges
2. **Segmentation:** GMM with class-weighted parameters to handle foreground/background imbalance
3. **Slice initialization:** Use GMM parameters from slice i to initialize slice i+1 (improves convergence)
4. **Postprocessing:** Morphological operations (dilation, hole filling) to close voids

### Failure Analysis

Works only on small subset with minimal voids. **Fails when voids are large:**
- Aggressive dilation required to fill voids → merges brain with nearby structures (tumor, vessels)
- Hole filling connects regions incorrectly
- **Key insight:** When postprocessing becomes too aggressive to handle artifacts, the base method is inappropriate

<!-- TODO: Add failure case image showing over-segmentation from aggressive morphology -->

---

## Approach 2: Atlas-Based Registration (Failed)

### Strategy

Manually segment reference brains; propagate masks to new samples via registration.

### Design Choices

**Transformation:** Similarity (rotation + uniform scaling)
- More robust than rigid (too restrictive) or affine/deformable (overfitting risk)
- Biological variability accommodated by uniform scaling

**Similarity metric:** Mutual information
- Robust to intensity variations between samples
- Normalized cross-correlation fails under intensity differences

**Masking high-intensity regions:** Limited improvement
- Voids are spatially distributed; cannot be excluded by intensity masking

### Failure Mode

**Anatomical mismatch:** Fails when target images are missing brain regions due to cropping or limited field-of-view. Registration assumes anatomical correspondence; incomplete anatomy produces incorrect warping.

**Root cause:** Different image stacks start and finish at slightly different anatomical regions. No single atlas covers all variations.

<!-- TODO: Add example showing registration failure due to anatomical mismatch -->

---

## Approach 3: Hybrid Registration + Refinement (Failed)

### Strategy

1. **Atlas selection:** Identify images with complete brain regions (key anatomical landmarks present)
2. **Registration:** Apply similarity transformation with mutual information
3. **Contour-based refinement:** Use edge information to correct boundary errors
4. **Goal:** Generate hundreds to thousands of masks for deep learning training

### Failure Analysis

**Still fails due to anatomical variability:** Registration cannot handle the fact that different image stacks start and finish at slightly different anatomical regions. No amount of refinement fixes fundamentally misaligned anatomy.

---

## Approach 4: GMM + Active Contour (Current Solution)

### Strategy

Re-examine Approach 1 with a two-stage pipeline that accepts intermediate over-segmentation:

**Stage 1: Aggressive morphological processing**
1. Apply GMM segmentation (as in Approach 1)
2. Use **very aggressive** morphological operations to fill large voids
3. Accept that this causes over-segmentation (merges brain with nearby structures)

**Stage 2: Active contour refinement**
1. Use edge information to correct boundaries
2. Shrink over-segmented regions back to true brain boundaries
3. Leverage intensity gradients at brain-tumor and brain-vessel interfaces

### Rationale

- **Stage 1 solves void problem:** Aggressive morphology successfully closes large voids
- **Stage 2 solves over-segmentation:** Active contours use local edge information to find correct boundaries
- **Key insight:** Over-segmentation is acceptable if it's correctable

### Current Status

work in progress

**Challenge:** Active contours may fail when boundaries are weak (low contrast between brain and adjacent structures). Visual inspection and quality control needed.

<!-- TODO: Add before/after images showing GMM over-segmentation → active contour correction -->

---

## Approach 5: Deep Learning with Transformers (Planned)

### Challenge

Standard CNNs rely on local features. Large voids (hundreds of pixels) exceed typical receptive field sizes - CNNs cannot distinguish voids from background using local context alone.

### Solution

**Architecture:** Transformer or CNN-Transformer hybrid
- Self-attention mechanism captures long-range dependencies
- Can infer that large voids surrounded by brain tissue are foreground, not background

**Implementation:** Extend nnU-Net framework with Transformer blocks

**Training data:** Generated from Approach 4 (GMM + active contour)

**Status:** Planned - pending sufficient training data from Approach 4

---

## Pipeline Architecture
```
Brain Channel (from 4-channel stack)
    ↓
Bilateral Filtering
    ↓
GMM Segmentation
    ↓
Aggressive Morphological Operations (accept over-segmentation)
    ↓
Active Contour Refinement (correct boundaries)
    ↓
Segmentation Masks / Training Data
    ↓
[Planned] Transformer-based nnU-Net → Robust Segmentation
```

---

## Key Design Principles

- **Accept intermediate failures if correctable:** Over-segmentation from morphology is acceptable if active contours can fix it
- **Two-stage processing:** Separate void-filling from boundary refinement
- **Long-range context for deep learning:** Voids require Transformers, not standard CNNs
- **Iterative refinement:** Each failed approach informs the next design

---

## Lessons Learned

- **Morphological operations have limits alone:** But useful as first stage in multi-stage pipeline
- **Registration requires anatomical consistency:** Atlas-based methods fail with field-of-view variations - not solvable with better atlases
- **Stage failures can be acceptable:** Over-segmentation from stage 1 can be fixed in stage 2
- **Problem structure dictates architecture:** Void size (hundreds of pixels) requires long-range methods (Transformers), not local methods (CNNs)
