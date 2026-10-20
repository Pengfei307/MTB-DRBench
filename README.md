# MTB-DRBench

**MTB-DRBench** is a benchmark for predicting **Mycobacterium tuberculosis (MTB) drug resistance** from genome mutation data. Starting from five merged mutation sources and an aggregated phenotype table, it builds a binary feature matrix **X** (51,547 isolates × 300 mutation features) and a resistance label matrix **Y** (18 drugs, S/R/unknown), and provides machine-learning (Logistic Regression, Random Forest, XGBoost) and deep-learning (MLP, CNNGWP, DeepAMR, WDNN) baselines evaluated per drug under stratified 5-fold cross-validation, including lineage-stratified and WHO resistance-layer (RR-/MDR-/pre-XDR-/XDR-TB, HR-TB, pan-susceptible) evaluations.

## Repository structure

```
MTB-DRBench/
├── data_processing/
│   ├── TB_preprocessing_pipeline.ipynb   # End-to-end pipeline: raw data -> X / Y matrices
│   ├── build_feature_matrix.py           # Build the X feature matrix (script version)
│   └── build_label_matrix.py             # Build the 18-drug Y label matrix (script version)
├── machine_learning/                     # Whole-cohort baselines
│   ├── train_logistic_regression.py      #   Logistic Regression   -> ./tb_results_LR/
│   ├── train_random_forest.py            #   Random Forest         -> ./tb_results_RF/
│   └── train_xgboost.py                  #   XGBoost               -> ./tb_results_XGB/
├── deep_learning/                        # Whole-cohort baselines
│   └── train_deep_learning_models.py     #   MLP / CNNGWP / DeepAMR / WDNN -> ./tb_results_DL/
├── machine_learning_lineage/             # Lineage-stratified ML (per-lineage evaluation)
├── deep_learning_lineage/                # Lineage-stratified DL (per-lineage evaluation)
├── requirements.txt
└── README.md
```

## Requirements

- Python ≥ 3.9 with `pip install -r requirements.txt` (pandas, numpy, matplotlib, seaborn, scikit-learn, xgboost, torch, tqdm).
- A CUDA GPU is recommended for the deep-learning models; they also run on CPU (slower).
- Preprocessing needs ~16 GB RAM and a few minutes for the full ~6 GB of raw inputs.

## Input requirements

The raw data are **not** included in this repository (size and redistribution restrictions, see *Data availability*). Place them in a data root directory with the following layout:

```
<data_root>/
├── all_mut/
│   ├── literature_mutation_all.csv     # literature-curated mutations
│   ├── china_mutation_all.csv          # China dataset mutations
│   ├── farhat_mutation_all.csv         # Farhat et al. dataset mutations
│   ├── cryptic_mutations_all.csv       # CRyPTIC catalog (comma-separated multi-mutation strings)
│   ├── extra_all_mutations.csv         # supplementary samples (pre-formatted)
│   ├── phenotype.csv                   # phenotype long table
│   └── gene_drug_tier1(original).csv   # 33 resistance-associated genes -> drug mapping
└── drug/                               # optional: paginated per-drug phenotype exports (data*.csv)
```

File schemas:

- **Mutation files** (literature / china / farhat / extra): `uniqueid, gene, mutation, ..., gene_position, codes_protein, indel_length, ...`
- **cryptic_mutations_all.csv**: `library_name, LINEAGE_NAME, MUTATION, uniqueid` — the `MUTATION` field may hold comma-separated `gene_mutation` strings, which the pipeline splits.
- **phenotype.csv**: `sra_accession, drug_abbr, binary_result (S/R), mic_result, source`
- **Lineage-stratified runs** additionally require `X_feature_matrix_final_with_lineage_final.csv` (the final X plus a `lineage` column, produced by external lineage calling) and the aligned `Y_drug_labels_final_with_lineage.csv`.

## Execution instructions

**1. Preprocessing (raw data → X / Y)**

Notebook (recommended):

```bash
# default data root is ../new_data_pipeline; override with an env var if needed
export TB_DATA_ROOT=/path/to/data_root
jupyter notebook data_processing/TB_preprocessing_pipeline.ipynb   # run all cells
```

