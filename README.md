# Prioritised non-target screening (pNTA) along the Rhine (RhineScan)

An end-to-end, notebook-based workflow that takes raw LC-HRMS data (positive mode) from eight Rhine sampling sites and four procedural blanks, removes laboratory background, and ranks the remaining chemical features by how interesting they are as early-warning candidates: do they appear suddenly, increase downstream, have MS2 data, or match known pollutants?

> **Status: method-development proof of concept.** Sites are ordered along the river (upstream to downstream), and that spatial gradient is used as a *proxy* for time. This is not a true temporal early-warning test. See [Limitations](#limitations).

---

## Data

| | |
|---|---|
| Source | Zenodo record [`17591915`](https://zenodo.org/records/17591915) *, file `P1_pos.zip` (about 1.8 GB) |
| Content | 12 `.mzXML` files, positive ESI: 8 river samples + 4 blanks (`MQ_1` to `MQ_4`) |
| Sites (upstream to downstream) | Weil, Karlsruhe, Worms, Koblenz, Bad Honnef, Duesseldorf, Duisburg, Bimmen |
| Integrity | The notebook downloads the file and verifies the Zenodo checksum before extracting |

Zenodo reference:  10.5281/zenodo.17591914.
---

## Pipeline

1. Download, checksum verification, sample labelling by river order
2. Data-driven noise threshold, pyOpenMS feature detection (m/z, RT, intensity), MS2 spectra retained
3. Cross-sample feature grouping, then re-quantification from raw data so that 0 means "not detected"
4. Blank filter (10x), intensity-drift normalisation on core features, isotope/adduct merging, solvent-front removal
5. Downstream trend (Spearman, with exact permutation p-values and Benjamini-Hochberg FDR) and jump size
6. Mass-error recalibration (leave-one-out error reported), suspect screening of 29 compounds by exact mass,
   MS2 fragment check
7. Priority score and tiers
8. Homologous-series detection (EO/PO units) with MS2 ladder test against a control; independence check of the
   Worms signals
9. Structure checks: RDKit formula validation, PubChem verification of SMILES (PubChemPy), in-silico fragmentation
   with a decoy test
10. Optional library matching (matchms + MassBank), exports (tables, MGF for MetFrag/matchms, run parameters)

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

## Identification confidence
 
Levels follow Schymanski et al. 2014 (Environ. Sci. Technol. 48, 2097-2098). Nothing here is level 1 or 2 unless the
optional MassBank step passes **and** the mirror plot is checked by a person. MS1 accurate-mass suspect matches are
level 5. Level 4 is not claimed because isotope-pattern/adduct evidence is not evaluated. Level 3 means curated
diagnostic fragments were found in measured MS2; it is a tentative candidate, not an identification.

### Structure-based checks (RDKit)
 
| Check | Result |
|---|---|
| Formula / [M+H]+ from SMILES vs suspect list | All 11 structures consistent (difference 0.000 mDa) |
| RT vs logP | Loose relationship; valsartan (+2.6 min) and clarithromycin (+2.0 min) elute later than predicted, most likely a limit of Crippen logP for ionisable acids and large macrolides (not tested further) |
| In-silico MS2, carbamazepine | 55% of MS2 intensity explained (median over 10 spectra); no decoy competes |
| In-silico MS2, clarithromycin | 40% explained (desosamine fragment m/z 158.118, +1.9 ppm); no decoy competes |
| In-silico MS2, tramadol / venlafaxine | 58% / 31% explained but **tied** with each other and with other dimethylamino/amine drugs (a shared low-mass amine fragment, likely C3H8N+ m/z 58.065, matches all) |
| In-silico MS2, metoprolol | Side-chain fragment (m/z 116.107) found, but only 2% of intensity explained: most of its signal needs rearrangements that single-bond cuts cannot produce |
| In-silico MS2, sartans, lamotrigine, citalopram, fexofenadine | < 10% explained (median); expected for ring-based fragmentation, which is not modelled; this is **not** evidence against these identifications |
| Benzotriazole | Not testable (only ring bonds) |
 
Summary: the structure-based test supports carbamazepine and clarithromycin, does not contradict any other assignment, and cannot discriminate structurally similar amines on low-mass fragments. A decoy score below about 0.05 is at the chance level. The evidence-matrix figure (`fig12`) collects all lines of evidence per suspect; its thresholds are heuristic and for communication only, not a statistical confidence score.
 
### Key figures
 
| | |
|---|---|
| Pipeline funnel | `docs/fig9_pipeline_funnel.png` |
| Suspect effect sizes (downstream vs upstream) | `docs/fig10_suspect_effect_sizes.png` |
| Structural similarity of the suspects | `docs/fig11_suspect_similarity.png` |
| Evidence matrix | `docs/fig12_evidence_matrix.png` |
 
![Evidence matrix](docs/fig12_evidence_matrix.png)

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
* **Identification confidence is low.** 8 of 11 suspects are level 5 (exact mass only) and 3 are level 3 (diagnostic fragments, taken from the literature and still to be verified in MassBank). No level 4 is claimed because isotope patterns were not tested against the formulas; level 2 needs a library match and level 1 a reference standard with retention time.
* **Sensitivity is modest.** About 900 to 1,500 features per sample at the chosen threshold (the notebook's own target was 2,000 to 8,000), and several common wastewater markers are not found.
* **Score design.** The dynamics term uses rank correlation, so a statistically monotonic but tiny increase (for example carbamazepine, about 0.2 log2 units) scores like a large one. Weights are expert-set, not fitted.
* **In-silico MS2 is a minimal test.** Only acyclic single bonds are cut, with no ring opening and no multi-step fragmentation, so ring-rich molecules score low and the result is a lower bound. Hydrogen shifts (-1 to +2 H) give each piece several chances to match, so the decoy test is needed to judge any score. SMILES were entered by hand (flat, no stereochemistry) and must be verified against PubChem.
* **MS2 mass accuracy.** After the MS1 mass-error correction, carbamazepine fragments sit at +2.6 to +4.3 ppm, so the MS1 model may not transfer to fragments; a 7 ppm window is used for MS2.
* **Hotspot explanations** (Worms, Mosel, Ruhr, and so on) are hypotheses, not findings.
* Report the sensitivity of the results to `NOISE_QUANTILE`, the blank choice and the score weights.

---

## Next steps
 
MetFrag + PubChemLite on `tier1_ms2_v2.mgf`; matchms against MassBank; paired negative-mode file; isotope-pattern
check (level 4); QC injections / true time series; ML-ready feature tables for downstream modelling.

---

## Licence

The source code in this repository is released under the
[MIT License](LICENSE).

The LC-HRMS dataset is not covered by this licence.
Please refer to the original [Zenodo record](https://zenodo.org/records/17591915)
for its data licensing and reuse conditions.



## Acknowledgements

Claude (Anthropic) was used as an AI coding assistant during the development of this project.
