# [Repo Name] — Code for [Paper Title]

## Structure

```
notebooks/
├── 00_data_preparation/        # merge, dedup, corrupt-file removal, 70/15/15 split,
│                                # CSV generation — see note below
├── 01_single_models/           # individual base-model training — 6/6 complete:
│                                # DenseNet121, LeViT, Swin, ViT, ResNet50, EfficientNetB0
├── 02_shared_predictions/       # prerequisite notebook(s) — see note below
├── 03_ensembles/                 # 27 canonical ensemble-combination notebooks
│   └── archive/                  # superseded/draft/duplicate versions, kept for history
├── 04_evaluation/                 # accuracy, calibration, error analysis, t-SNE,
│                                  # PR curves, radar chart, per-class breakdown
└── 05_significance_testing/       # separate 3-part pipeline — see note below

data/
├── processed/  # train_data.csv, val_data.csv, test_data.csv (added)
└── raw/        # empty — raw agrivision_dataset not uploaded, see note below
results/         # empty — figures/tables not yet extracted from notebooks (optional, see note)
```

## `00_data_preparation/data_merge_dedup_split.ipynb`
This single notebook covers the whole pipeline described in the paper's
methodology: loads the merged dataset (`agrivision_dataset`), removes
corrupted image files, finds and removes duplicates via hashing, then
does a 70/15/15 train/val/test split with `splitfolders` (seed=42), and
writes out the CSVs (`train_data.csv`, `val_data.csv`, `test_data.csv`)
that every other notebook in this repo reads from. **This is the file
that produces the "36.2% duplicates removed" figure discussed earlier**
— worth double-checking the exact percentage against what's printed in
this notebook's output before it goes in the paper, since I haven't
verified that number against this code's actual output.

Note: the notebook starts from an already-merged `agrivision_dataset`
folder rather than showing the merge of the 3 original Kaggle sources —
if that merge step happens in a separate notebook/script, send it along
too; otherwise the paper's data availability section should probably
point to wherever `agrivision_dataset` itself is hosted (Kaggle/Zenodo),
since it isn't reconstructable from this notebook alone.

## Why `02_shared_predictions/` exists
`part1_swin_predictions_generator.ipynb` isn't itself an ensemble experiment —
it's a prerequisite. Its own markdown explains why: `tfswin` (Swin
Transformer) needs Keras 3's normal mode, while `LeViT` needs
`TF_USE_LEGACY_KERAS=1`; running both in one Colab session crashes with
`AttributeError: 'KerasTensor' object has no attribute 'ndim'`. So Swin's
predictions are generated once here and cached to Drive, then reused by
any ensemble notebook that includes Swin. Worth keeping this note (or an
expanded version of it) in the paper's or repo's implementation notes —
it explains a real engineering constraint a reviewer/reproducer would hit.

## `03_ensembles/` — canonical vs. archive
I classified files as canonical or archived based on two signals:
1. **Explicit labeling** — e.g. `swin_resnet50_clean.ipynb`'s own first
   cell literally says "Clean, correctly-ordered version" — so its
   un-labeled sibling was archived as `swin_resnet50_unclean.ipynb`.
2. **File size / "simple" naming** — files named `*_simple*` were
   consistently much smaller (~20-40 KB vs. ~160-230 KB for the full
   notebooks with val-set weight tuning, classification reports, and
   confusion matrices) — treated as earlier drafts and archived.

**Please double check this before submission** — I inferred canonical
status from internal evidence, I don't know which run actually produced
the numbers in the paper's tables. If any archived file is actually the
"real" one, say so and I'll swap it back.

## Coverage vs. paper's "33 configurations" — ✅ complete
The paper's 33 configurations = **6 single models + 27 ensembles**, not
27 ensembles alone. Both counts now match exactly:
- Single models: **6/6** — ResNet50, DenseNet121, EfficientNetB0, LeViT, Swin, ViT
- Ensembles: **9 pairs + 18 triples = 27/27**

The two triples I earlier flagged as "missing" — `{LeViT,Swin,ViT}` and
`{ResNet50,DenseNet121,EfficientNetB0}` — are **not** part of the
reported 33 after all; they were presumably excluded by design (possibly
the LeViT+Swin Keras conflict made the 3-way combo not worth the extra
prediction-cache complexity). No further ensemble notebooks needed for
coverage.

## `03_ensembles/swin_vit_resnet50__PROPOSED_MODEL_full_pipeline.ipynb` — read this first
**This is the single most important notebook in the repo.** It's not just
an ensemble notebook — it's the paper's entire main-result pipeline in
one file. Cell-by-cell:

| Cells | Content |
|---|---|
| 1–14 | Standard ensemble pipeline (same pattern as the other 26 ensemble notebooks): load Swin + ViT + ResNet50, simple-average ensemble, accuracy |
| 25–32 | Val-set weight tuning (leakage-free), classification report, confusion matrix |
| **Cell A** | **Grad-CAM++ setup** — explicitly noted as "CORRECTED" to use pre-softmax logits (avoids vanishing gradients when softmax is saturated near 1.0, which is nearly every prediction here) |
| **Cell B** | Runs Grad-CAM++ + bounding box on **one uploaded image at a time** |
| (cell 37) | **McNemar's test**: best single model (ViT, 98.38%) vs. the proposed ensemble (98.86%) |
| **Cell C** | **Occlusion Sensitivity** — independent, gradient-free cross-check, reusing the same models |
| **Cell D** | **Spearman rank correlation** between Grad-CAM++ and Occlusion Sensitivity on that one image — this is the paper's core XAI cross-verification contribution |

