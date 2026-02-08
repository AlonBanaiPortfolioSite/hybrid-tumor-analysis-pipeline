# System B: Healthy Tissue-Tumor Segmentation Pipeline

## Overview

Two-class segmentation (healthy tissue + tumor) from 2-channel 3D microscopy of patient-derived tissue invaded by tumor cells. Pipeline addresses fluorescence leakage, temporal misalignment, and absence of manual labels through iterative bootstrapping and data correction.

**Next stage:** Invasion segmentation (identifying individual tumor cells invading healthy tissue - see `invasion_segmentation.md`)

---

## Challenge

- **Channel leakage:** Fluorescence signal bleeds between channels despite channel-specific markers
- **Temporal misalignment:** Hours between channel acquisitions → cells move, causing spatial misregistration
- **No manual labels:** 3D annotation prohibitively expensive
- **Variable quality:** Inconsistent signal intensity and contrast across samples
- **Downstream constraint:** Segmentation feeds invasion analysis where false negatives are catastrophic (see Loss Design)

<!-- TODO: Add image showing raw data with visible leakage and misalignment -->

---

## Pipeline Overview
```
Raw 2-Channel Data
    ↓
Stage 1: Classical Segmentation + Manual Curation → Clean seed labels
    ↓
Stage 2: Train nnU-Net (First Model) → Inference on full dataset
    ↓
Stage 3: Filter High-Quality Predictions → Visual QC
    ↓
Stage 4: Fix Data Issues (Registration + Leakage Correction)
    ↓
Stage 5: Cross-Channel Bootstrapping → Segment second class without labels
    ↓
Stage 6: Retrain with Expanded Dataset (high-quality results from Stage 5)
    ↓
Stage 7: Post-Processing (largest component, hole filling, active contours)
    ↓
Stage 8: Final Two-Class Model (unified healthy tissue + tumor)
    ↓
High-Quality Masks → Invasion Analysis
```

---

## Stage 1: Initial Label Generation (Classical Methods)

**Approach:**  
Apply thresholding + morphological operations to generate candidate masks. Manually curate high-quality subset through visual inspection.

**Rationale:**  
Deep models need clean seed labels. Starting with interpretable classical methods prevents propagating errors from noisy initial data.

**Output:** Small set of high-confidence training labels

<!-- TODO: Add example of classical segmentation result -->

---

## Stage 2: First Deep Model (nnU-Net)

**Approach:**  
Train nnU-Net on curated classical subset. Run inference across full dataset to generate pseudo-labels.

**Rationale:**  
nnU-Net automates hyperparameter tuning, providing strong baseline without manual model engineering. Expands labeled dataset through prediction on unlabeled samples.

**Output:** Pseudo-labels for previously unlabeled samples

---

## Stage 3: Filter High-Quality Predictions

**Approach:**  
Visual inspection + failure analysis to select only high-confidence predictions from Stage 2 inference.

**Rationale:**  
Prevents amplifying systematic errors across training iterations. Only reliable predictions become training labels for next iteration.

**Output:** Filtered subset of high-quality pseudo-labels

---

## Stage 4: Fix Channel-Specific Issues

### Registration: Spatial Alignment
Hours-long gap between channel acquisitions causes cell movement. Apply multi-channel registration to align channels spatially.

### Leakage Correction
Zero out tumor signal in healthy tissue channel to remove fluorescence bleed-through. Enables cleaner healthy tissue segmentation.

**Rationale:**  
Corrects two major systematic noise sources, creating more consistent inputs for modeling.

<!-- TODO: Add before/after images showing registration and leakage correction -->

---

## Stage 5: Cross-Channel Bootstrapping

**Approach:**  
Apply tumor model to leakage-corrected healthy tissue channel. Structures look morphologically similar after preprocessing, enabling transfer.

**Rationale:**  
**Segment the second class (healthy tissue) without manual labels.** Exploits morphological similarity post-correction to bootstrap healthy tissue segmentation from tumor model.

**Critical difference from Stage 2:** Not expanding the tumor dataset - this generates healthy tissue labels by leveraging corrected data structure.

**Output:** Pseudo-labels for healthy tissue class

---

## Stage 6: Retrain with Expanded Dataset

**Approach:**  
Retrain nnU-Net using **only high-quality results from Stage 5** (filtered bootstrapped healthy tissue labels) combined with earlier curated tumor labels.

**Output:** Improved two-class model trained on larger, cleaner dataset

---

## Stage 7: Post-Processing

**Approach:**
- Largest component selection (remove spurious small blobs)
- Hole filling + active contours (refine boundaries)

**Rationale:**  
Corrects systematic artifacts from aggressive preprocessing. Active contours use edge information to improve boundary accuracy.

---

## Stage 8: Two-Class Model

**Approach:**  
Train unified model on fully registered, leakage-corrected multi-channel inputs. Outputs simultaneous healthy tissue + tumor predictions.

**Rationale:**  
Joint modeling captures spatial relationships between classes (e.g., tumors cluster near healthy tissue boundaries, relevant for invasion analysis).

**Custom Loss Function:**  
Asymmetric loss (1.1× weighting on tumor class) + confusion-aware loss. See `loss_design.md` for detailed rationale.

**Brief rationale:**  
Tumor false negatives are catastrophic for downstream invasion analysis (they are likely to be identified as invading cells in the next stage → orders of magnitude error). False positives acceptable (slight search zone shrinkage ~few %). Loss function prioritizes recall over precision. (See [Loss Design](loss_design.md) for full analysis)

**Output:** Production-quality two-class segmentation masks

<!-- TODO: Add final segmentation results showing both classes -->

---

## Next Stage: Invasion Segmentation

With clean healthy tissue and tumor masks, the next challenge is identifying **individual invading tumor cells** within healthy tissue. See `invasion_segmentation.md` for approach.

---

## Key Design Principles

- **Iterative refinement over single-shot training:** Bootstrap → train → filter → correct data → retrain
- **Fix data issues before modeling:** Registration and leakage correction improve all downstream steps
- **Leverage cross-channel similarity:** Corrected channels enable model transfer
- **Task-aligned loss design:** Optimize for downstream requirements (invasion analysis), not pixel accuracy
- **Quality over quantity:** Filter pseudo-labels aggressively to prevent error propagation

---

## Lessons Learned

- **Data correction enables bootstrapping:** Leakage correction made cross-channel transfer possible
- **Iterative expansion beats one-shot labeling:** Building training set incrementally with QC gates prevents error amplification
- **Downstream constraints dictate design:** Asymmetric loss directly addresses invasion analysis requirements
- **Visual QC is non-negotiable:** Automated metrics miss systematic failure modes visible to human inspection
