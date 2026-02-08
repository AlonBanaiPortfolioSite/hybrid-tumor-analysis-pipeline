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

Tested two independent loss modifications to reduce tumor class false negatives.

### Modification 1: Weighted Cross-Entropy on Tumor Class

Increased CE penalty when ground truth is tumor class.

**Tested configurations:**
- Default (w=1.0): Tumor FN=5.15%, FP=3.36%
- Tumor 1.1×: Tumor FN=3.88%, FP=4.78%
- Tumor 1.25×: Tumor FN=4.64%, FP=3.31%
- Tumor 1.5×: Tumor FN=4.48%, FP=4.09%
- 3× Tumor: Tumor FN=7.68%, FP=2.93%

![Weighted CE results](/images/weighted_ce_results.png)
*Tumor FN rate vs FP rate for different weight values. Each point is one configuration.*

---

### Modification 2: Confusion-Aware Loss

Explicit penalty for tumor→healthy misclassification.

**Tested configurations:**
- Default: Tumor FN=5.15%, FP=3.36%
- Conf 0.05: Tumor FN=4.71%, FP=4.09%
- Conf 0.075: Tumor FN=5.49%, FP=2.90%
- Conf 0.1: Tumor FN=5.35%, FP=2.96%
- Conf 0.11: Tumor FN=4.21%, FP=3.81%
- Conf 0.15: Tumor FN=6.40%, FP=3.21%
- Conf 0.2: Tumor FN=5.13%, FP=3.60%
- 3× Organoid: Tumor FN=7.68%, FP=2.93%

![Confusion-aware loss results](/images/confusion_loss_results.png)
*Tumor FN rate vs FP rate for different confusion penalty weights.*

---

