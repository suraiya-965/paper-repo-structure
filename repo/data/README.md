# Data

## Files
- `processed/train_data.csv` — 18,037 rows (includes synthetic augmented
  images — see below)
- `processed/val_data.csv` — 2,519 rows (natural distribution, no augmentation)
- `processed/test_data.csv` — 2,538 rows (natural distribution, no augmentation)
- Final total: 23,094 images across **12 classes** (raw merged dataset
  was 26,391 images before cleaning — see pipeline breakdown below):
  `Corn_Common_Rust, Corn_Gray_Leaf_Spot, Corn_Healthy, Corn_Leaf_Blight,
  Rice_Bacterial_Leaf_Blight, Rice_Brown_Spot, Rice_Healthy, Rice_Leaf_Blast,
  Wheat_Brown_Rust, Wheat_Healthy, Wheat_Loose_Smut, Wheat_Yellow_Rust`

Each CSV has two columns: `filepath`, `label`. `filepath` currently points
to `/content/split_dataset/...` (a Colab path) — update this to a relative
path before anyone outside the original Colab environment tries to use
these CSVs to load images.

## ⚠️ How the train set actually got to 1,500/class (correcting an earlier guess)
Earlier I guessed the training set was **capped/undersampled** to reach
its class balance. Having now traced the notebook's own printed output
cell-by-cell, that guess was wrong — **it's the opposite: oversampling
via synthetic augmentation**, and the scale of it is significant enough
that it should be stated explicitly in the paper.

Full pipeline, reconciled against the notebook's own printed numbers:

| Step | Count | Notes |
|---|---|---|
| Raw merged (23 Kaggle sources) | 26,391 | ~2,200/class, from your message |
| − corrupted/junk files | −21 | `.DS_Store` etc. |
| − duplicate images | **−9,557** | up to **76.0%** of `Wheat_Loose_Smut` alone was duplicates |
| = after cleaning | 16,834 | now **imbalanced** — 526 (`Wheat_Loose_Smut`) to 2,197 (`Rice_Healthy`) |
| 70/15/15 split (`splitfolders`, seed=42) | → train/val/test | val/test kept as-is from here on (natural imbalance) |
| **Train-only augmentation** to 1,500/class | 18,037 final train | see below |

