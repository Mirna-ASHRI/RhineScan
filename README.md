# RhineScan: prioritised non-target screening (pNTA) along the Rhine

An end-to-end, notebook-based workflow that takes raw LC-HRMS data (positive mode) from eight Rhine sampling sites and four procedural blanks, removes laboratory background, and ranks the remaining chemical features by how interesting they are as early-warning candidates: do they appear suddenly, increase downstream, have MS2 data, or match known pollutants?

> **Status: method-development proof of concept.** Sites are ordered along the river (upstream to downstream), and that spatial gradient is used as a *proxy* for time. This is not a true temporal early-warning test. See [Limitations](#limitations).

---

## Data

| | |
|---|---|
| Source | Zenodo record [`17591915`](https://zenodo.org/records/17591915), file `P1_pos.zip` (about 1.8 GB) |
| Content | 12 `.mzXML` files, positive ESI: 8 river samples + 4 blanks (`MQ_1` to `MQ_4`) |
| Sites (upstream to downstream) | Weil, Karlsruhe, Worms, Koblenz, Bad Honnef, Duesseldorf, Duisburg, Bimmen |
| Integrity | The notebook downloads the file and verifies the Zenodo checksum before extracting |

Please cite the Zenodo record and its authors when using this data: `<add citation and licence from the Zenodo page>`.

Note: the sample-name mapping in the notebook contains `karlsruhuhe`. This deliberately matches the spelling of the file name in the dataset. Do not "fix" it unless the upstream file name changes.

---

## Pipeline

| # | Step | What it does | Key parameters |
|---|---|---|---|
| 1 | Download and verify | Fetches the zip from Zenodo, checks the checksum, extracts | `RECORD_ID`, `FILENAME` |
| 2 | Noise tuning | Picks an MS1 intensity threshold from the intensity distribution of the first file | `NOISE_QUANTILE = 0.70` |
| 3 | Feature detection | pyOpenMS mass-trace detection, elution-peak detection and feature finding; MS2 spectra (top 60 peaks) are cached | `MASS_ERR_PPM = 5`, `min_trace_length = 4` |
| 4 | Cross-sample grouping | Intensity-seeded clustering of features across files (no RT alignment) | `PPM_TOL = 5`, `RT_TOL = 15 s` |
| 5 | Requantification (gap filling) | Re-measures every feature group in every raw file as an EIC, so "0" means "not detected" | LOD = 5 x noise, apex >= 3 x local median, >= 3 scans |
| 6 | Blank filter | Sample signal must exceed 10 x the maximum blank signal and the feature must be seen at >= 2 sites | `BLANK_FOLD = 10`, `MIN_SITES = 2` |
| 7 | Drift normalisation | Per-site log2 offset estimated from "core" features detected at all sites | 235 core features |
| 8 | Spatial dynamics | Spearman rank vs river position (`rho`) and a "jump" metric (site vs median of all upstream sites) | non-detects set to LOD |
| 9 | Deduplication | Merges 13C isotopes and NH4/Na/K adducts if profiles correlate; drops RT < 2.5 min (solvent front) | `min_corr = 0.7` |
| 10 | Mass-error model | Fits a linear ppm-vs-m/z error from anchor suspects and corrects all m/z values | residual MAD 0.59 ppm |
| 11 | Suspect screening | [M+H]+ match against 29 suspects within 3 ppm of the corrected mass (Schymanski level 5) | `SUSPECT_PPM = 3` |
| 12 | MS2 fragment check | Diagnostic fragment ions for carbamazepine, metoprolol and clarithromycin (tentative level 3) | 10 ppm, >= 5% base peak |
| 13 | Priority score | Weighted sum of intensity, detection frequency, MS2 presence, dynamics and suspect match | weights `W` in the notebook |
| 14 | Homologous series | Strict search for C2H4O / C3H6O repeat-unit series plus an MS2 ladder check against a control set | >= 4 consecutive units, RT step <= 20 s |
| 15 | Export | Priority table, suspect hits, MS2 fragment check, event-like lists, Tier-1 MGF, run parameters | |

The priority score only rewards signals that **increase downstream** or **appear suddenly**; features that decrease downstream score zero on the dynamics term.

---

## Headline results (from the current run)

Feature funnel: 9,053 feature groups, then 2,028 after requantification, 613 after the blank filter, 547 after deduplication and solvent-front removal, and 431 of those have at least one MS2 spectrum.

### Suspect hits (level 5, exact mass only, unless stated)

| Compound | m/z (corrected) | RT (min) | ppm | Sites | rho | Note |
|---|---|---|---|---|---|---|
| Carbamazepine | 237.1027 | 9.25 | 1.9 | 8 | 0.88 | MS2 fragments found (tentative level 3) |
| Benzotriazole | 120.0554 | 6.13 | -2.2 | 8 | 0.38 | Largest mass residual; low-m/z extrapolation |
| Venlafaxine | 278.2114 | 7.16 | -0.1 | 8 | 0.67 | |
| Tramadol | 264.1960 | 5.91 | 0.6 | 8 | 0.93 | A second candidate at 6.36 min exists (isomer or ambiguity) |
| Metoprolol | 268.1910 | 6.31 | 1.0 | 8 | 0.83 | MS2 fragments found (tentative level 3) |
| Irbesartan | 429.2388 | 9.57 | -2.3 | 8 | 0.93 | |
| Valsartan | 436.2345 | 11.60 | 0.5 | 8 | 0.88 | |
| Lamotrigine | 256.0155 | 6.18 | 1.4 | 8 | 0.79 | |
| Citalopram | 325.1712 | 8.15 | 0.5 | 8 | -0.17 | Flat, non-monotonic profile |
| Clarithromycin | 748.4837 | 9.23 | -0.7 | 8 | 0.81 | MS2 fragment found (tentative level 3) |
| Fexofenadine | 502.2950 | 9.00 | -0.4 | 6 | 0.23 | Not detected at Worms and Bimmen; needs EIC check |

Not found: diclofenac, caffeine, gabapentin, sulfamethoxazole (poor ESI+ response, elution in the solvent front, or below the detection limit). The sitagliptin "feature" (16 min) is a spurious exact-mass match and was removed by the blank/site filters.

### Event-like features and series

* 44 features jump at least 16x with persistence >= 80% and MS2; 25 remain after excluding jumps at the last site (Worms: 18, Duisburg: 7).
* 8 strict ethoxylate-type series (48 features). Their MS2 spectra show C2H4O ladders in 59 to 100% of spectra per series, against 34.6% in non-series control features.
* The 18 "Worms" features show Worms/downstream ratios from 2.3x to 10.7x. This could be real, but a single-sample effect cannot be excluded.

---

## How to reproduce

### On Kaggle (as developed)

1. Create a notebook with internet enabled and upload `rhine_pnta.ipynb`.
2. Run all cells top to bottom (`Save Version > Save & Run All` so outputs persist).
3. First run: about 20 min download, then a few minutes for extraction, requantification and plotting.

### Locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install pyopenms numpy pandas scipy scikit-learn matplotlib requests jupyter
jupyter lab rhine_pnta.ipynb
```

Outside Kaggle the notebook falls back to `./work/` for data and results. You need about 5 GB of free disk space (zip plus extracted mzXML).

> Pin your versions. The code was verified on **pyOpenMS 3.6**; the exact version is written to `run_params.json` on every run. A `requirements.txt` with pinned versions is recommended (see TODO).

### Caching behaviour (read this before changing parameters)

Per-sample features (`features/*.csv.gz`, `*_ms2.pkl`) and the requantified matrix (`requantified_matrix.csv.gz`) are cached and **not** keyed to parameters. If you change `NOISE_QUANTILE`, `MASS_ERR_PPM`, `EIC_PPM` or similar, delete `wastewater_watch_results/features/` and `requantified_matrix.csv.gz` first, otherwise stale results will be reused silently.

---

## Outputs (`wastewater_watch_results/`)

| File | Content |
|---|---|
| `priority_table.csv` | All retained features with scores, tiers, dynamics metrics, suspect label |
| `suspect_hits.csv` | Suspect matches with ppm error, RT, rho |
| `ms2_fragment_check.csv` | Diagnostic fragment counts and verdicts |
| `event_like_features*.csv` | Sudden-appearance candidates (plain and strict version) |
| `series_ms2_ladder_check.csv` | Homologous-series MS2 ladder test |
| `features_before_dedup.csv` | Features before adduct/isotope merging |
| `tier1_ms2.mgf` | MS2 spectra of the top 100 features, for MetFrag / matchms |
| `run_params.json` | All parameters and the pyOpenMS version used |
| `figures/` | Blank filter, normalisation drift, mass-error model, PCA, heatmap, suspect profiles, m/z-RT space, EICs |

---

## Quality control built into the workflow

* Checksum-verified download.
* Data-driven noise threshold instead of library defaults.
* Requantification from raw data, so zeros mean "not detected".
* Sensitivity check of the blank filter (all blanks: 613 features kept; MQ_3/MQ_4 only: 789).
* Mass-error model with residual reporting; tolerance tightened to 3 ppm only after correction.
* A control set (fixed random seed) for the homologous-series MS2 test, with false series removed by a strict criterion.

---

## Limitations

* **Spatial gradient is not a time series.** It supports method development, not a validated early-warning claim.
* **One injection per site, no replicates, no pooled QC injections.** Rankings have no uncertainty estimates, and run-to-run drift cannot be fully separated from real gradients when run order equals river order. The normalisation offsets (-0.32 at Weil to +0.20 at Duisburg) follow river order, so the normalisation may remove part of a real gradient.
* **Identification confidence is low.** Suspects are level 5 (exact mass) unless the MS2 fragment check applies (tentative level 3, fragments from the literature that still need to be verified in MassBank). Levels 2 and 1 require library spectra or reference standards.
* **Sensitivity is modest.** About 900 to 1,500 features per sample at the chosen threshold (the notebook's own target was 2,000 to 8,000), and several common wastewater markers are not found.
* **Score design.** The dynamics term uses rank correlation, so a statistically monotonic but tiny increase (for example carbamazepine, about 0.2 log2 units) scores like a large one. Weights are expert-set, not fitted.
* **Hotspot explanations** (Worms, Mosel, Ruhr, and so on) are hypotheses, not findings.
* Report the sensitivity of the results to `NOISE_QUANTILE`, the blank choice and the score weights.

---

## TODO / next steps

- [ ] Add `requirements.txt` / `environment.yml` with pinned versions
- [ ] Key caches to a hash of the parameters
- [ ] Add a minimum effect size (for example >= 2x downstream/upstream) to the dynamics term; add FDR-aware ranking
- [ ] Apply the mass-error correction to MS2 fragments and tighten the fragment tolerance from 10 ppm to about 5 ppm
- [ ] Check EICs for fexofenadine at Worms/Bimmen and for the 18 Worms features
- [ ] Test whether the Bimmen-only ethoxylate series could be carry-over (needs the injection order)
- [ ] Remove the "paste this over the cell" comments; merge the strict homologous-series cell into the main flow
- [ ] Replace global variables such as `a`, `b` (mass-error model) and `diffs` with explicit parameters
- [ ] MetFrag + PubChemLite on `tier1_ms2.mgf`; matchms against MassBank
- [ ] Process the paired negative-mode file (diclofenac, sulfamethoxazole, and so on)
- [ ] Split the notebook into a small package (`src/`) plus a config file; add a smoke test on one file

---

## Licence

`<choose a licence for the code>`. Data licence: see the Zenodo record.
