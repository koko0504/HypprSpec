# Physiology-Guided Cultivar-Invariant Hyperspectral Learning for Grapevine Stress Diagnosis

This repository investigates whether hyperspectral signatures can distinguish **healthy grapevine leaves from leaves affected by Flavescence dorée (FD)** when the model is evaluated on grape cultivars it did not see during training. It also examines whether auxiliary physiological targets—chlorophyll, flavonol, and the Nitrogen Balance Index (NBI)—improve this cross-cultivar generalization, and whether cultivar-adversarial learning can reduce cultivar-specific information in the learned representation.

The repository contains two linked notebooks. **Project 1 produces the saved spectra, split definitions, model predictions, and evaluation artifacts that Project 2 loads.** Project 2 is principally an aggregation and reporting stage: it reconstructs the held-out-cultivar predictions, computes and compares evaluation metrics, and writes tables and plots. Run Project 1 first, then make its output dataset available to Project 2.

## Research questions

- **RQ1 — cultivar invariance:** Can stress signatures learned using several cultivars generalize to a completely unseen cultivar?
- **RQ2 — physiological supervision:** Does auxiliary supervision using chlorophyll, flavonol, and NBI improve stress recognition across cultivars?

## Task and validation design

- **Classification target:** Healthy versus Flavescence dorée (binary labels `0` and `1` in the evaluation notebook).
- **Validation:** Leave-One-Cultivar-Out (LOCO). All leaves belonging to one cultivar form a held-out test fold; the remaining cultivars are used for training. The evaluation notebook rebuilds train/test sample indices from the saved `splits.json` and the leaf/cultivar metadata.
- **Samples:** The saved LOCO evaluation covers 110 leaves across six held-out cultivars. The notebook reports the following number of leaves and FD-positive leaves per cultivar:

| Cultivar | Held-out leaves | FD-positive leaves |
|---|---:|---:|
| Chardonnay | 23 | 12 |
| Colombard | 15 | 10 |
| Duras | 12 | 9 |
| Fer | 15 | 10 |
| Gamay | 30 | 19 |
| Mauzac | 15 | 10 |
| **Total** | **110** | **70** |

Because some individual cultivar folds may contain only one class, per-cultivar macro-F1 can be undefined; the notebook labels such values as `NaN`. Accuracy is reported per cultivar, while the combined LOCO evaluation reports metrics on the pooled held-out predictions.

## Project 1 → Project 2 handoff

The notebooks are a sequential workflow rather than two independent analyses:

1. Run [`final-year-project-1(1).ipynb`](final-year-project-1%281%29.ipynb) to create the upstream project outputs.
2. In Project 2, set `PREV` to the directory containing those outputs. The notebook's configured Kaggle example is `/kaggle/input/datasets/keya0112/outputs-of-previous`; its default also uses `WORK = PREV`.
3. Run [`final-year-project-2(1).ipynb`](final-year-project-2%281%29.ipynb). It loads the processed spectra and labels, classic-model results and predictions, neural-network predictions, saved within-domain neural results, and LOCO split definitions from Project 1's output directory.
4. Project 2 computes the comparison tables and figures in the writable `/kaggle/working` directory. Its input directory is treated as read-only.

The exact artifact names read by Project 2 are:

| Project 1 artifact | How Project 2 uses it |
|---|---|
| `spectra_trimmed.npz` | Provides `Xt` (trimmed spectra), `wlt` (wavelengths), `leaf` (leaf identifiers), `y` (binary labels), `variety` (cultivar), and `chl`, `flav`, `nbi` physiological measurements. |
| `results_classical.pkl` | Provides classic-model test predictions/probabilities and within-domain results. Project 2 selects each model's best preprocessing option from `raw`, `sg`, and `snv` based on its saved inner-CV results. |
| `nn_final.pkl` | Provides neural-network prediction probabilities for each model variant and seed on LOCO test samples. |
| `within_nn.json` | Provides within-domain neural-network reference scores used in the domain-generalization comparison. |
| `splits.json` | Provides the saved LOCO fold definitions. Project 2 maps the saved leaf IDs to rows in `spectra_trimmed.npz` to reconstruct train/test indices. |