Duplicate rate as a share of the **raw merged 26,391**: 9,557/26,391 =
**36.2%** — so that figure from the earlier paper-review conversation
was accurate after all (it's against the raw total, not the final one).

**The augmentation step** (`RandomFlip`, `RandomRotation(0.3)`,
`RandomZoom(0.25)`, `RandomContrast(0.25)`, `RandomBrightness(0.25)`,
`RandomTranslation(0.1,0.1)`) runs **only on the already-split train
folder** — val/test are never touched, so there's no leakage from this
step. But the scale varies hugely by class, and for the worst case,
most of the final training images for that class are synthetic:

| Class | Real images (after split) | Synthetic added | Synthetic share of final 1,500 |
|---|---|---|---|
| `Wheat_Loose_Smut` | 368 | **1,132** | **75.5%** |
| `Corn_Healthy` | 813 | 687 | 45.8% |
| `Corn_Leaf_Blight` | 800 | 700 | 46.7% |
| `Wheat_Brown_Rust` | 854 | 646 | 43.1% |
| `Rice_Bacterial_Leaf_Blight` | 1,368 | 132 | 8.8% |
| `Rice_Healthy` | 1,537 | 0 (already over target) | 0% |
| *(remaining 6 classes)* | 800–1,257 | 243–687 | 16–46% |

**This needs to be stated explicitly in the paper's methodology.** It's
a defensible and fairly common technique for class imbalance, but a
reviewer would reasonably want to know that ~75% of `Wheat_Loose_Smut`'s
training images are synthetic augmentations of only 368 real source
photos — this affects how much genuine visual diversity the model saw
for that class, and is worth a sentence in the limitations section
alongside the existing single-split and small-XAI-sample caveats.

It also still means the reported 70/15/15 split ratio (78.1%/10.9%/11.0%
once augmentation is included) needs the same wording fix — e.g. "images
were split 70/15/15, after which the training set was augmented to
1,500 images per class to address class imbalance; validation and test
sets retain the natural post-cleaning class distribution."

## Source
The dataset merges images from multiple Kaggle sources across the 3 crop
categories (Rice, Corn, Wheat). You sent 25 links; after removing exact
duplicates and one non-dataset link, here are the **23 unique dataset
sources**, grouped by crop:

**Rice (12 sources)**
- https://www.kaggle.com/datasets/sikhaok/riceleafdisease-5class-balanced
- https://www.kaggle.com/datasets/slygirl/rice-leaf-disease-with-segmentation-labels
- https://www.kaggle.com/datasets/tiswan14/rice-leaf-disease-classification-dataset
- https://www.kaggle.com/datasets/yusufmurtaza01/rice-leaf-diseases
- https://www.kaggle.com/datasets/anshulm257/rice-disease-dataset
- https://www.kaggle.com/datasets/jay7080dev/rice-plant-diseases-dataset
- https://www.kaggle.com/datasets/sdeysocial/rice-leaf-disease-image-samples
- https://www.kaggle.com/datasets/shayanriyaz/riceleafs
- https://www.kaggle.com/datasets/vbookshelf/rice-leaf-diseases
- https://www.kaggle.com/datasets/nizorogbezuode/rice-leaf-images
- https://www.kaggle.com/datasets/dedeikhsandwisaputra/rice-leafs-disease-dataset

**Corn (6 sources)**
- https://www.kaggle.com/datasets/unknown6874/corn-leaf-disease-dataset
- https://www.kaggle.com/datasets/galaxionzero000/corn-leaf-diseases-dataset
- https://www.kaggle.com/datasets/yasirahmad0810/consolidated-corn-dataset
- https://www.kaggle.com/datasets/ndisan/corn-leaf-disease
- https://www.kaggle.com/datasets/abdelrahmanemad2199/corn-or-maize-leaf-disease-dataset
- https://www.kaggle.com/datasets/kamal01/crop-prediction *(general crop dataset — check whether corn subset only was used)*

**Wheat (4 sources)**
- https://www.kaggle.com/datasets/freedomfighter1290/wheat-disease
- https://www.kaggle.com/datasets/khanaamer/wheat-leaf-disease-dataset
- https://www.kaggle.com/datasets/olyadgetch/wheat-leaf-dataset
- https://www.kaggle.com/datasets/kushagra3204/wheat-plant-diseases

**Multi-crop / other (1 source)**
- https://www.kaggle.com/datasets/musfiqurtuhin/bangladeshi-crops-disease-dataset-bcdd

### Removed from your list
- **Duplicate**: `anshulm257/rice-disease-dataset` was sent twice — kept once above.
- **Duplicate**: `sdeysocial/rice-leaf-disease-image-samples` was sent twice
  (same page, different `?select=` query parameter) — kept once above,
  query param dropped since it just pre-selects a folder on the page.
- **Not a dataset**: `kaggle.com/code/emrearslan123/corn-leaf-disease-detection-with-resnet-pytorch`
  is a Kaggle *Code* notebook (someone's ResNet training tutorial), not a
  dataset page — removed from the source list. If a specific dataset was
  actually pulled via that notebook's own data source, link that instead.

### ⚠️ Still needs a fix in the paper text
The manuscript's data section reads: *"It was constructed by combining
multiple separate, publicly available crop-specific datasets from
Kaggle, **one for each crop**."* The phrase "one for each crop" reads as
exactly 3 total (1 per crop × 3 crops), which conflicts with itself
("multiple separate... datasets") and with reality — there are **23**.
Suggested fix: *"combining multiple crop-specific datasets from Kaggle
for each of the three crops"* (drop "one for each crop"). This matters
for reproducibility: a reader trying to rebuild `agrivision_dataset`
from "one dataset per crop" would get a very different (much smaller)
dataset than the one actually used.

Licenses for each source aren't listed here — check each Kaggle page's
license field before including in a public Data Availability statement,
since not all Kaggle datasets share the same license.
