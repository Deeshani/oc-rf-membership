# Random-Forest Open-Cluster Membership with Gaia DR3

Code and membership tables for:

> Mitra, D. & Bisht, D, "Random-Forest classification of open-cluster membership with
> Gaia DR3: multi-cluster validation and comparison with independent catalogues",
> submitted to MNRAS (2026).

The pipeline trains a supervised Random Forest per cluster on the membership labels of
Cantat-Gaudin et al. (2018, A&A 618, A93; "CG18") and validates the classifications
against held-out splits, spatially disjoint splits, and the independent Gaia DR3
catalogue of Hunt & Reffert (2023, A&A 673, A114; "HR23").

## Repository contents

```
NGC752_RF_Extended.ipynb      Full analysis notebook (Sections 0-9; all tables/figures)
membership_tables/            Per-cluster membership tables (CSV, see columns below)
outputs_extended/             All numerical results (CSV) and figures (PNG)
README.md                     This file
```

## Cluster sample

| Cluster  | Cone radius (deg) | Stars in cleaned sample | CG18 members (PMemb >= 0.5) |
|----------|------------------:|------------------------:|----------------------------:|
| NGC 752  | 2.91 | 194,494   | 245  |
| NGC 2632 | 3.00 | 113,841   | 706  |
| NGC 2682 | 1.00 | 14,791    | 754  |
| NGC 6633 | 2.16 | 1,525,454 | 209  |
| NGC 2516 | 2.98 | 598,093   | 913  |
| NGC 6791 | 1.00 | 158,471   | 1,623|

Cone centres, r50, and distances are taken from the CG18 cluster table
(VizieR J/A+A/618/A93, table 1). Gaia DR3 sources with parallax > 0 were retrieved
per cone; CG18 (DR2-based) entries were matched to DR3 by sky position within 1 arcsec.
Stars with any missing feature are removed before classification.

## Membership table columns (`membership_tables/<CLUSTER>_membership_table.csv`)

Each table contains the union of CG18 catalogue members and stars classified as members
by the RF (out-of-fold probability >= 0.5), sorted by `rf_prob_oof` descending.

| Column            | Description |
|-------------------|-------------|
| `source_id`       | Gaia DR3 source identifier |
| `ra`, `dec`       | Gaia DR3 position (deg, ICRS, epoch 2016.0) |
| `parallax`        | Gaia DR3 parallax (mas), **uncorrected** for the Lindegren et al. (2021) zero point; apply the correction before converting to distance |
| `pmra`, `pmdec`   | Gaia DR3 proper motions (mas/yr); `pmra` includes cos(dec) |
| `phot_g_mean_mag` | Gaia DR3 G-band mean magnitude |
| `bp_rp`           | G_BP - G_RP colour (mag) |
| `proba_cg`        | CG18 membership probability (0 if not in CG18) |
| `member`          | Training label: 1 if `proba_cg` >= 0.5, else 0 |
| `rf_prob_oof`     | Out-of-fold RF membership probability (see below) |
| `rf_member`       | 1 if `rf_prob_oof` >= 0.5, else 0 |

Interpretation notes:
- `rf_prob_oof` is produced by 5-fold stratified cross-validation, so **no star is
  scored by a model that was trained on it**.
- The probabilities are *approximately calibrated, catalogue-relative* membership
  probabilities (calibrated against the CG18/HR23 membership definitions), reliable
  near 0 and 1 and indicative in the intermediate range; see the paper, Section on
  probability reliability.
- Stars with `rf_member = 1` and `member = 0` are candidate members beyond the CG18
  catalogue; their HR23 status is discussed in the paper.

## Exact classifier configuration

```python
sklearn.ensemble.RandomForestClassifier(
    n_estimators = 200,
    max_depth = 10,
    max_features = 'sqrt',
    class_weight = 'balanced',
    random_state = 42,
    n_jobs = -1,
)
```

- Features (9): delta_ra, delta_dec (tangent-plane offsets, deg), parallax,
  pmra, pmdec, pm_mag = hypot(pmra, pmdec), phot_g_mean_mag, bp_rp, bp_g.
- Split: stratified 80/20 train/test, `random_state = 42` (seed stability over
  {42, 7, 123, 2026, 555} reported in the paper).
- Out-of-fold probabilities: `cross_val_predict` with
  `StratifiedKFold(n_splits=5, shuffle=True, random_state=42)`.
- Decision threshold: 0.5 (sensitivity over {0.3, 0.5, 0.7, 0.9} reported in the paper).
- Software: Python 3.10; scikit-learn 1.8.0; astropy; astroquery; numpy; pandas;
  matplotlib. Gaia data via the ESA Gaia TAP archive; catalogues via VizieR.

## Reproducing the analysis

1. Install dependencies: `pip install scikit-learn astropy astroquery numpy pandas
   matplotlib pyarrow gaiadr3_zeropoint`.
2. Run `NGC752_RF_Extended.ipynb` top to bottom. Gaia cone searches and supplementary
   column fetches are cached to parquet in `outputs_extended/` on first run
   (the NGC 6633 field is ~1.5M stars; first downloads take a while).
3. All tables (CSV) and figures (PNG) used in the paper are written to
   `outputs_extended/`.

## Data sources and credit

- Gaia DR3: Gaia Collaboration (2016, 2023). This work has made use of data from the
  ESA mission Gaia, processed by DPAC.
- CG18 membership labels: Cantat-Gaudin et al. (2018), VizieR J/A+A/618/A93.
- HR23 comparison catalogue: Hunt & Reffert (2023), VizieR J/A+A/673/A114.

## License and citation

Code: MIT license. If you use the membership tables or code, please cite the paper
above and the underlying catalogues (CG18, HR23) and Gaia.

[Zenodo DOI badge to be added on release]