Project 2 verifies the alignment through the saved leaf IDs and uses the same held-out sample indices across model predictions. The saved files are therefore part of the scientific workflow: keep them together, preserve their names and matching versions, and point `PREV` at the correct Project 1 output folder. Do not silently substitute predictions or splits from a different run.

## Modeling approach

### Classical spectral baselines

Project 2 evaluates the saved results for five classical models:

- **PLS-DA**
- **Linear SVM**
- **RBF SVM**
- **Random Forest**
- **L2-regularized Logistic Regression** (`LogReg-L2`)

For each classical model, the notebook selects the best preprocessing variant from the saved inner-CV results. The variants are raw spectra, Savitzky–Golay (`sg`) processed spectra, and standard normal variate (`snv`) spectra. Project 2 does not refit these models; it loads their saved probabilities and predictions from `results_classical.pkl`.

### Neural models

The saved neural predictions are evaluated for four variants, each with five seeds (`0` through `4`):

| Code | Notebook model name | Design represented by the variant |
|---|---|---|
| M0 | `1D-CNN` | Stress-classification CNN baseline. |
| M1 | `CNN+Physiology` | Stress CNN with physiological auxiliary supervision. |
| M2 | `CNN+Cultivar-GRL` | Stress CNN with a cultivar-adversarial head using a gradient-reversal learning setup. |
| M3 | `PhyloSpec-DG` | Combined physiology and cultivar-adversarial variant. |

The `nn_final.pkl` predictions are stored per seed and are averaged by Project 2 to produce the final probability for each neural model. The README describes evaluation of the saved models; network construction and training occur upstream in Project 1, not in the Project 2 reporting cells.

## Evaluation and code approach in Project 2

The reporting notebook loads the arrays and saved result objects, creates a single set of LOCO test indices, and evaluates the classic and neural predictions on those same held-out leaves. Its principal metrics are:

- **Macro-F1** — equal-weighted performance across the two classes.
- **Balanced accuracy**.
- **Matthews correlation coefficient (MCC)**.
- **Sensitivity and specificity**.
- **ROC AUC**, calculated from FD probabilities when both classes are present.
- **Within-domain F1** — a saved reference score from the within-domain validation results.
- **Generalization gap** — within-domain F1 minus pooled LOCO macro-F1. A larger positive gap indicates a larger drop on unseen cultivars.

For the pooled LOCO macro-F1, Project 2 calculates a **95% bootstrap confidence interval** using 2,000 resamples (seed 42), retaining resamples that contain both classes. For neural models, probabilities are averaged over the five saved seeds before classification at the 0.5 threshold. The notebook also writes per-cultivar accuracy/F1 and summarizes stability by mean cultivar accuracy, worst-cultivar accuracy, the identity of the worst cultivar, and standard deviation across cultivars.

The code uses NumPy and pandas for array/table handling; scikit-learn metric functions for classification metrics; SciPy's exact binomial test and custom paired resampling/McNemar calculations for model comparisons; and Matplotlib/Seaborn for plots. The notebooks run in a Python 3 Jupyter/Kaggle environment.

## Results recorded in the Project 2 notebook

The following values are the saved notebook outputs, rounded to three decimals. The `within-F1` and `LOCO-F1` columns are macro-F1; confidence limits are the bootstrap CI for pooled LOCO F1. The gap is `within-F1 − LOCO-F1`.

| Model | Within F1 | LOCO F1 | 95% LOCO F1 CI | Gap | Balanced accuracy | MCC | ROC AUC |
|---|---:|---:|---:|---:|---:|---:|---:|
| PLS-DA | 0.965 | 0.909 | 0.846–0.963 | 0.056 | 0.898 | 0.823 | 0.980 |
| Linear SVM | 0.969 | **0.930** | 0.874–0.972 | 0.038 | **0.923** | **0.862** | **0.985** |
| RBF SVM | **0.970** | 0.891 | 0.828–0.949 | 0.079 | 0.889 | 0.783 | 0.963 |
| Random Forest | 0.935 | 0.890 | 0.827–0.949 | 0.045 | 0.884 | 0.782 | 0.937 |
| LogReg-L2 | 0.968 | 0.911 | 0.853–0.962 | 0.057 | 0.909 | 0.822 | 0.984 |
| 1D-CNN | 0.856 | 0.836 | 0.762–0.906 | 0.020 | 0.841 | 0.672 | 0.898 |
| CNN+Physiology | 0.854 | 0.836 | 0.760–0.906 | 0.019 | 0.841 | 0.672 | 0.875 |
| CNN+Cultivar-GRL | 0.854 | 0.827 | 0.752–0.898 | 0.027 | 0.834 | 0.656 | 0.857 |
| PhyloSpec-DG | 0.854 | 0.836 | 0.760–0.906 | 0.019 | 0.841 | 0.672 | 0.886 |

