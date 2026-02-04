# System A: Animal Model - Overview

## Biological Context

This system uses zebrafish larvae with implanted human tumor cells to study tumor behavior in a living organism. The goal is to understand how tumors interact with brain tissue, blood vessels, and immune cells.

## Imaging Modality

- **Channels**: 4 (brain tissue, tumor, vasculature, immune cells)
- **Dimensions**: 1024×1024×~30 slices per volume
- **Resolution**: Varies, approximately few micrometers per pixel

## Key Challenges

1. **Variable staining quality**: Not all tissue areas stain uniformly
2. **Incomplete spatial coverage**: Some regions lack clear boundaries
3. **Multi-structure segmentation**: Must segment 4 distinct structures with overlapping signals
4. **Inter-channel intensity variability**: Signal strength differs significantly across channels
5. **Optical distortions**: Some areas have occasional black spots due to optical distortions

## Clinical/Research Relevance

Enables testing drug effects on tumor-immune interactions and tumor progression in a living organism before human trials. The zebrafish model provides a controlled environment to study complex biological interactions.

## Dataset Characteristics

- **Total samples**: Several dozens (additional samples are being obtained)
- **Labeled samples**: 0 (fully unlabeled)
