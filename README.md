# Important RBC update: Processed Functional Data+Pipeline TR Issue
Dear RBC User Community,

We are writing to inform you of a recently identified TR metadata issue in the RBC preprocessing pipeline that affected bandpass filtering for publicly released RBC functional derivatives, with larger deviations for datasets with longer TRs. 

Specifically, during one-step spatial normalization/resampling, the TR value was reset to 1.0 second rather than preserving the true acquisition TR. As a result, bandpass filtering was applied using an incorrect TR.

## Summary 

### Unaffected / remains safe to use:

Unprocessed data (BIDS), all structural derivatives and anatomical measures, and minimally preprocessed BOLD data that precede spatial normalization/resampling.

### Can use after correcting the TR:

Spatially normalized BOLD files remain available in the RBC repositories. These are the files ending in `*space-MNI152NLin6ASym_desc-head_bold.nii.gz`. These files were affected at the metadata level: the NIfTI header contains an incorrect TR value, while the correct TR is preserved in the accompanying JSON sidecar. Users who wish to use these files should first correct the NIfTI header using tools such as nibabel’s `set_zooms()` or AFNI’s `3drefit`, using the TR from the JSON sidecar.

### Removed / avoid using from the current release:

Downstream TR-dependent derivatives computed from these spatially normalized BOLD files, including nuisance-regressed timeseries, functional connectivity matrices, ALFF/fALFF, ReHO, and related outputs, have been removed from the RBC repositories while corrected versions are prepared. These derivatives should not be treated as corrected simply by changing the TR header after the fact, because the affected temporal processing has already been applied.

### Immediate action:

Beyond removal of affected processed data from the RBC repository, we have developed a three-phase correction plan to address this issue, which is described in detail below. While fully reprocessed data are being prepared, we are releasing patch code and user guidance so that users can inspect affected files, verify TR metadata, and regenerate corrected downstream outputs from minimally preprocessed data.

### Recommendation:
If your analyses or manuscripts rely on affected TR-dependent functional derivatives, please review the guidance below. Where feasible, we encourage users to wait for corrected outputs or rerun affected steps using the patch code. If you continue with analyses based on the current release, we recommend documenting the effective filter ranges and considering sensitivity analyses, particularly for higher-TR datasets.

<img width="5252" height="3708" alt="fMRI Data Processing-2026-05-23-001419" src="https://github.com/user-attachments/assets/6f3d09d2-0b71-4184-821f-b14e83a6c8a0" />


## What Happened

The RBC preprocessing pipeline was implemented in C-PAC as a new configuration option. This work extended C-PAC to support a workflow aligned with fMRIPrep/XCP-style patterns for spatial normalization and downstream functional postprocessing, while retaining C-PAC’s broader design as a configurable platform for composing, evaluating, and comparing multiple preprocessing strategies.
Within the RBC-specific C-PAC workflow, resampled 4D NIfTI outputs generated during the one-step spatial normalization/resampling stage carried a TR value of 1.0 second rather than the original acquisition TR. Downstream bandpass filtering in the RBC workflow then used this incorrect NIfTI header value.

The BIDS sidecar metadata contained the correct TR, which made the issue more difficult to detect: the acquisition metadata were accurate, but the downstream workflow relied on the resampled NIfTI header after spatial normalization, where the TR value was incorrect. As a result, the effective bandpass range was unintentionally shifted in a dataset-dependent way based on each dataset’s true acquisition TR.
The issue was identified during benchmark testing of a NiWrap-based reimplementation of the RBC pipeline, which helped reveal the metadata propagation problem in the C-PAC-based RBC workflow. We sincerely apologize that this issue was not identified during RBC benchmarking and quality assurance efforts conducted prior to data release. Those efforts did include pipeline checks and validation, but they did not detect this specific metadata propagation issue or its impact on effective bandpass filtering (in part due to our focus on connectomes, which were less impacted by this issue). We recognize that users rely on RBC derivatives for ongoing analyses and publications, and we are strengthening validation procedures to reduce the likelihood of similar issues in future releases.

## Likely Scientific Impact
The impact of this issue depends on the acquisition TR and the analytic question.

For datasets with low TR values, particularly TRs below approximately 1 second, the effects are expected to be relatively modest because the effective passband is only marginally shifted. For example, if the intended bandpass was 0.01-0.1 Hz, a dataset with TR = 0.8 seconds (e.g., Healthy Brain Network) would have an effective bandpass of approximately 0.0125-0.125 Hz.
For datasets with higher TR values, especially TRs of 2.0 seconds or greater, the impact is more substantial because the effective passband can shift more meaningfully. For example, if the intended bandpass was 0.01-0.1 Hz, a dataset with TR = 2.0 seconds would have an effective bandpass of approximately 0.005-0.05 Hz. The effective passband for each RBC dataset is detailed below.