**Summary of the recorded comparison:** Linear SVM has the highest pooled LOCO macro-F1 (0.930) and the best balanced accuracy, MCC, and ROC AUC among the listed models. The classical models achieve higher pooled LOCO F1 than the saved neural variants in this run. Among the neural models, CNN+Physiology and PhyloSpec-DG match the stress-only 1D-CNN LOCO macro-F1 (0.836); this particular result does **not** show an improvement in the primary LOCO F1 from adding physiology. CNN+Cultivar-GRL has lower pooled LOCO F1 (0.827) than the stress-only baseline. These are observations from the saved evaluation outputs, not proof that the methods cannot help under other data, training, or validation settings.

### Per-cultivar held-out accuracy

The notebook's per-cultivar table reports accuracy for each cultivar held out as a test fold:

| Held-out cultivar | PLS-DA | Linear SVM | RBF SVM | Random Forest | LogReg-L2 | 1D-CNN | CNN+Physiology | CNN+Cultivar-GRL | PhyloSpec-DG |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Chardonnay | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 |
| Colombard | 0.933 | 0.933 | 0.933 | 0.933 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 |
| Duras | 1.000 | 1.000 | 1.000 | 0.917 | 1.000 | 0.917 | 0.833 | 0.917 | 0.833 |
| Fer | 0.933 | 1.000 | 1.000 | 1.000 | 1.000 | 0.467 | 0.400 | 0.467 | 0.400 |
| Gamay | 0.767 | 0.833 | 0.800 | 0.800 | 0.833 | 0.800 | 0.800 | 0.800 | 0.800 |
| Mauzac | 1.000 | 0.933 | 0.733 | 0.800 | 0.800 | 0.867 | 1.000 | 0.800 | 1.000 |

The per-cultivar view matters: aggregate results can conceal difficult transfers. In this run, the neural variants perform notably less well on the held-out **Fer** fold than the classic baselines, while Chardonnay is classified perfectly by all listed models. That heterogeneity is why the notebook also exports cultivar-level scores and stability summaries rather than relying only on one pooled score.

### Model-comparison test

The notebook records the best classical model as **Linear SVM** and the best neural model, by LOCO F1, as **1D-CNN**. Their paired comparison on held-out predictions has:

- Mean paired macro-F1 difference (Linear SVM minus 1D-CNN): **0.095**.
- Bootstrap 95% interval: **0.028 to 0.164**.
- Exact McNemar discordant counts: **12** samples where Linear SVM is correct and 1D-CNN is wrong, versus **2** samples where Linear SVM is wrong and 1D-CNN is correct.
- Exact McNemar p-value: **0.013**.

The interval and discordant counts describe this saved test set and should be interpreted in the context of the sample size, cultivar grouping, and the evaluation design. They are not a substitute for additional independent validation.

## Files produced by Project 2

Project 2 writes these reports and figures to `/kaggle/working` (or to the path assigned to `OUT`):

| Output | Contents |
|---|---|
| `table4.csv` | Pooled model comparison: within-domain F1, LOCO F1 and confidence interval, generalization gap, balanced accuracy, MCC, and ROC AUC. |
| `table_per_cultivar.csv` | Held-out leaf counts, FD counts, accuracy, and macro-F1 by cultivar and model. |
| `table_cultivar_stability.csv` | Mean and worst-cultivar accuracy, worst cultivar, and standard deviation across cultivars. |
| `fig_per_cultivar_accuracy.png` | Heatmap of accuracy for each model on each held-out cultivar. |
| `table_rq1_gap.csv` | Within-domain versus LOCO performance, gap, bootstrap limits, and whether the LOCO interval excludes the within-domain score. |
| `table_rq1_model_comparison.csv` | Paired best-classical versus best-neural comparison, bootstrap difference interval, and exact McNemar test summary. |
| `fig_rq1_within_vs_loco.png` | Figure comparing within-domain F1 with LOCO F1 and its 95% interval. |

