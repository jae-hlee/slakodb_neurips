# SlakoNet DB

Results of applying SlakoNet (a machine-learned Slater-Koster tight-binding model) to a range of crystalline, molecular, defect, surface, and interface datasets, together with ALIGNN cross-checks on the same structures.

Each sub-project is a self-contained batch-inference pipeline: one script loads a dataset, runs SlakoNet (or ALIGNN) on every valid structure, and writes per-structure JSON outputs plus plots, summary CSVs, and a written `analysis.md` describing what the numbers say.

## Repository layout

```
slakonet/                     SlakoNet inference per dataset
  slako_v03_alex/             Alexandria PBE 3D crystals  (N = 31,211 paired)
  slako_v04_cccbdb/           NIST CCCBDB molecules       (N = 1,318)
  slako_v05_interface/        JARVIS interface_db slabs   (N = 433)
  slako_v06_surface/          JARVIS surface_db slabs     (N = 466)
  slako_v07_vacancy/          JARVIS vacancy_db defects   (N = 444)
  slako_v08_supercon/         Alexandria supercon set     (N = 4,827)
  slako_v09_1d/               Alexandria PBE 1D           (N = 8,636)
  slako_v10_2d/               Alexandria PBE 2D           (N = 79,903)
  slakonet_comprehensive_analysis/  Cross-dataset aggregation + unified plots

alignn/                       ALIGNN runs grouped by source dataset
  alignn_v03_alex/            Alexandria PBE 3D hull (paired with slakonet/slako_v03_alex)
    alignn_v1_pbe/            ALIGNN  mp_gappbe_alignn       (label-matched)
    alignn_v2_mbj/            ALIGNN  jv_mbj_bandgap_alignn  (TB-mBJ)
    alignn_v3_opt/            ALIGNN  jv_optb88vdw_bandgap_alignn
    pbe_mbj_opt_analysis/     SlakoNet vs three ALIGNN variants, side-by-side
  alignn_v04_cccbdb/          CCCBDB molecules           jalignn4.py
  alignn_v05_interface/       JARVIS interface_db        jalignn5.py
  alignn_v06_surface/         JARVIS surface_db          jalignn6.py
  alignn_v07_vacancy/         JARVIS vacancy_db          jalignn7.py
  alignn_v08_supercon/        Alexandria supercon set    jalignn8.py
  alignn_v09_1d/              Alexandria PBE 1D          jalignn9.py
  alignn_v10_2d/              Alexandria PBE 2D          jalignn10.py
  alignn_v11_alexwz/          Alexandria PBE 3D, no Z<=65 filter (jalignn11.py)
  alignn_v12_all/             Alexandria PBE 3D, full set (jalignn12.py; 99/100 shards complete)
  alignn_comprehensive_analysis/  Cross-dataset ALIGNN roll-up (analysis.md + csv + plots)
```

Every sub-project has a top-level inference script (`jslako_v*.py` for SlakoNet, `jalignn{N}.py` for the v04..v12 ALIGNN runs, `predict_alignn.py` for the older `alignn_v03_alex/alignn_v*` sub-runs), a `results/` directory of per-structure JSONs, and an `analysis/` directory of plots, metrics, and a written `summary.md` (or `analysis.md` in v03_alex).

## What this study finds

Two ML methods (SlakoNet tight-binding and pretrained ALIGNN graph network) are run on 9 datasets covering molecules, surfaces, interfaces, defects, superconductor candidates, and 1D/2D/3D Alexandria crystals (~4.5M structures total). Three core findings:

1. **ALIGNN wins by ~5x on the matched bulk-crystal set.** On the paired N = 31,211 Alexandria 3D hull subset where both methods produced output, ALIGNN reaches MAE 0.193 eV vs PBE; SlakoNet sits at 0.930 eV. Same structures, same reference.

2. **The two methods have differently shaped error distributions.** SlakoNet's residuals are **bimodal**: median error is tiny (0.027 eV, most predictions are dead-on) but ~10% of structures collapse to predicted gap = 0 when DFT says 5+ eV. The largest residuals concentrate on transition-metal compounds and ionic fluorides. ALIGNN's residuals are **unimodal and gentler**: spread Gaussian-ish around zero with no catastrophic collapse, but accuracy on non-metals is 4-5x worse than on metals. ALIGNN's largest residuals are also fluorides, but in the over-predict direction (off-hull fluoroborates where Alexandria PBE gives 0; ALIGNN may be more physically correct than the label here).

   A consistent cross-method pattern: **SK predicts more metals than ALIGNN on every dataset except molecules.** The metallicity gap is biggest on v10 2D (SK 59% metal, ALIGNN 28%) and v07 vacancy (SK 91% / ALIGNN 54%); similar but smaller on v05 / v06 / v09. SK's "extra metals" are silent-dropout cases on chemistries the model can't handle, showing up across all geometries.