| RBC dataset | True TR | Effective bandpass (when filtered as TR = 1.0s) |
|:----|:---|:---|
| Nathan Kline Institute - Rockland Sample | 0.645 s | ~0.0155-0.155 Hz |
| HBN - City University of New York | 0.80 s | ~0.0125-0.125 Hz |
| HBN - Citigroup Biomedical Imaging Center | 0.80 s | ~0.0125-0.125 Hz |
| Nathan Kline Institute - Rockland Sample | 1.40 s | ~0.0071-0.071 Hz |
| HBN - Staten Island | 1.45 s | ~0.0069-0.069 Hz |
| Brazilian High Risk Cohort | 2.00 s | ~0.005-0.05 Hz |
| Nathan Kline Institute - Rockland Sample | 2.50 s | ~0.004-0.04 Hz |
| Developmental Component of the Chinese Color Nest Project | 2.50 s | ~0.004-0.04 Hz |
| Philadelphia Neurodevelopmental Cohort | 3.00 s | ~0.0033-0.033 Hz |

Initial evaluations suggest that broad connectome-level analyses may be comparatively less sensitive to the shifts in bandpass filter than those looking at frequency-sensitive measures or specific individual connections/edges. However, the degree of impact is expected to vary across datasets and analytic approaches. Depending on the subsequent harmonization approach used, cross-site analyses may also be impacted because TR-dependent differences in the effective passband can introduce additional site- or dataset-related batch effects and reduce detectability.

Preprocessing-related variability is a well-recognized feature of functional neuroimaging analyses, including across independently developed minimal preprocessing pipelines applied to the same data (Li et al., 2024, Nature Human Behaviour). Based on our initial RBC-specific evaluations, the differences introduced by this TR metadata issue appear to fall within the range of variability observed across independent preprocessing implementations for the measures examined, and in some cases (e.g., short TR data) are notably smaller. The distinction here is that the source of variance is known, attributable to a specific metadata issue, and can therefore be characterized, reported, and corrected.

For studies using affected derivatives, we recommend reporting the effective filtering range used in the analysis in addition to the intended filtering range, as described above.  Reporting effective filter ranges will improve interpretability and facilitate sensitivity analyses and comparisons across studies. 

## Correction Plan

### Phase 0: Data Access 

Impacted RBC functional derivatives have been removed from INDI while corrected versions are being prepared.

### Phase 1: Immediate User Guidance and Patch Code

We are preparing patch code and documentation for the RBC GitHub repository (no later than June 1). This will help users identify affected files, inspect TR metadata, restore the correct TR from source/BIDS metadata where appropriate, and rerun affected downstream steps from minimally preprocessed data using the intended passband. This is intended to provide an immediate path for sensitivity analyses and independent verification while the RBC team prepares complete, corrected releases.

### Phase 2: Updated fMRIPrep/XCP-D Pipeline Release

Independent of the current situation, the Penn team has been preparing an updated version of the RBC derivatives using their standard workflow, which includes recent versions of  fMRIPrep for preprocessing and XCP-D for postprocessing. These derivatives are not intended as a direct reproduction of the original C-PAC-based RBC derivative release. Rather, they represent an updated processing stream that includes several useful features, including support for surface-based spaces such as CIFTI. We will expedite the upload process and make the data available in upcoming weeks.

### Phase 3: NiWrap RBC Pipeline Release

We will complete a full re-run of affected datasets through the NiWrap-based replication of the RBC pipeline previously implemented in C-PAC, incorporating the correction for the identified TR metadata issue. This archival release will include fully regenerated derivatives across all processing stages, harmonized versioning and provenance tracking, DataLad integration, expanded QC, and metadata validation procedures. This will be available in approximately 3 months. Additionally, the NiWrap RBC pipeline will be made publicly available this summer, as a replacement to the RBC config in C-PAC.

## Responsibility and Transparency

The responsibility for validating and correcting the RBC derivatives is ours. 

We recognize that many users rely on these resources for active analyses and manuscripts. We sincerely apologize to users for any delay, worry, or increase in workload that this issue causes. Our goal is to move quickly and minimize any additional disruption to users. We aim to provide transparent guidance about what is affected, what remains usable, and when corrected resources will become available.

We appreciate your patience as we complete this correction. Please post your question on NeuroStars.org with the #rbc tag, or direct any questions to the RBC support/issues page: https://github.com/ReproBrainChart/2026FunctionalPatch/issues/new  