So the "missing" XAI and cross-verification notebooks from earlier
weren't missing — they're Cells A–D of this file. I've renamed it with
`__PROPOSED_MODEL_full_pipeline` so it doesn't get mistaken for just
another one of the 27 ensemble notebooks when someone browses the folder.

**One thing to resolve**: Cells A–D work on one uploaded image per run
(`files.upload()`), so the paper's n=25 aggregate statistic (mean
Spearman ρ = 0.339 across 25 images) had to come from running this cell
repeatedly and collecting the per-image results somewhere else. See
"Still missing" below.

## `04_significance_testing/` — a separate pipeline, and an important finding
This is a 3-part pipeline (`part1_predictions` → `part2_levit_predictions`
→ `part3_mcnemar_analysis`) that runs McNemar's test comparing **one
specific ensemble against all 6 single-model baselines individually**.

**`part3_mcnemar_analysis.ipynb`'s own output reveals which ensemble is
the paper's actual proposed/headline model: Swin + ViT + ResNet50, at
98.86% accuracy** — this matches the accuracy figure from the earlier
reference-checking conversation, so this is almost certainly *the*
result reported as the paper's main contribution. Worth double-checking
that `03_ensembles/swin_vit_resnet50__PROPOSED_MODEL_full_pipeline.ipynb`
is flagged/labeled clearly
as the "proposed model" in the repo and paper, since it's easy to lose
track of which of 27+ notebooks is the headline one.

**Possible duplication to check**: `04_evaluation/mcnemar_visual_analysis.ipynb`
(from an earlier batch) and `04_significance_testing/part3_mcnemar_analysis.ipynb`
both do McNemar analysis. They may be doing different things (one all-pairs
comparison table, one visualization) or one may be an earlier draft of the
other — please check and let me know if one should move to `archive/`.

## Still missing / open questions
- **n=25 aggregation — reopened.** I re-checked the proposed-model
  notebook cell by cell (41 cells total). Cell B (`files.upload()`) and
  Cell D (`compare_gradcam_occlusion` + `top_k_overlap`) only **define**
  the comparison functions and run them **once per manual upload** —
  there's no loop, no list/DataFrame collecting results across runs, and
  nothing saved back to Drive from this step. So if the n=25 rollup
  really happened inside this notebook, it isn't visible in the code
  itself — it'd mean re-running Cell B+D by hand 25 times and copying
  each printed ρ value out manually (e.g. into a spreadsheet or directly
  into the paper draft), which is a legitimate way to work but means
  **the n=25 mean/aggregation doesn't exist anywhere as code** — only
  the per-image method does. Worth mentioning in the paper's
  reproducibility/implementation notes so a reviewer doesn't go looking
  for an aggregation script that isn't there.
- ~~Raw dataset source~~ — ✅ resolved, see `data/README.md`. 23 unique
  Kaggle sources across Rice/Corn/Wheat (2 duplicates and 1 non-dataset
  link removed from what was sent). **But this surfaced a real
  discrepancy**: the paper's text says "3 Kaggle sources," which is off
  by a wide margin from the actual 23 — almost certainly "3" was meant
  as *3 crop categories*, not 3 dataset downloads. Needs a wording fix
  in the methodology section before submission, or a reader trying to
  reproduce the dataset from "3 links" will end up with a much smaller,
  wrong dataset.

## ✅ Privacy cleanup done
714 cells across 57 notebooks had Google Colab's `executionInfo.user`
metadata embedded (`displayName` + internal Google `userId`) — this is
added automatically by Colab every time a cell runs and normally isn't
visible in the notebook UI, but it's plain text inside the `.ipynb` JSON
and shows up if anyone inspects the raw file or views it on GitHub via
"raw". Two names appeared repeatedly: **Amanuzzaman Riyon** and
**Suraiya Jahan** — presumably the paper's authors, running cells under
their own Google accounts.

This has been stripped from every notebook in this repo (code, markdown,
and cell outputs/results are untouched — only the `executionInfo.user`
field was removed). No emails were found embedded anywhere.

**If you still have the originals in Google Drive/Colab**, note that
they still contain this metadata — only the copies in this repo are
cleaned. Also worth a quick manual look at any `results/*.txt` or `.csv`
output files saved by these notebooks (e.g. `ensemble_swin_resnet50_report.txt`),
since file-level outputs aren't covered by this scan.

## Reproducing results
*(To fill in: environment setup, run order, which notebook produces each
paper table/figure.)*

## Other small things found in this pass
- **Empty folders won't survive `git add`** — Git doesn't track empty
  directories. Added a `.gitkeep` placeholder to `results/tables/`,
  `results/figures/`, and `data/raw/` so the folder structure survives
  the first commit even before you add real files to them.
- **Added `03_ensembles/archive/README.md`** — a short warning that
  numbers in archived/draft notebooks shouldn't be cited, so nobody
  accidentally pulls an accuracy figure from a superseded run.
- **Scanned for secrets/API keys/tokens** (Google API keys, OpenAI-style
  keys, GitHub tokens, hardcoded passwords) and phone numbers across
  every notebook, script, and markdown file — **none found**.
- **`requirements.txt` added** — reconstructed from every `!pip install`
  line and printed version found across all 58 notebooks. Only
  `numpy`/`pandas` had confirmed pinned versions (printed in a
  notebook's own output); everything else is unpinned since the
  notebooks never printed exact versions for those packages — they just
  used whatever Colab had installed at run time. Good enough to get
  someone started outside Colab, but not a guaranteed-identical
  environment. See the file's own header comment for details.
- Confirmed all 58 `.ipynb` files are valid/parseable JSON — no corrupted
  files from the upload process.