Outputs are written to `outputs/`:

| File | Shape | Description |
|---|---|---|
| `X_feature_matrix_final.csv` | 51,547 × 302 | binary feature matrix (`uniqueid`, `source_dataset` + 300 features) |
| `Y_drug_labels_final.csv` | 51,547 × 19 | per-drug labels: S→0, R→1, unknown→NaN |
| `sample_source_info.csv` | 51,547 × 2 | sample → source-dataset mapping |
| `Y_gene_drug_intermediate.csv` | 51,547 × 16 | intermediate gene–drug association matrix (15 drugs; not the final labels) |

Script alternative: place the raw CSVs in the working directory and run
`python data_processing/build_feature_matrix.py` followed by
`python data_processing/build_label_matrix.py` (they read/write files in the current directory).

**2. Train and evaluate the models**

Put `X_feature_matrix_final.csv` and `Y_drug_labels_final.csv` in the working directory, then:

```bash
# machine learning (one model per run)
python machine_learning/train_logistic_regression.py
python machine_learning/train_random_forest.py
python machine_learning/train_xgboost.py

# deep learning (all four models, or select one)
python deep_learning/train_deep_learning_models.py --model ALL
python deep_learning/train_deep_learning_models.py --model MLP --x-file X.csv --y-file Y.csv

# lineage-stratified runs (need the lineage-augmented matrices in the working directory)
python machine_learning_lineage/train_logistic_regression_by_lineage.py
python deep_learning_lineage/train_deep_learning_models_by_lineage.py --model ALL
```

All models use stratified 5-fold cross-validation with fixed random seeds (2026 for the splits; the lineage-stratified scripts automatically reduce the fold count to `min(5, minority-class size)` when a lineage has too few isolates). Per drug, the scripts report accuracy, precision, recall, F1, AUROC, AUPRC, confusion matrices and ROC/PR curves, plus a stratified summary over the WHO resistance layers. Results are written to `./tb_results_<MODEL>/` (`./tb_results_<MODEL>_lineage/<lineage>/` for the lineage runs). Drugs with fewer than 3 isolates in either class are skipped automatically.

## Data availability

- The raw inputs total ~6 GB and are not redistributed in this repository.
- The mutation catalogs derive from the **CRyPTIC** consortium data release, the **Farhat et al.** whole-genome dataset, a **China** sequencing dataset, literature-curated mutations, and supplementary samples; the phenotype table was aggregated from per-drug database exports and online-database sample records. The CRyPTIC-derived components are subject to the CRyPTIC data use policy.
- All isolates are identified by public SRA/ENA accessions; `sample_source_info.csv` records the source dataset of every sample.
- The processed matrices and the raw input bundle are available from the corresponding author on reasonable request. *(A permanent archived version will be deposited on Zenodo and linked here upon publication.)*

## Reproducibility caveats

1. The preprocessing pipeline is fully deterministic: the notebook and `build_label_matrix.py` were executed end-to-end and their outputs (`X_feature_matrix_final.csv`, `Y_drug_labels_final.csv`, `sample_source_info.csv`) are **byte-identical** to the study matrices (verified with pandas 2.3 / numpy 2.3); `build_feature_matrix.py` is a line-for-line translation of the same logic. The notebook reads only the required columns and applies the phenotype/gene filters per source before concatenation to keep memory usage low; this is mathematically equivalent to the script implementation.
2. `phenotype.csv` is treated as a given input: its assembly from the per-drug exports and the supplementary online-database samples is not scripted in this repository.
3. The lineage-augmented matrix (`X_feature_matrix_final_with_lineage_final.csv`) is produced by an external lineage-calling step and is only consumed, not regenerated, by the lineage-stratified scripts.
4. Cross-validation splits and model initializations use fixed seeds (2026), but small numerical differences can arise across library versions and hardware; in particular, deep-learning results may differ slightly between CPU and GPU executions.
5. Several target drugs have very few resistant isolates; per-drug metrics for those drugs are inherently unstable, and drugs with fewer than 3 isolates per class are skipped.
6. The Y labels encode S→0 / R→1 / unknown→NaN; no imputation is performed. Only isolates with a recorded phenotype for a given drug enter that drug's evaluation.
