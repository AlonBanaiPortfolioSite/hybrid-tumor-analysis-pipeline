# System B: Loss Function Design

## Problem Context

Standard segmentation losses (Dice, cross-entropy) equally penalize false positives and false negatives. For invasion analysis, **these errors have drastically different consequences.**

---

## Downstream Task: Invasion Analysis

**Goal:** Identify individual tumor cells invading healthy tissue.

**Method:** 
1. Use healthy tissue mask to define "healthy tissue region"
2. Look at tumor channel intensity within this region
3. Segment invading cells using intensity statistics

**Critical dependency on tumor segmentation accuracy:**
- Invading cells are identified as "tumor-like signal within healthy tissue region"
- Tumor segmentation errors propagate to invasion detection

---

## Error Impact Analysis

### False Negative (FN): Tumor classified as healthy tissue

**Consequence:**  
- Tumor region incorrectly included in "healthy tissue" search zone
- **Downstream sees tumor signal in healthy region → misclassified as invading cells**
- Original tumor is orders of magnitude larger than invading cell population
- **Result:** Massive overestimation of invasion (orders of magnitude error)

### False Positive (FP): Healthy tissue classified as tumor

**Consequence:**  
- Healthy tissue region excluded from invasion search zone
- Slightly shrinks search area
- **Result:** Small error (~few % reduction in search zone size)

---

## Trade-off Analysis

**FN impact:** Orders of magnitude error
**FP impact:** Few percent error

**Conclusion:** Prioritize recall (minimize FN) over precision (minimize FP) for tumor class.

---

## Loss Function Modifications

Tested two independent modifications to address tumor FN:

### Modification 1: Weighted Cross-Entropy on Tumor Class

**Standard CE:** Equal penalty for all misclassifications

**Modified CE:** Increased penalty when ground truth is tumor

$$L_{CE} = 
\begin{cases}
w \cdot \text{CE}(pred, gt) & \text{if } gt = \text{tumor} \\
\text{CE}(pred, gt) & \text{if } gt = \text{healthy}
\end{cases}$$

**Hypothesis:**  
Increased penalty applies when ground truth is tumor:
- If model predicts tumor and GT is tumor: Slightly increased penalty (not 100% confident)
- If model predicts healthy and GT is tumor: Significantly increased penalty (FN - bad!)
- If GT is healthy: Loss unchanged regardless of prediction

**Expected behavior:** In uncertain boundary cases, model biased toward predicting healthy tissue (lower-risk prediction given asymmetric penalty).


### Modification 2: Confusion-Aware Loss (Tumor FN Penalty)

**Approach:** Explicit additional penalty specifically for tumor FN (tumor→healthy misclassification)

**Implementation:** Extra loss term added when model predicts healthy but ground truth is tumor


---

## Experimental Results

**Weighted Cross-Entropy:**

| Weight | Tumor FN Rate | Tumor FP Rate |
|--------|---------------|---------------|
| 1.0 (baseline) | 5.15% | 0.132% |
| **1.1** | **3.88%** | **0.187%** |
| 1.25 | 4.64% | 0.129% |
| 1.5 | 4.48% | 0.160% |
| 3.0 | 7.68% | 0.113% |

**Confusion-Aware Loss:**

| Weight | Tumor FN Rate | Tumor FP Rate |
|--------|---------------|---------------|
| 0.0 (baseline) | 5.15% | 0.132% |
| 0.05 | 5.49% | 0.113% |
| 0.075 | 5.35% | 0.116% |
| 0.1 | 4.21% | 0.149% |
| 0.11 | 6.40% | 0.125% |
| 0.15 | 5.13% | 0.141% |
| 0.2 | 5.77% | 0.112% |

**Selected:** Weighted CE with 1.1× tumor class weight (FN=3.88%, FP=0.187%). 
note that this configuration get the lowest FN at the price sgnificant FP rise while Confusion-Aware Loss weight 0.1 get the second lowest FN with minor FP rise. 
However, visual inspection of representative subset showed less critical error for downstream task at Weighted Cross-Entropy weight 1.1.

![Custom loss results](/images/loss_comparison.png)
*Tumor FN rate vs FP rate for different confusion penalty weights.*

---

