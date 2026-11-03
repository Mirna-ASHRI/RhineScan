# Prioritised non-target screening as an early-warning proxy (RhineScan)
 
Retrospective prioritised non-target analysis (pNTA) of LC-HRMS/MS data (positive ESI) from eight sites along the
Rhine (upstream to downstream: Weil, Karlsruhe, Worms, Koblenz, Bad Honnef, Duesseldorf, Duisburg, Bimmen) plus four
laboratory blanks. The downstream gradient is used as a stand-in for a time series, to develop and test the logic of
a unit that flags a sudden chemical release.
 
> **Status: method-development notebook.** Eight sites, one injection each, run order equals river order.
> This is a proxy for early-warning logic, not a validated early-warning test. See [Limitations](#limitations).
 
## Contents
 
| File | Purpose |
|---|---|
| `RhineScan-pNTA-env-chemoinformatics.ipynb` | The full pipeline with outputs (run top to bottom) |
| `RhineScan_one_page_summary.docx` | One-page project summary |
| `requirements.txt` | Python packages |
| `LICENSE` | Licence of the code |
 
## Data
 
Zenodo record `17591915`, file `P1_pos.zip` (about 1.8 GB, 12 mzXML files). The notebook downloads it and verifies
the Zenodo checksum. The data are not stored in this repository.
 
The optional MassBank step downloads a NIST-format `.msp` release from the MassBank-data GitHub page. Cite MassBank and
check its licence; the library is not redistributed here.
 
## Pipeline
 
1. Download, checksum verification, sample labelling by river order
2. Data-driven noise threshold, pyOpenMS feature detection (m/z, RT, intensity), MS2 spectra retained
3. Cross-sample feature grouping, then re-quantification from raw data so that 0 means "not detected"
4. Blank filter (10x), intensity-drift normalisation on core features, isotope/adduct merging, solvent-front removal
5. Downstream trend (Spearman, exact permutation p-values, Benjamini-Hochberg FDR) and jump size
6. Mass-error recalibration (leave-one-out error reported), suspect screening of 29 compounds by exact mass
7. Priority score and tiers
8. Homologous-series detection (EO/PO units) with an MS2 ladder test against a control; independence check of the
   Worms signals
9. Structure checks: RDKit formula validation, PubChem verification of SMILES (PubChemPy), in-silico fragmentation
   with a decoy test
10. MassBank spectral matching (matchms) with mirror plots; identification confidence per Schymanski et al. 2014
11. Small ML section: release detectors compared on simulated events injected into the measured background
    (group-aware cross-validation), then used to rank real features; top features clustered by profile shape.
    The simulation measures detector sensitivity only; it is not evidence of real events.
## Main results (single run; see the notebook for exact numbers)
 
- 2,028 re-quantified feature groups, 613 after blank filtering, 547 after de-duplication.
- 11 of 29 suspects found at 6-8 of 8 sites; all 11 SMILES match PubChem.
- Library matches: carbamazepine, clarithromycin and fexofenadine are level 2a (mirror plots checked); seven more are
  2a candidates pending mirror plots; tramadol is ambiguous with its isomer O-desmethylvenlafaxine (level 3).
- 21 of 547 features increase significantly downstream (BH q < 0.05); among suspects only tramadol and irbesartan pass.
- ML benchmark: modest extra sensitivity over the jump rule for 8-16-fold simulated events, no overall AUC gain
  (0.865 vs 0.861). An earlier simulator produced a spurious advantage and was corrected.
- The top 30 ML-ranked features fall into four profile clusters (19 share a step at Worms, 7 a late-eluting spike at
  Duisburg). Three Worms-cluster features have MassBank candidates (triphenylphosphine oxide, tetrabutylammonium,
  chloridazon); a common source is a hypothesis, not a finding.
## Identification confidence
 
Levels follow Schymanski et al. 2014 (Environ. Sci. Technol. 48, 2097-2098). MS1 accurate-mass suspect matches are
level 5. A library match with the same compound name (cosine >= 0.7, >= 4 peaks) makes a level-2a candidate; it becomes
level 2a only after a person has checked the mirror plot. Isomer ambiguity keeps a suspect at level 3. Level 4 is not
claimed (isotope-pattern/adduct evidence not evaluated) and level 1 is not possible (no reference standards).
 
## Reproducing
 
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter lab rhinescan.ipynb     # Restart kernel and run all cells; the first run downloads the data (~20 min)
```
 
Pin the pyOpenMS version printed in `run_params.json`; its API has changed between releases.
 
## Limitations
 
- Spatial gradient is not a time series; tributary and matrix effects cannot be separated from temporal dynamics.
- One injection per site and no QC injections: drift correction and real gradients are partly confounded.
- p-values from n = 8 ordered sites describe monotone trend only and ignore spatial autocorrelation.
- Mass calibration uses suspect hits, so the +/-3 ppm match window is optimistic; leave-one-out error is reported.
- Library spectra come from other instruments and collision energies; no retention-time check against standards.
- In-silico fragmentation (single acyclic bond cuts) cannot explain ring-opening fragmentation.
- The MS2 ethoxylate-ladder test discriminates weakly (35% of control spectra also show a ladder).
- Source hypotheses (e.g. a Worms or tributary input) are hypotheses, not findings.
- Thresholds (noise quantile, blank fold, score weights) are heuristics; sensitivity has not been reported.
  
## AI-assistance disclosure
 
Claude (Anthropic) was used as an AI coding assistant during the development of this project.
 
## Next steps
 
Check the remaining mirror plots; MetFrag + PubChemLite on `tier1_ms2_v2.mgf` for the Duisburg cluster; isotope-pattern
check (level 4); paired negative-mode file; QC injections / true time series; ML-ready feature tables for downstream
modelling.
