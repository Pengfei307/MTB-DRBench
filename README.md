# MTB-DRBench

MTB-DRBench is a resource and benchmarking framework for evaluating genomic prediction of drug resistance in *Mycobacterium tuberculosis*. The accompanying manuscript describes an integrated collection of 51,547 WGS accessions linked to phenotypic drug susceptibility testing (pDST), 18 drug-level prediction tasks, and six composite resistance categories.

This repository contains matrix preparation and evaluation scripts for logistic regression (LR), random forest (RF), XGBoost (XGB), multilayer perceptron (MLP), wide-and-deep neural network (WDNN), CNNGWP, and a single-task adaptation of DeepAMR.

Documentation below describes commit `d0d4ba42ab235544e75ccf31601035c0f96d558e`. Manuscript figures and tables should be associated with the code version and input files used to generate them.

## Repository contents

| Directory | Purpose |
|---|---|
| `data_processing/` | Build aligned feature and phenotype matrices from prepared input tables. |
| `machine_learning/` | Evaluate LR, RF, and XGB on each drug task. |
| `deep_learning/` | Evaluate MLP, CNNGWP, WDNN, and the adapted DeepAMR model. |
| `machine_learning_lineage/` | Train and evaluate separate classical models within each lineage. |
| `deep_learning_lineage/` | Train and evaluate separate neural models within each lineage. |
| `requirements.txt` | Python dependencies used by the scripts. |

## Data availability

The current repository contains code. Raw sequencing reads, source mutation tables, `phenotype.csv`, the external feature-selection table, final input matrices, and accession manifests are not included in this commit. The manuscript identifies the original public data sources. Readers need the prepared input files to reproduce the analysis; this repository cannot currently reconstruct the complete study cohort from public accessions alone.

Before a reproducible data release, provide a persistent download location for the accession manifest, sample inclusion/exclusion records, final phenotype and feature matrices, lineage annotations, and corresponding version information. This README does not supply a dataset DOI or license that has not yet been issued.

## Installation

Use a Python environment with the packages listed in `requirements.txt`:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Install a PyTorch build appropriate for the available CPU or accelerator. Neural scripts select CUDA, then Apple MPS, then CPU when available. The dependency file currently has no version pins. For reproducible runs, record the full environment and use the software versions corresponding to the manuscript results.

Run the commands below from the repository root. File paths and output directories are defined near the beginning of each script. Reusing an output directory can replace earlier result files.

## Input matrices

| File | Required content |
|---|---|
| `X_feature_matrix_final.csv` | A `uniqueid` column followed by numeric model features. An optional `source_dataset` metadata column is removed by the training scripts. |
| `Y_drug_labels_final.csv` | A `uniqueid` column and drug columns containing `0` for susceptible, `1` for resistant, and missing values for unavailable phenotypes. |
| `X_feature_matrix_final_with_lineage_final.csv` | Feature matrix with a `lineage` column for the lineage-specific scripts. |
| `Y_drug_labels_final_with_lineage.csv` | Corresponding phenotype matrix. |

Identifiers must refer to the same isolates in both matrices. Check uniqueness, label values, and sample alignment before analysis. The lineage annotation is used to select samples and is removed from model inputs.

## Matrix preparation

`build_feature_matrix.py` expects the mutation source files listed in `FILE_CONFIG`, `phenotype.csv`, and the external gene-selection table named by `INPUT_TARGET_GENE_FILE`. Mutation files need `uniqueid` and mutation annotations; the exact columns and source-specific transformations are defined in the script. The feature builder applies gene filtering, frequency-based feature construction, and aggregation of some rare variants. It expects already prepared mutation tables; raw-read processing is outside these scripts.

`build_label_matrix.py` requires the phenotype columns `sra_accession`, `drug_abbr`, and `binary_result`, with `S`/`R` phenotype values. Conflicting and duplicate phenotype records must be resolved upstream. The pivot operation assumes that the remaining records can be aggregated without producing fractional class labels.

Run both steps in this order:

```bash
python data_processing/build_feature_matrix.py
python data_processing/build_label_matrix.py
```

