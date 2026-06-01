# fix_rbc_release_tr.py

A patch script for correcting TR metadata and regenerating downstream functional derivatives from the initial [ReproBrainChart (RBC)](https://github.com/ReproBrainChart) data release.

## Background

The initial RBC functional derivatives release contains a bandpass filtering bug: during one-step spatial normalization/resampling, the TR value in the NIfTI header was silently coerced to `1.0 s`, rather than preserving the true acquisition TR. All bandpass filtering downstream of resampling was therefore applied using the wrong TR, shifting the effective passband in a dataset-dependent way.

This script corrects that by:

1. Reading the correct TR from the native-space `desc-preproc_bold.nii.gz` header (or from `--tr-override`).
2. Patching the template-space `desc-head_bold.nii.gz` header in-place (in a scratch directory) to restore the correct TR.
3. Re-running nuisance regression + bandpass via AFNI `3dTproject`, using the original raw regressor files.
4. Re-running nuisance regression without bandpass (used as input to ALFF/fALFF).
5. Recomputing downstream metrics: ALFF, fALFF, ReHo (at `sm6`/`smZstd`/`zstd` variants), atlas-based mean timeseries, and Pearson/partial connectivity matrices.

Only the bandpass TR bug is addressed; JSON sidecars are not regenerated.

## Requirements

- Python ≥ 3.12
- [`uv`](https://github.com/astral-sh/uv) (recommended) — the script uses PEP 723 inline metadata to self-provision its dependencies
- AFNI (available to the NiWrap runner — see `--runner`)

The script's inline dependencies are pinned and provisioned automatically when run via `uv run`:

```
rbc @ git+https://github.com/childmindresearch/rbc.git@7cfa758
nibabel >= 5.3.3
nilearn >= 0.10.4
numpy >= 2.0
```

> **Note for local development:** `uv run scripts/fix_rbc_release_tr.py` provisions an isolated environment from the pinned `rbc` SHA and does **not** use your local editable install. To test against your working tree, run via the project venv directly: `.venv/Scripts/python scripts/fix_rbc_release_tr.py` (or `uv run --no-project ...`).

## Usage

### Standalone (no repo clone required)

```bash
uv run --script fix_rbc_release_tr.py \
  --input-dir /path/to/rbc_release \
  --output-dir /path/to/fixed
```

### From inside the cloned repository

```bash
uv run scripts/fix_rbc_release_tr.py \
  --input-dir /path/to/rbc_release \
  --output-dir /path/to/fixed
```

## Arguments

| Argument | Required | Default | Description |
|---|---|---|---|
| `--input-dir PATH` | Yes | — | Root of the downloaded RBC release (containing `sub-*` folders). |
| `--output-dir PATH` | Yes* | — | Where to write the corrected parallel BIDS-derivatives tree. *Not required with `--dry-run` or `--verify`. |
| `--participant-label LABEL [...]` | No | all | Restrict processing to one or more subjects (with or without the `sub-` prefix). |
| `--bandpass F_LOW F_HIGH` | No | `0.01 0.1` | Bandpass cutoffs in Hz. |
| `--tr-override FLOAT` | No | auto-detected | Override auto-detection and use this TR (in seconds) for all runs. |
| `--fwhm FLOAT` | No | `6.0` | Smoothing kernel FWHM in mm for the `sm6` metric variant. |
| `--runner {auto,local,docker,podman,singularity}` | No | `auto` | NiWrap runner for AFNI `3dTproject`. |
| `--work-dir PATH` | No | system temp | Parent directory for the auto-cleaned scratch folder. Point at a roomy disk for large releases. |
| `--skip-metrics` | No | off | Only regenerate the cleaned BOLD and bandpassed regressors; skip ALFF/fALFF/ReHo/timeseries/connectivity. |
| `--overwrite` | No | off | Regenerate outputs even when they already exist. |
| `--dry-run` | No | off | List what would be written and exit without processing anything. |
| `--verify` | No | off | Inspect runs in `--input-dir` and report whether the TR bug is present; exit with a status code. |
| `--fail-fast` | No | off | Abort on the first run that raises an error instead of continuing to the next. |
| `-v` / `--verbose` | No | off | Increase log verbosity (pass twice for DEBUG). |

## Examples

**Check whether your release is affected before running the fix:**

```bash
uv run --script fix_rbc_release_tr.py \
  --input-dir /data/rbc_release \
  --verify
```

Exit codes from `--verify`: `0` = bug found (safe to fix), `1` = no runs discovered, `2` = no buggy runs found.

**Preview what would be processed without writing anything:**

```bash
uv run --script fix_rbc_release_tr.py \
  --input-dir /data/rbc_release \
  --output-dir /data/rbc_fixed \
  --dry-run
```

**Full fix for two subjects:**

```bash
uv run --script fix_rbc_release_tr.py \
  --input-dir /data/rbc_release \
  --output-dir /data/rbc_fixed \
  --participant-label sub-0001 sub-0002
```

**Full release fix, skipping metric recomputation, using a scratch disk:**

```bash
uv run --script fix_rbc_release_tr.py \
  --input-dir /data/rbc_release \
  --output-dir /data/rbc_fixed \
  --skip-metrics \
  --work-dir /scratch/rbc_tmp
```

**Override TR explicitly (e.g., when the native BOLD header is also corrupted):**

```bash
uv run --script fix_rbc_release_tr.py \
  --input-dir /data/rbc_release \
  --output-dir /data/rbc_fixed \
  --tr-override 2.0
```

## Outputs

For each functional run the script writes into a parallel BIDS-derivatives tree under `--output-dir`, mirroring the directory structure of the input release:

| File pattern | Description |
|---|---|
| `*_reg-{REG}_desc-preproc_bold.nii.gz` | Corrected nuisance-regressed + bandpass-filtered BOLD |
| `*_reg-{REG}_desc-bandpassed_regressors.1D` | Bandpass-filtered regressor matrix |
| `*_reg-{REG}_desc-{sm6,smZstd,zstd}_{alff,falff,reho}.nii.gz` | Corrected scalar metrics (3 variants × 3 metrics) |
| `*_atlas-{ATLAS}_..._desc-Mean_timeseries.1D` | Atlas-parcellated mean timeseries `(T × n_rois)` |
| `*_atlas-{ATLAS}_..._desc-PearsonNilearn_correlations.tsv` | Pearson connectivity matrix |
| `*_atlas-{ATLAS}_..._desc-PartialNilearn_correlations.tsv` | Partial correlation connectivity matrix |

`REG` is one of `36Parameter` or `aCompCor`. Atlas outputs are produced for all atlases resolvable from `rbc_resources` (AAL, Brodmann, CC200, CC400, Glasser, Harvard-Oxford, Juelich, Schaefer 200/300/400/1000, Slab907, Yeo 7/7-liberal/17/17-liberal).

Existing outputs are skipped unless `--overwrite` is passed.

## What is NOT fixed

- JSON sidecars are not regenerated.
- Only the bandpass TR bug (described above) is addressed.

## Getting help

Post questions on [NeuroStars.org](https://neurostars.org) with the `#rbc` tag, or open an issue at:  
<https://github.com/ReproBrainChart/2026FunctionalPatch/issues/new>
