# RhineScan: prioritised non-target screening as an early-warning proxy

Retrospective prioritised non-target analysis (pNTA) of LC-HRMS/MS data (positive ESI) from eight sites along the
Rhine (upstream to downstream: Weil, Karlsruhe, Worms, Koblenz, Bad Honnef, Duesseldorf, Duisburg, Bimmen) plus four
laboratory blanks. The downstream gradient is used as a stand-in for a time series, to develop and test the logic of
a unit that flags a sudden chemical release.

> **Status: method-development notebook.** Eight sites, one injection each, run order equals river order.
> This is a proxy for early-warning logic, not a validated early-warning test. See [Limitations](#limitations).

## Data

Zenodo record `17591915`, file `P1_pos.zip` (about 1.8 GB, 12 mzXML files). The notebook downloads it and verifies
the Zenodo checksum. **Before publishing, check the licence and the preferred citation on the Zenodo record and add
them here.** The data are not stored in this repository.

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
11. Small ML section (`ml_cells.py`): release detectors compared on simulated events injected into the measured
    background (group-aware CV, noise stress test), then used to rank real features; ML-ready tables exported.
    The simulation measures detector sensitivity only; it is not evidence of real events.
    Result: modest extra sensitivity over the jump rule for 8-16-fold events and no overall AUC gain; the
    top-ranked features appear to form two source-like groups (Worms, Duisburg), which a clustering cell
    quantifies. An earlier simulator produced a spurious advantage and was corrected (see the notes).

## Identification confidence

Levels follow Schymanski et al. 2014 (Environ. Sci. Technol. 48, 2097-2098). Nothing here is level 1 or 2 unless the
optional MassBank step passes **and** the mirror plot is checked by a person. MS1 accurate-mass suspect matches are
level 5. Level 4 is not claimed because isotope-pattern/adduct evidence is not evaluated. Level 3 means curated
diagnostic fragments were found in measured MS2; it is a tentative candidate, not an identification.

## Limitations

- Spatial gradient is not a time series; tributary and matrix effects cannot be separated from temporal dynamics.
- One injection per site and no QC injections: drift correction and real gradients are partly confounded.
- p-values from n = 8 ordered sites describe monotone trend only and ignore spatial autocorrelation.
- Mass calibration uses suspect hits, so the +/-3 ppm match window is optimistic; leave-one-out error is reported.
- In-silico fragmentation (single acyclic bond cuts) cannot explain ring-opening fragmentation; use MetFrag/CFM-ID.
- MS2 ethoxylate-ladder test: 35% of control spectra also show a ladder, so it discriminates weakly.
- Source hypotheses (e.g. a Worms or tributary input) are hypotheses, not findings.
- Thresholds (noise quantile, blank fold, score weights) are heuristics; report sensitivity before claiming results.

## AI-assistance disclosure

Claude (Anthropic) was used as an AI coding assistant during the development of this project.

## Next steps

MetFrag + PubChemLite on `tier1_ms2_v2.mgf`; matchms against MassBank; paired negative-mode file; isotope-pattern
check (level 4); QC injections / true time series; ML-ready feature tables for downstream modelling.