The first script also writes a provisional `Y_drug_labels_final.csv` derived from its feature mapping. The second script replaces that file with pDST labels. Only the pDST-derived version should be used for the manuscript's phenotype prediction tasks. Verify that every nonmissing final label is either 0 or 1.

## Drug-level evaluation

```bash
python machine_learning/train_logistic_regression.py
python machine_learning/train_random_forest.py
python machine_learning/train_xgboost.py
python deep_learning/train_deep_learning_models.py --model ALL
```

To run one neural model with explicitly named matrices:

```bash
python deep_learning/train_deep_learning_models.py \
  --model MLP \
  --x-file X_feature_matrix_final.csv \
  --y-file Y_drug_labels_final.csv
```

The scripts evaluate independent drug-level tasks using random stratified outer folds. Missing drug labels are excluded from that task. For XGB and neural models, the first split of an inner stratified partition supplies a validation set for early stopping. The implementation does not conduct a full inner cross-validation hyperparameter search. LR and RF fit the outer training set directly. Binary predictions use a fixed threshold of 0.5.

The split seed is 2026. RF and XGB estimator seeds are 42 in this commit. Neural scripts initialize their supported random generators with 2026. Record the requested model order because it can affect subsequent random-number state.

## Evaluation within lineages

```bash
python machine_learning_lineage/train_logistic_regression_by_lineage.py
python machine_learning_lineage/train_random_forest_by_lineage.py
python machine_learning_lineage/train_xgboost_by_lineage.py
python deep_learning_lineage/train_deep_learning_models_by_lineage.py --model ALL
```

These scripts select one lineage, then train and evaluate models within that lineage. They do not measure transfer to a lineage excluded from training. Drug tasks with insufficient class counts are skipped, and the number of folds can vary in the classical scripts. Report the evaluated tasks, class counts, and folds alongside lineage-level summaries.

The lineage-aware input matrices must be supplied separately; this commit does not contain a script that constructs them from the original cohort metadata.

## Outputs and aggregation

The scripts write per-fold drug metrics, drug-level summaries, out-of-fold prediction/probability matrices, confusion matrices, and diagnostic plots to their configured output directories. Exact filenames vary by model.

Drug-level metrics are averaged over the evaluated folds. The overall row averages drug-level means. Its reported dispersion is the mean of the individual drug-level standard deviations, not the standard deviation of a pooled overall estimate. AUPRC in these scripts is computed using `average_precision_score`.

The public code does not yet include the complete scripts used to assemble every final manuscript panel and supplementary table. Preserve the per-fold outputs and the final plotting inputs with each release.

## Composite resistance categories

The implementation derives RR-TB, MDR-TB, HR-TB, pan-susceptibility, and MDR-based pre-XDR/XDR categories from individual drug calls. In this commit, pre-XDR and XDR require INH and RIF resistance. These operational categories are narrower than the WHO MDR/RR-based definitions. Eligibility depends on available phenotypes; missing calls require careful interpretation.

The scripts calculate some composite-category files during the drug-training loop, when predictions for other drugs may still be incomplete. These intermediate outputs require separate verification against the complete out-of-fold prediction matrix before use as final category-level estimates. This calculation has not been changed in the documented commit.

## Interpretation and reproducibility

The benchmark supports retrospective method comparison under the stated input representation, model implementations, and splits. External and prospective evaluations are needed to establish performance in clinical settings. The adapted DeepAMR configuration uses a single-drug task and no separate autoencoder pretraining; comparisons should identify this adaptation explicitly.

For each published run, retain the input checksums, accession list, feature definitions, fold assignments, dependency versions, code commit, per-fold metrics, and figure-generation inputs. Raw data availability, processed data availability, and availability of evaluation code should be stated separately.

## Citation and licensing

Please cite the MTB-DRBench manuscript using its final bibliographic record when available. No publication DOI or repository license is asserted by this draft README. Reuse of third-party data remains subject to the conditions of the original sources.

Questions about the implementation can be submitted through the repository's GitHub Issues page.
