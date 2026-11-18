# BridgeRecon Align

Code, intermediate data and visualization demos for the article **"Multi-Sensor Verification of Rapid UAV Bridge Reconstruction: Onboard RTK and Laser Ranging Expose a Depth-Scale Bias Hidden by Image Consistency"** (submitted to *Sensors*).

BridgeRecon Align is a training-free workflow with two modules.

- **Reconstruction module.**
  - A global COLMAP structure-from-motion (SfM) solution provides the camera poses, intrinsic parameters and sparse points.
  - A frozen [VGGT](https://github.com/facebookresearch/vggt) model predicts depth for overlapping twelve-photograph chunks.
  - The depth of each chunk is placed in the global cameras and calibrated with sparse training anchors, by a pooled scale per chunk and a bounded scale per photograph. No chunk-to-chunk registration is needed.
- **Verification module.** It works on three levels:
  1. image consistency on held-out SfM tracks: cross-view reprojection discrepancy with support and success rates, and seam consistency between chunks;
  2. agreement with withheld RTK camera positions;
  3. agreement with the onboard laser rangefinder.

  A few laser ranges also suffice to correct the metric scale.

<p align="center"><img src="docs/figures/chunk_calibration.png" width="88%"></p>

## Repository layout

| Folder | Content |
|---|---|
| `pipeline/` | Reconstruction and evaluation code: SfM, data partition, VGGT chunks, depth calibration, held-out evaluation, seams, COLMAP MVS and Metashape comparisons, RTK and rangefinder checks |
| `revision/` | Scripts of the additional controls in the revised article, as run on an NVIDIA L4 GPU server: reconstruction without the checking features, prior and focal sweeps, camera and timing controls, flight-geometry subsets, confidence and anchor scans, staged timing and MVS reproduction |
| `data/` | Intermediate data, described in `data/README.md`: aggregate result tables, camera-fold and point-split identifiers, settings, per-chunk and per-image depth scales, per-image range comparison |
| `demos/` | Visualization demos that run on `data/` alone |
| `docs/figures/` | Figures produced by the demos |

## Demos

The demos need only Python 3.10 or later with NumPy and Matplotlib.

```
pip install numpy matplotlib
python demos/plot_chunk_calibration.py
python demos/range_scale_correction.py
python demos/plot_range_correction.py
python demos/make_figures_7_11.py
```

| Demo | Shows | Article |
|---|---|---|
| `plot_chunk_calibration.py` | Pooled chunk scales along the route and per-image scale ratios | Figure 4e |
| `range_scale_correction.py`, `plot_range_correction.py` | Scale factor estimated from a few laser ranges and tested on held-out photographs | Section 3.8, Table S18 |
| `make_figures_7_11.py` | Seam consistency of the depth variants; focal-length and prior sensitivity of the depth-to-range bias | Figures 7 and 11 |

<p align="center"><img src="docs/figures/range_correction.png" width="62%"></p>
<p align="center"><img src="docs/figures/figure11_camera_constraints.png" width="88%"></p>
<p align="center"><img src="docs/figures/figure7_seam_consistency.png" width="88%"></p>

## Full workflow

The full workflow needs the survey photographs, which are not public.

Requirements:
- An NVIDIA GPU with 8 GB of memory. The article used an RTX 4060 Laptop GPU and an NVIDIA L4.
- Python 3.10 with PyTorch 2.7.1 + CUDA 11.8 and pycolmap 3.11.1 (`requirements.txt`).
- [COLMAP](https://colmap.github.io/) 3.13 with CUDA; set `COLMAP_EXE` if it is not on `PATH`.
- VGGT code and weights, which are not part of this repository and are covered by the VGGT License:
  - clone `https://github.com/facebookresearch/vggt` into `pipeline/vendor/vggt` (commit `a288dd0f14786c93483e45524328726ab7b1b4ce`);
  - download `model.pt` of `facebook/VGGT-1B` (revision `860abec7937da0a4c03c41d3c269c366e82abdf9`, SHA-256 `d15bf50a8615c8225ed48b51ea5cac673d82442ec0309036df555a053253afe0`) to `pipeline/weights/model.pt`.
- Set `BRIDGERECON_IMAGES` to the folder of original photographs. They need DJI XMP metadata with RTK and rangefinder tags.

All scripts in `pipeline/` write to `pipeline/runs/<stage>_<timestamp>_<id>/`, each with a `report.json`. Run them from `pipeline/` in this order:

1. **Global SfM** on 1600×1200 derivatives: `full_route_survey.py` (`resume_full_route_mapping.py` continues a saved mapping).
2. **Camera configurations and data partition**:
   - `prepare_global_hybrid.py`: hash-based split into calibration anchors and held-out evaluation tracks.
   - `run_navigation_prior_ba.py`: position-prior bundle adjustment for C2, C4 and C5.
   - `run_rtk_weighted_ba.py`: RTK-weighted and fixed-focal configurations C6–C8.
   - `export_hybrid_model_anchors.py`: anchors for a camera control.
3. **VGGT depth** for overlapping chunks: `run_vggt_route_chunks.py`, with model loading in `vggt_probe.py`.
4. **Depth calibration and held-out evaluation** of variants V0–V3: `run_global_hybrid.py`, with `global_hybrid_geometry.py`, `route_chunk_geometry.py`, `similarity.py`, `geometry_core.py` and `local_validation.py`.
5. **Seams and summaries**: `audit_full_hybrid_seams.py`, `bridge_roi_audit.py` and `summarize_full_hybrid.py`.
6. **COLMAP MVS**:
   - build and fuse: `triangulate_fixed_route.py`, then `dense_saved_route.py` (`--width 1600` or `518`), then `audit_native_depth_maps.py` and `fuse_depth_tiles.py`, with `colmap_common_grid.py` for the common 518-pixel grid;
   - evaluate: `evaluate_native_mvs_checking.py` or `evaluate_native_mvs_attempt.py`.
7. **Metashape**: `validate_metashape_adapter.py` and `evaluate_metashape_checking.py`.
8. **Timing on the RTX 4060 laptop**: `revision_resource_benchmark.py`.
9. **Analyses**:
   - `revision_v041_analysis.py`: anchor baselines, pair geometry, bootstrap.
   - `revision_v041_lrf_check.py`: RTK camera centres and rangefinder distances.
   - `revision_v042_rtk_analysis.py`, `revision_v042_mvs518_analysis.py` and `audit_image_lens_metadata.py`.

The additional controls of the revised article are in `revision/`, which has its own README. Several analysis scripts name the authors' run folders as constants (for example `global_hybrid_20261019_205203_28fc26c6`); replace them with the folders produced by your runs.

## Portability changes from the authors' working copy

- The COLMAP executable is read from `COLMAP_EXE` (default `colmap`), and the photographs from `BRIDGERECON_IMAGES` (default `images/`).
- `work_schedule.py` is a permissive stand-in for a local quiet-hours guard that never affected any computation.
- `evaluate_metashape_checking.py` hashes the internal evaluation-protocol note only if that file exists.
- In `revision/`:
  - the project root of the cloud runtime can be set with `--root` or `BRIDGERECON_ROOT`, and checkpoint copies go to `BRIDGERECON_CHECKPOINTS`;
  - `production_v3_case.py` takes the calibration folder as an argument;
  - queue, checkpoint and cloud-storage helpers are not included.

## Data availability

The UAV photographs, the exact trajectory, the SfM database, the model weights and the full prediction arrays are not public. The imagery is available from the corresponding author upon reasonable request and with the permission of the infrastructure management authority. `data/` contains the intermediate data released with the article.

## Citation

Citation details will be added on publication.

## License

The code is released under the MIT License (`LICENSE`), and the data in `data/` under CC BY 4.0 (`data/LICENSE`). VGGT, COLMAP, pycolmap and the other dependencies are covered by their own licenses.