3. **Geometry matters more than model architecture.** ALIGNN's MAE rises cleanly as inputs leave its 3D-bulk training distribution: 3D bulk crystals 0.17-0.19 eV, slabs/interfaces/1D/2D ~0.5 eV (~3x worse), isolated molecules 3.4 eV. The same model degrades by 3x just from removing the periodic-bulk assumption.

The two methods are complementary, not redundant: SlakoNet offers physical interpretability and DOS access plus a known set of pathological compositions; ALIGNN offers uniformly decent accuracy within its training distribution but no transferability guarantee outside it. The v07 vacancy result (SK 91% metal vs ALIGNN 54% metal on the same 444 transition-metal-defect cells) is the cleanest direct illustration of SK's documented error mode in a held-out cross-method test. The v12 fluoroborate worst-prediction pattern (DFT=0, ALIGNN=8 eV on Li/Sr/Ba/Ca-fluoroborates) is the cleanest illustration that "model error" and "label error" are not the same thing.

Full deep-analysis writeups: `slakonet/slako_v03_alex/analysis/analysis.md` (SlakoNet error modes), `alignn/alignn_comprehensive_analysis/analysis.md` (ALIGNN cross-dataset narrative).

## Headline SlakoNet results

Band gap, all values in eV. MAE / RMSE / Pearson *r* are against the dataset's DFT reference (PBE for Alexandria and surface_db, OptB88vdW for interface_db, HOMO–LUMO for CCCBDB). Vacancy and supercon sets have no DFT gap reference available.

| Dataset        |     N  | SK mean | SK median | Frac metallic | Ref mean | MAE   | RMSE  |  r    |
|----------------|-------:|--------:|----------:|--------------:|---------:|------:|------:|------:|
| Alexandria 3D  | 31,211 | 1.54    | 0.01      | 0.63          | 1.22     | 0.93  | 1.65  | 0.81  |
| Alexandria 2D  | 79,903 | 1.16    | 0.02      | 0.62          | 0.67     | 0.62  | 1.33  | 0.89  |
| Alexandria 1D  |  8,636 | 1.87    | 0.31      | 0.40          | 1.09     | 0.99  | 1.70  | 0.88  |
| CCCBDB mols.   |  1,318 | 7.45    | 6.31      | 0.00          | 6.74     | 2.52  | 3.52  | 0.65  |
| interface_db   |    433 | 1.43    | 1.41      | 0.17          | 0.43     | 1.01  | 1.26  | 0.73  |
| surface_db     |    466 | 1.67    | 1.18      | 0.35          | 0.77     | 0.97  | 1.59  | 0.75  |
| vacancy_db     |    444 | 0.16    | 0.00      | 0.92          | n/a      | n/a   | n/a   | n/a   |
| alex_supercon  |  4,827 | 0.02    | 0.00      | 0.98          | n/a      | n/a   | n/a   | n/a   |

## Headline ALIGNN results

ALIGNN `mp_gappbe_alignn` (PBE-trained) was run on every dataset (PBE-only by design). Conventions match the SlakoNet table above. The Alexandria 3D row uses the paired N = 31,211 subset where both methods succeeded (basis for the head-to-head section below). The vacancy_db and alex_supercon rows have no DFT gap reference; their MAE / RMSE / r columns are blank and the cross-method comparison appears below.

| Dataset        |     N  | ALIGNN mean | ALIGNN median | Frac metallic | Ref mean | MAE   | RMSE  |  r    |
|----------------|-------:|------------:|--------------:|--------------:|---------:|------:|------:|------:|
| Alexandria 3D  | 31,211 | 1.23        | 0.34          | 0.42          | 1.22     | 0.19  | 0.46  | 0.96  |
| Alexandria 2D  | 87,903 | 0.86        | 0.28          | 0.28          | 0.67     | 0.47  | 0.84  | 0.79  |
| Alexandria 1D  |  9,540 | 0.93        | 0.25          | 0.30          | 1.07     | 0.48  | 0.76  | 0.87  |
| CCCBDB mols.   |  1,330 | 3.83        | 3.85          | 0.00          | 7.02     | 3.36  | 7.93  | 0.28  |
| interface_db   |    587 | 0.95        | 0.88          | 0.03          | 0.47     | 0.53  | 0.72  | 0.56  |
| surface_db     |    487 | 0.97        | 0.59          | 0.22          | 0.78     | 0.51  | 0.89  | 0.69  |
| vacancy_db     |    444 | 0.73        | 0.02          | 0.54          | n/a      | n/a   | n/a   | n/a   |
| alex_supercon  |  4,827 | 0.03        | 0.00          | 0.94          | n/a      | n/a   | n/a   | n/a   |

