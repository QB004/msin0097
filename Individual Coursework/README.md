# Hypothyroid Notebook: Setup, Run, and Artefact Validation

## Project entry points
- Primary notebook: `msin0097_individual_coursework.ipynb`
- Dataset: `hypothyroid.csv`
- Reproducibility artefacts: `artifacts/pipeline_v1/`

## 1. Setup

### Windows (PowerShell)
```powershell
# Run from the project root
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install jupyterlab notebook pandas numpy matplotlib seaborn plotly missingno scikit-learn xgboost lightgbm shap joblib
```

### macOS/Linux
```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install jupyterlab notebook pandas numpy matplotlib seaborn plotly missingno scikit-learn xgboost lightgbm shap joblib
```

## 2. Run the notebook

### Interactive run
```powershell
jupyter lab
```
Open `msin0097_individual_coursework.ipynb` and run all cells from top to bottom.

### Non-interactive run
```powershell
jupyter nbconvert --to notebook --execute --inplace msin0097_individual_coursework.ipynb
```

After a successful run, these files should exist in `artifacts/pipeline_v1/`:
- `manifest.json`
- `validation_report.json`
- `split_report.json`
- `duplicate_resolution_report.json`
- `leakage_report.json`
- `model_input_summary.json`
- `train_ids.csv`, `val_ids.csv`, `test_ids.csv`
- `preprocessor.joblib`

## 3. Validate outputs using artefact files

### 3.1 Quick checks
- `validation_report.json`: `"passed": true`, no errors.
- `split_report.json`: `"passed": true`, `"coverage_ok": true`, all overlap counters are `0`.
- `manifest.json`:
  - `dataset_sha256` = `9513c8975d0e1d365e203e9196c422a16b04a3d04db256566fd7fd36b05e31e8`
  - `seed` = `42`
  - `split_sizes` = train `2158`, val `463`, test `463`
  - `row_count` = `3084`
- `model_input_summary.json`: `feature_count = 27`, transformed shapes match train/val/test sizes.

### 3.2 PowerShell validation script
```powershell
$a = "artifacts/pipeline_v1"
$manifest = Get-Content "$a/manifest.json" | ConvertFrom-Json
$validation = Get-Content "$a/validation_report.json" | ConvertFrom-Json
$split = Get-Content "$a/split_report.json" | ConvertFrom-Json
$model = Get-Content "$a/model_input_summary.json" | ConvertFrom-Json
$dup = Get-Content "$a/duplicate_resolution_report.json" | ConvertFrom-Json

$checks = [ordered]@{
  validation_passed = ($validation.passed -eq $true)
  split_passed = ($split.passed -eq $true)
  coverage_ok = ($split.coverage_ok -eq $true)
  no_id_overlap = (
    $split.id_overlap.train_val -eq 0 -and
    $split.id_overlap.train_test -eq 0 -and
    $split.id_overlap.val_test -eq 0
  )
  no_signature_overlap = (
    $split.signature_overlap.train_val -eq 0 -and
    $split.signature_overlap.train_test -eq 0 -and
    $split.signature_overlap.val_test -eq 0
  )
  dataset_hash_ok = ($manifest.dataset_sha256 -eq "9513c8975d0e1d365e203e9196c422a16b04a3d04db256566fd7fd36b05e31e8")
  seed_ok = ($manifest.seed -eq 42)
  row_count_ok = ($manifest.row_count -eq 3084)
  split_sizes_ok = (
    $manifest.split_sizes.train -eq 2158 -and
    $manifest.split_sizes.val -eq 463 -and
    $manifest.split_sizes.test -eq 463
  )
  feature_count_ok = ($model.feature_count -eq 27)
  shapes_ok = (
    $model.X_shapes.train[0] -eq 2158 -and $model.X_shapes.train[1] -eq 27 -and
    $model.X_shapes.val[0] -eq 463 -and $model.X_shapes.val[1] -eq 27 -and
    $model.X_shapes.test[0] -eq 463 -and $model.X_shapes.test[1] -eq 27
  )
  duplicate_resolution_ok = (
    $dup.rows_before -eq 3163 -and
    $dup.rows_after -eq 3084 -and
    $dup.conflict_rows_removed -eq 2 -and
    $dup.feature_duplicate_rows_removed -eq 77
  )
}

$checks.GetEnumerator() | ForEach-Object { "{0}: {1}" -f $_.Key, $_.Value }
if ($checks.Values -contains $false) { throw "Artefact validation failed." }
"Artefact validation passed."
```

## 4. Optional integrity check for input data
```powershell
(Get-FileHash "hypothyroid.csv" -Algorithm SHA256).Hash.ToLower()
```
Expected hash: `9513c8975d0e1d365e203e9196c422a16b04a3d04db256566fd7fd36b05e31e8`.