Project 2 also prints the selected best classical preprocessing variant, model names, number of LOCO test leaves, fold counts, per-cultivar scores, and the comparison results in the notebook output.

## Metadata and data validation — required

**Before running the complete pipeline, verify all expected metadata column names against `description.csv`.** This requirement is explicit in the project statement. Do not assume the field names or units for labels, cultivar, chlorophyll, flavonol, or NBI. Check that the metadata is aligned to the spectral samples and the leaf IDs, then confirm the label encoding and the split membership.

In the saved Project 2 artifact, metadata/arrays are accessed using the keys `Xt`, `wlt`, `leaf`, `y`, `variety`, `chl`, `flav`, and `nbi`. These are **keys in the processed `.npz` artifact**, not necessarily the literal source-column names in `description.csv`. Verify the source headers and the mapping that Project 1 uses to create these arrays before rerunning the upstream notebook. Also check missing physiological values, duplicates, sample alignment, and cultivar spelling/encoding. The repository listing does not currently establish that `description.csv` or the original dataset is checked in; provide it from the authorized dataset source as needed and respect its license/access terms.

## Running the notebooks

### Kaggle

1. Open and run Project 1: [Kaggle — final-year-project-1](https://www.kaggle.com/code/keya0112/final-year-project-1).
2. Confirm that Project 1 has produced the five handoff artifacts listed above. Publish/add the output directory as a Kaggle dataset (the notebook's example input folder is `outputs-of-previous`) or otherwise make that directory available as a Project 2 input.
3. Open Project 2: [Kaggle — final-year-project-2](https://www.kaggle.com/code/keya0112/final-year-project-2).
4. Check the `PREV` and `WORK` path cell. Set `PREV` to the actual Project 1 output dataset directory; set `WORK` to a writable location. The default example is Kaggle-specific and will need adjustment outside Kaggle.
5. Run Project 2 from top to bottom and retrieve the report CSVs and PNG figures from the writable output directory.

### Local Jupyter

The notebooks use Python 3. A local run requires the dataset, `description.csv`, and the Project 1 output files in locations matching the paths configured in the notebooks. Install the packages imported by the notebooks (including NumPy, pandas, scikit-learn, SciPy, Matplotlib, and Seaborn); install any additional Project 1 training dependencies from its import cells. Adjust Kaggle-specific `/kaggle/input/...` and `/kaggle/working/...` paths before execution. No pinned `requirements.txt` is currently provided, so exact package versions are not specified by the repository.

## Reproducibility and interpretation

- Keep Project 1 spectra, model predictions, preprocessing results, and split definitions from the same run together.
- Do not use any held-out cultivar samples to fit preprocessing, select features, tune parameters, or choose checkpoints. Such leakage would compromise the intended unseen-cultivar evaluation.
- Preserve the five neural seed predictions and average probabilities as Project 2 does; do not average hard labels instead.
- Report both pooled and per-cultivar performance, and retain the LOCO confidence interval and cultivar counts.
- The values above are the outputs embedded in the current Project 2 notebook. Re-executing the pipeline with different data, software versions, random seeds, or artifacts may produce different results.

## Repository contents

- [`final-year-project-1(1).ipynb`](final-year-project-1%281%29.ipynb) — upstream project: data/model workflow and output artifacts consumed downstream.
- [`final-year-project-2(1).ipynb`](final-year-project-2%281%29.ipynb) — downstream evaluation, LOCO aggregation, model comparison, tables, and plots.
- [`Objective1 (1).pdf`](Objective1%20(1).pdf) — project objective/reference document.

## Data and citation

Add the original dataset citation, spectral acquisition details, and any project/publication citation here when available. Ensure dataset licensing, attribution, and access conditions are respected.