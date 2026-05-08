# MRI Registration QC: Pairwise Group Comparison

An automated quality-check pipeline for **multi-image MRI registration comparison**.  
Given a set of NIfTI images, the app computes pairwise registration similarity metrics
for all upper-triangular image pairs and produces both per-pair QC figures and
group-level metric matrices.
This app is especially useful for comparing repeated-session FA maps or other scalar
MRI-derived maps across subjects, sessions, or processing outputs.

---

## Author
**Gabriele Amorosino**  
(email: [gabriele.amorosino@utexas.edu](mailto:gabriele.amorosino@utexas.edu))

---
## Description
The pipeline performs:
1. **Multi-image input parsing** — A list of input images is read from `config.json`
   under the `images` field.
2. **Pairwise upper-triangular comparison** — For `N` input images, the app computes
   registration QC for each unique pair only:
   ```text
   image_0 vs image_1
   image_0 vs image_2
   image_1 vs image_2
   ...

The reverse comparisons are not repeated because the resulting scalar metrics are
treated as symmetric.

3. Image loading & canonicalisation — Images are reoriented to RAS+ canonical
    orientation. 4-D images are reduced to their temporal mean before metric computation.
4. Resampling — For each pair, the second image is resampled into the voxel grid
    of the first image when the two grids differ, using affine-based trilinear
    interpolation.
5. Metric computation — For each pair, the following voxel-based similarity
    metrics are computed:

Metric	Abbreviation	Interpretation
Normalised Mutual Information	NMI	> 1, higher is better
Normalised Cross-Correlation	NCC	∈ [-1, 1], closer to 1 is better
Mean Squared Error	MSE	≥ 0, lower is better
Structural Similarity Index	SSIM	∈ [-1, 1], closer to 1 is better
Dice coefficient	Dice	∈ [0, 1], closer to 1 is better
Jaccard index	Jaccard	∈ [0, 1], closer to 1 is better

6. Optional threshold-derived masks — If thr_mask is provided, binary masks are
    created from both images and used to compute Dice and Jaccard overlap metrics.
    * A single value, e.g. 0.5, means:

image > 0.5

    * Two values, e.g. 0.5,1.0, mean:

0.5 < image < 1.0

    This is useful for scalar maps such as FA, where a threshold like 0.5,1.0 can
    define high-FA white matter regions.
7. Per-pair QC figures — For each image pair and anatomical plane, a QC figure is
    produced with:
    * Fixed image
    * Moving image resampled into fixed space
    * Checkerboard + fixed-image edge overlay
    * Absolute difference map
8. Threshold mask overlap figures — When thr_mask is used, the app also produces
    mask-overlap figures for each anatomical view. These figures show seven internal
    representative slices and use the following color convention:

Color	Meaning
Green	Overlap / true positive
Red	Mismatch / false positive or false negative
Black	Background

9. Group-level metric matrices — After all pairwise comparisons are complete, the
    app aggregates the scalar metrics into group-level pairwise matrices, one matrix
    per metric.
    Example outputs include:

matrix_ncc.png
matrix_nmi.png
matrix_mse.png
matrix_ssim.png
matrix_dice.png
matrix_jaccard.png

10. Brainlife outputs — All generated PNG files are collected into product.json
    for display in the brainlife.io UI. A separate figures/ folder is also created
    with copied figures and an images.json manifest.

⸻

## Requirements

To run the app, you only need one of:

* Singularity
* Apptainer

All required software dependencies are included in the container image, including:

* Python ≥ 3.8
* nibabel
* numpy
* scipy
* matplotlib
* jq

The app also uses helper scripts to generate:

* product.json
* figures/images
* figures/images.json

⸻

## Usage

Running on Brainlife.io

1. Go to Brainlife.io￼.
2. Search for the app, for example:

MRI Registration QC: Pairwise Group Comparison

3. Click the Execute tab.
4. Select multiple NIfTI images to compare.
5. Configure optional parameters:
    * checkerboard_tiles
    * thr_mask
6. Submit the job to view the pairwise QC report and group-level metric matrices.

⸻

Running Locally

1. Clone the repository:

git clone https://github.com/gamorosino/app-mri-registration-qc-multiple.git
cd app-mri-registration-qc-multiple

2. Prepare a config.json file:

{
    "images": [
        "path/to/session01_FA.nii.gz",
        "path/to/session02_FA.nii.gz",
        "path/to/session03_FA.nii.gz"
    ],
    "checkerboard_tiles": 8,
    "thr_mask": [0.5, 1.0]
}

3. Run the pipeline:

bash ./main

⸻

## Configuration

Key	Type	Default	Description
images	array of strings	—	List of NIfTI images to compare. At least two images are required.
checkerboard_tiles	integer	8	Number of checkerboard tiles per axis in the overlay figure.
thr_mask	array / number / null	null	Optional threshold used to create binary masks for Dice/Jaccard overlap. Use [LOW] or [LOW, HIGH].

⸻

Example Configuration

Pairwise FA comparison with lower and upper threshold

{
    "images": [
        "sub-01_ses-01_FA.nii.gz",
        "sub-01_ses-02_FA.nii.gz",
        "sub-01_ses-03_FA.nii.gz"
    ],
    "checkerboard_tiles": 8,
    "thr_mask": [0.5, 1.0]
}

This computes Dice/Jaccard using:

0.5 < FA < 1.0

⸻

Pairwise comparison with only lower threshold

{
    "images": [
        "sub-01_ses-01_FA.nii.gz",
        "sub-01_ses-02_FA.nii.gz"
    ],
    "checkerboard_tiles": 8,
    "thr_mask": [0.5]
}

This computes Dice/Jaccard using:

FA > 0.5

⸻

Pairwise comparison without Dice/Jaccard threshold masks

{
    "images": [
        "image_01.nii.gz",
        "image_02.nii.gz",
        "image_03.nii.gz"
    ],
    "checkerboard_tiles": 8
}

In this case, NMI, NCC, MSE, and SSIM are computed, but threshold-derived Dice and
Jaccard are not produced.

⸻

## Outputs

Main outputs

Path	Description
product.json	Brainlife product file embedding generated PNGs as base64
figures/images/	Folder containing copied PNG figures
figures/images.json	Manifest describing generated figures
output/group_metrics.json	JSON file containing image labels, pairwise metrics, and metric matrices
output/input_labels.json	JSON file storing labels derived from Brainlife metadata or filenames

⸻

Per-pair outputs

For each unique image pair, the app creates a folder:

output/pair_XX_YY/

where XX and YY are the image indices.

Each pair folder may contain:

Path	Description
metrics.json	Pairwise NMI, NCC, MSE, SSIM, quality label, and optional Dice/Jaccard
qc_axial.png	4-panel QC figure — axial view
qc_coronal.png	4-panel QC figure — coronal view
qc_sagittal.png	4-panel QC figure — sagittal view
mask_overlap_axial.png	Threshold mask overlap figure — axial view
mask_overlap_coronal.png	Threshold mask overlap figure — coronal view
mask_overlap_sagittal.png	Threshold mask overlap figure — sagittal view

⸻

Group-level matrix outputs

Path	Description
output/matrix_ncc.png	Pairwise NCC matrix
output/matrix_nmi.png	Pairwise NMI matrix
output/matrix_mse.png	Pairwise MSE matrix
output/matrix_ssim.png	Pairwise SSIM matrix
output/matrix_dice.png	Pairwise Dice matrix, if thr_mask is provided
output/matrix_jaccard.png	Pairwise Jaccard matrix, if thr_mask is provided

⸻

Labeling

When run on Brainlife, the app attempts to derive matrix labels from input metadata:

subject_session

For example:

ss01_ses-01
ss01_ses-02
ss01_ses-04

If subject/session metadata is unavailable, the app falls back to the input filename.

⸻

## Notes

* The app computes only the upper-triangular pairwise comparisons.
* Matrix outputs are filled symmetrically for easier visualization.
* Pairwise visual QC figures are generated with one image treated as fixed and the
    other as moving.
* Scalar metrics such as NCC, NMI, MSE, SSIM, Dice, and Jaccard are treated as
    symmetric for the group-level matrices.
* Dice and Jaccard are only produced when thr_mask is provided.

⸻

## Citation

If you use this app in a publication, please cite:

* Hayashi, S., Caron, B. A., Heinsfeld, A. S., … & Pestilli, F. (2024).
    brainlife.io: a decentralized and open-source cloud platform to support
    neuroscience research. Nature Methods, 21(5), 809–813.
* Cox, R. W. (1996). AFNI: software for analysis and visualization of functional magnetic resonance neuroimages.
  Computers and Biomedical research, 29(3), 162-173.

⸻

