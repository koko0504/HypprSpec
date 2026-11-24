# Physiology-Guided Cultivar-Invariant Hyperspectral Learning for Grapevine Stress Diagnosis

This repository contains the notebooks for a grapevine hyperspectral-imaging study focused on diagnosing **Flavescence dorée (FD) stress** while testing whether learned signatures transfer to grapevine cultivars that were not seen during training.

The work is organized as two linked project stages: **Project 1 produces an intermediate output that is used as input to Project 2**. The notebooks are the executable record of the analysis; consult them for the exact transformations, file paths, and saved artifact names used in a particular run.

## Research questions

1. **Cultivar generalization:** Can stress signatures learned from several grapevine cultivars generalize to a completely unseen cultivar?
2. **Physiological supervision:** Does auxiliary supervision using chlorophyll, flavonol, and the Nitrogen Balance Index (NBI) improve stress recognition across cultivars?

## Study design

- **Primary task:** binary classification of **Healthy** versus **Flavescence dorée**.
- **Generalization evaluation:** **Leave-One-Cultivar-Out (LOCO)** validation. In each fold, all samples from one cultivar are withheld for testing; training and model selection use the other cultivars. This tests transfer to a cultivar absent from the training set.
- **Spectral input:** hyperspectral measurements, with their exact representation and preprocessing determined by the notebook pipeline.
- **Auxiliary physiological targets:** chlorophyll, flavonol, and NBI.
- **Cultivar information:** used to define LOCO folds and, for adversarial models, to train a cultivar-prediction head. Cultivar identity should not be supplied as an ordinary predictive feature to the primary stress classifier if the goal is cultivar-invariant stress recognition.

## Models in the study

| Model | Role in the comparison |
|---|---|
| Logistic Regression | Linear, classical baseline |
| Linear SVM | Linear-margin baseline |
| RBF SVM | Nonlinear kernel baseline |
| Random Forest | Tree-ensemble baseline |
| Stress-only 1D CNN | Deep-learning baseline trained for stress classification |
| CNN + physiology | Adds auxiliary prediction/supervision for chlorophyll, flavonol, and NBI |
| CNN + cultivar adversarial head | Encourages a representation that is less informative about cultivar while predicting stress |
| Full physiology + cultivar-adversarial CNN | Combines physiological supervision with cultivar-adversarial learning |

The CNN comparisons are intended to isolate the contribution of the auxiliary physiological tasks and cultivar-adversarial objective. The exact network layers, loss weights, optimizer, training schedule, and scoring metrics should be read from the notebooks rather than inferred from this overview.

## Project workflow

1. **Project 1 — upstream analysis and output generation.** Run `final-year-project-1(1).ipynb` to perform the first project stage and generate its intermediate output.
2. **Project 2 — downstream stress-diagnosis analysis.** Run `final-year-project-2(1).ipynb` using the output produced by Project 1. Project 2 builds on that upstream result; it is not intended to be run as an independent replacement for Project 1.
3. **Compare models under LOCO.** Evaluate each model on the held-out cultivar fold(s), aggregate the results consistently, and compare the baseline, physiology-supervised, and cultivar-adversarial approaches.

> **Important:** The notebook excerpts available in this repository view do not expose enough of the cells to reliably state the exact intermediate filename, dataset source, preprocessing sequence, or numerical results. Use the notebooks to confirm these implementation details before reproducing a run, and update this section with the artifact name and exact handoff once verified.

## Metadata and data checks — required before a full run

**Verify the metadata column names against `description.csv` before running the complete pipeline.** Do not assume that cultivar, disease/stress label, chlorophyll, flavonol, or NBI fields have the names expected by the code. Confirm the actual headers, map them explicitly to the pipeline's expected variables, and check that:

- cultivar values are present and consistently encoded for LOCO splitting;
- Healthy and Flavescence dorée labels are mapped consistently;
- physiological targets have the expected units and valid numeric values;
- metadata rows align with the corresponding spectra/samples;
- missing values and duplicate or mismatched sample identifiers are handled deliberately.

The repository listing currently does not establish that `description.csv` is included. If it is external to the repository, obtain it from the project's dataset source and document its location and access requirements before executing the notebooks. Avoid committing restricted or sensitive source data.

## Repository contents

- [`final-year-project-1(1).ipynb`](final-year-project-1(1).ipynb) — Project 1, the upstream notebook whose output is consumed by Project 2.
- [`final-year-project-2(1).ipynb`](final-year-project-2(1).ipynb) — Project 2, the downstream analysis for the linked project workflow.
- [`Objective1 (1).pdf`](Objective1%20(1).pdf) — project objective/reference document.

## Running the notebooks

The notebooks are designed for a Python 3 Jupyter environment and can be opened locally or run in an environment such as Kaggle. Because the precise imports, data locations, and generated artifact paths must be taken from the notebook cells, use this order:

1. Make the dataset and required metadata available to the runtime.
2. Inspect `description.csv` and verify/map its column names before running the complete pipeline.
3. Open and run `final-year-project-1(1).ipynb` from top to bottom.
4. Confirm that Project 1 completed and identify the intermediate output it created.
5. Set Project 2 to read that exact output, then run `final-year-project-2(1).ipynb`.
6. Record the runtime environment, data version, random seeds (if set), LOCO fold assignments, and results when reporting or comparing experiments.

For exact package requirements, follow the import cells in the notebooks. A dedicated dependency file and a short configuration section documenting dataset paths and output locations would improve reproducibility.

## Evaluation and reporting

Report results for each held-out cultivar as well as an aggregate across LOCO folds. Include the evaluation metrics actually calculated by the notebooks, the fold-level sample counts, and any uncertainty or variability across cultivars. Keep all preprocessing, normalization, feature selection, and model tuning inside each fold using training-cultivar data only; fitting these steps on held-out-cultivar samples would leak information and make the generalization estimate optimistic.

For the physiology and adversarial comparisons, report the same folds and stress-classification metrics as the baseline models. Clearly state how physiological missing values are treated, how multi-task losses are combined, and how the cultivar-adversarial objective is implemented. No numerical performance claims are included here because verified result values were not available in the provided notebook excerpts.

## Current scope and reproducibility notes

This README describes the stated research objective and model/validation plan. It does not claim that every listed model, metric, or comparison has already completed successfully. Before treating the work as a reproducible pipeline, document the dataset citation/access instructions, verified metadata headers, exact Project 1 → Project 2 artifact handoff, preprocessing details, software dependencies, and final LOCO results.

## Citation and data provenance

Add the dataset citation and any relevant project/publication citation here when available. Ensure that dataset licensing and access conditions are respected.