**Three regimes.** Alexandria 3D is in-distribution: PBE-trained ALIGNN reaches MAE 0.19 eV with r = 0.96. Alexandria 2D, 1D, surface_db, and interface_db cluster around MAE 0.47 to 0.53 eV (~3x worse) because their geometries (1D, 2D, slab + vacuum, layered interface) sit outside the 3D-bulk training distribution. CCCBDB molecules are the third, far-OOD regime at MAE 3.36 eV with r = 0.28: ALIGNN is trained on crystals (graph neighborhood mismatch for isolated molecules) and the reference is molecular DFT (Gaussian basis, different functional), not solid-state PBE.

**Extended Alexandria 3D coverage (ALIGNN only).** Beyond the paired 31,211, ALIGNN was also run on `v11_alexwz` (115,535 hull-stable structures, no Z ≤ 65 element filter) and `v12_all` (4,444,402 structures, full hull + off-hull). On v11, ALIGNN reaches MAE 0.168 / RMSE 0.476 / metal-vs-gap accuracy 89.1% (its in-distribution best), with median |err| of 0.015 eV. On v12, MAE rises modestly to 0.185 / RMSE 0.551 / accuracy 78.1%; the on-hull subset of v12 (114k entries) reproduces v11 exactly. Hull-bin breakdown: on-hull MAE 0.168 / 89.1%; near-hull (0 to 0.1 eV/atom) 0.186 / 85.4%; off-hull (0.1 to 0.5) 0.205 / 79.8%; far off-hull (>0.5) 0.154 / 66.0%. Bias grows monotonically from on-hull (+0.024) to off-hull (+0.178). Shard 9 of 100 is missing (~45k entries, easy to fill in via resubmit).

**vacancy_db and alex_supercon (cross-method comparison).** Both lack a DFT bandgap reference. On **vacancy_db** (N = 444 paired), SK predicts 91.2% metallic, ALIGNN 54.3%, with metal/gap agreement of 61.3% and SK-vs-ALIGNN MAE of 0.634 eV. The parity plot in `alignn/alignn_v07_vacancy/analysis/plots/sk_vs_alignn.png` shows a vertical pile-up at SK gap = 0 against ALIGNN gaps spanning 0 to 6 eV, the visual signature of SK's silent dropout on open-shell transition metals (full diagnosis in `slakonet/slako_v03_alex/analysis/analysis.md`). On **alex_supercon** (N = 4,827) both methods predict the candidates are metallic (97.3% SK / 93.5% ALIGNN), with 92.4% agreement and SK-vs-ALIGNN MAE of 0.04 eV; the high-Tc subset (Tc > 5 K, N = 704) is 95.3% predicted metallic by ALIGNN, passing the sanity check.

**Functional-shift caveat for interface_db.** The reference is `optb88vdw_bandgap` (OptB88vdW), not PBE. ALIGNN predicts PBE. OptB88vdW typically gives slightly larger gaps than PBE for non-metals, so part of the +0.49 eV ALIGNN-over-DFT bias is the functional shift, not pure model error. 107 of 587 entries had negative OptB88vdW gaps (the documented interface SCF artifact) and were clipped to 0 before parity.
## Headline ALIGNN vs SlakoNet (Alexandria PBE 3D, paired N = 31,211)

From `alignn/alignn_v03_alex/pbe_mbj_opt_analysis/`. Reference is Alexandria PBE indirect gap.

| Model                               | MAE   | RMSE  |   R²    | Non-metal MAE |
|-------------------------------------|------:|------:|--------:|--------------:|
| SlakoNet (DFTB)                     | 0.930 | 1.649 | −0.008  | 1.781         |
| ALIGNN `mp_gappbe_alignn` (PBE)     | **0.193** | **0.463** | **+0.920** | **0.274**     |
| ALIGNN `jv_mbj_bandgap_alignn`      | 0.752 | 1.461 | +0.208  | 1.236         |
| ALIGNN `jv_optb88vdw_bandgap_alignn`| 0.354 | 0.746 | +0.794  | 0.602         |

**What this says.** On the accuracy-matched ALIGNN checkpoint, non-metal MAE is ~0.27 eV: the accuracy ceiling for these structures. SlakoNet reaches 1.78 eV on the same subset, dominated by two error modes (open-shell transition-metal compounds and ionic fluorides predicted as metals). On metals alone SlakoNet is actually the most accurate model (MAE 0.024 eV), because its default behaviour is to return ≈0. See `alignn/alignn_v03_alex/pbe_mbj_opt_analysis/analysis.md` for the full breakdown, including the functional-shift calibration between PBE / TB-mBJ / OptB88vdW.

## Reproducing a run

Inference is designed for a SLURM cluster with GPUs. SLURM job scripts (`job.sh`) are not tracked in this repo; write your own based on your cluster's partition, walltime, and GPU/CPU layout. The general flow for any `slako_v*`:

```bash
# On the cluster, with the slakonet conda env active
conda activate slakonet
python jslako_v<N>.py          # auto-detects multi-GPU / single-GPU / multi-CPU
```

The inference environment needs `torch`, the full `slakonet` package (`pip install` from an anonymized cluster/slakonet), `jarvis-tools`, and `tqdm`. ALIGNN runs additionally need the `alignn` package and a pretrained model that `alignn.pretrained.get_figshare_model` can fetch on first use.

Each run filters to elements with Z ≤ 65, checkpoints per-structure into `results/<id>.json`, and times out any single structure that exceeds 180 s. Re-running skips structures whose result file already exists.

### Data

No dataset zips ship with the repo. Download each from the an anonymized cluster JARVIS databases page and drop it into the matching sub-project working directory before running. The inference scripts look zips up by filename.

| Sub-project                                                                 | Expected zip                                                 |
|-----------------------------------------------------------------------------|--------------------------------------------------------------|
| `slako_v03_alex`, `alignn_v03_alex/alignn_v{1,2,3}_*`, `alignn_v11_alexwz`, `alignn_v12_all`   | `alexandria_pbe_3d_2024.10.1_jarvis_tools.json.zip` (1.1 GB) |
| `slako_v04_cccbdb`                                                          | `cccbdb.json.zip`                                            |
| `slako_v05_interface`                                                       | `interface_db_dd.json.zip`                                   |
| `slako_v06_surface`                                                         | `surface_db_dd.json.zip`                                     |
| `slako_v07_vacancy`                                                         | `vacancydb.json.zip`                                         |
| `slako_v08_supercon`                                                        | `alex_supercon.json.zip`                                     |
| `slako_v09_1d`                                                              | `alexandria_pbe_1d_2024.10.1_jarvis_tools.json.zip`          |
| `slako_v10_2d`                                                              | `alexandria_pbe_2d_2024.10.1_jarvis_tools.json.zip`          |

### Compute venue

- SlakoNet inference runs on a CPU cluster (`parallel` partition, 48-core nodes).
- ALIGNN inference runs on a CPU cluster (`main` partition, 256-core / 500 GB nodes); `dgl` must be installed explicitly from the dgl wheels index since it is not a transitive dependency of `pip install alignn`.

### Analysis

Each sub-project's `analysis/` directory ships with pre-built plots, a `summary.md` (or `analysis.md` in v03_alex), and a `metrics.csv` (or `summary.csv`) of the key scalars. Parity and residual plots in v04 to v11 ALIGNN dirs carry inset annotations of N, MAE, RMSE, and ME directly on the plot.

The cross-dataset layers are reader-only. Pre-built outputs live in:

- `slakonet/slakonet_comprehensive_analysis/` (cross-dataset SlakoNet roll-up)
- `alignn/alignn_comprehensive_analysis/` (cross-dataset ALIGNN roll-up, mirrors the SlakoNet layout)
- `alignn/alignn_v03_alex/pbe_mbj_opt_analysis/` (SlakoNet vs three ALIGNN variants on Alexandria 3D)

The scripts that produced them are kept local; analysis lives entirely in the artifacts on this side.

## Output schema

Each per-structure result JSON contains at least:

- `id` (or `jid` / `mat_id` depending on dataset): structure identifier
- `sk_bandgap`: SlakoNet band gap (eV)
- `dos_values`, `dos_energies`: DOS on a Fermi-aligned grid, `E − E_F ∈ [−10, 10]` eV, 5000 points, Gaussian broadening σ = 0.1 eV
- `atoms` (or `defective_atoms` for v07): the input geometry in JARVIS dict format
- Dataset-specific labels (PBE gap, formation energy, Tc, etc.)

ALIGNN predictions use the same per-structure layout with `alignn_bandgap` instead of `sk_bandgap`.

## License

MIT, see `LICENSE`.
