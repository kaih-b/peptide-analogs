# CD109–CD26 Disruptor Peptide Optimization

Computational optimization of a lead peptide that disrupts the **CD109–CD26** interaction, a
checkpoint axis that restrains Th1/Th17 immunity and limits CAR T cell antitumor activity. Starting
from the lead peptide `HIYTHMSHFIKQCFSLP`, a multi-stage in silico pipeline reduced 5,000 de novo
sequences to **8** viable analogs, of which **7** were synthesized; **5** additional analogs were
then designed by hand using unnatural amino acids (uAAs).

**Biological context:** CD109 is an activation-induced checkpoint that engages the co-stimulator
CD26 to restrain proinflammatory CD4 T cell responses. Transient disruption of this interaction
enhances CAR T cell manufacturing and potency against neuroblastoma (Merle et al.). This project
optimizes the disruptor peptide for stability, solubility, and reduced immunogenicity.

Full method: **[`docs/COMPUTATIONAL_PIPELINE.md`](docs/COMPUTATIONAL_PIPELINE.md)**

---

## Pipeline at a glance

| Stage | Method | In | Out |
|---|---|---|---|
| De novo generation | ProteinMPNN | 5,000 | 2,126 |
| Solubility filter | ProtParam (GRAVY) | 2,126 | 180 (top 100 advanced) |
| Proteolytic stability | PeptideCutter | 100 | 51 |
| Immunogenicity | Windowed MHC-I/II | 51 | 48 |
| Structural QC | MOE visual | 48 | 11 |
| Docking | MOE | 11 | **8** |
| Synthesis | Automated SPPS | 8 | **7** |

Lead docking benchmark: **−41.1158 kcal/mol** (MOE, mean of top 10 poses). Top analog
(`SISSHSSHFSSQSKSLK`) improves on this by **20.3%**; analog `AISAHSSHFSSQSKSLP` reduces the lead's
9 protease-liable motifs to **4**.

---

## Repository layout

```
├── docs/
│   └── COMPUTATIONAL_PIPELINE.md    # Full method, parameters, candidate tables
├── environment/
│   ├── requirements.txt             # Pinned tool/library versions
│   └── notebook_metadata/           # Recovered run metadata → exact outputs
├── notebooks/
│   ├── 01_generation_filtering.ipynb   # ProteinMPNN → GRAVY → PeptideCutter → immunogenicity
│   ├── 02_structure_ranking.ipynb      # ColabFold + composite scoring
│   └── 03_docking_analysis.ipynb       # MOE outputs, final ranking, figures
├── data/
│   ├── peptidecutter_batch_180.csv     # PeptideCutter batch
│   ├── premoe_metrics.csv              # Pre-MOE metrics
│   ├── premoe_ranked_candidates.csv    # Ranked pre-MOE candidates
│   ├── colabfold_final_candidates.csv  # Candidates entering ColabFold
│   └── final_clinical_candidates.csv   # Post-MOE rank, pre-synthesis (the 8)
├── structures/
│   ├── lead/                        # Lead complex + separated chains + template inputs
│   │   ├── lead_complex.pdb
│   │   ├── lead_complex.a3m          # Positional info — template source for all docks
│   │   ├── lead_peptide.pdb
│   │   ├── lead_receptor.pdb
│   │   └── lead_dock.moe
│   ├── colabfold_complexes/         # 11 complexes passing MOE QC (pre-backbone-break)
│   │   ├── complexes/                # 11 complex .pdb
│   │   └── peptides_solo/            # Solo peptide .pdb from each complex
│   ├── moe_docks/                   # 10 .moe docks (backbone intact after docking)
│   └── poses/                       # 8 pre-synthesis candidates, 3 poses each (.pdb)
├── design/
│   └── chemdraw/                    # Natural + uAA structures (ChemDraw)
└── figures/
    ├── figure1_baseline_complex.png
    └── figure2_analog_overlay.png
```

---

## Where to find what

| You want… | Look in |
|---|---|
| The full method and every parameter | `docs/COMPUTATIONAL_PIPELINE.md` |
| The final 8 candidates + scoring matrix | `data/final_clinical_candidates.csv` (mirrored in pipeline doc §10) |
| Per-stage attrition (auditable filter outputs) | `data/*.csv` |
| Predicted complexes | `structures/colabfold_complexes/` |
| Docked poses | `structures/moe_docks/`, `structures/poses/` |
| How the docking templates were built | `structures/lead/` (`.a3m` + complex `.pdb`) |
| uAA / natural analog chemistry | `design/chemdraw/` |

---

## Data provenance and known gaps

This repo begins at the PeptideCutter stage. Earlier pools are handled as follows:

- **5,000-sequence ProteinMPNN pool and the ~2,126 charge/GRAVY-filtered set are not archived.**
  They are **regenerable** from the ProteinMPNN configuration documented in pipeline §3 (frozen
  residues, temperature 0.5, solubility/charge bias, position-14 restriction), so they are
  reproducible rather than lost.
- **Structures are MOE and ColabFold outputs.** MOE docking (`.moe` files) was run interactively
  in a licensed GUI and cannot be re-executed from a notebook; the pipeline documents every
  setting (site residues, refinement, pose counts, acceptance thresholds) so results are
  reconstructable. Record the MOE version in `environment/requirements.txt`.
- **Notebook run metadata** in `environment/notebook_metadata/` can, if needed, reconstruct exact
  per-run outputs for the archived stages.

### Open items before release

- **Name the immunogenicity predictor + version** in `environment/requirements.txt` (§6 of the
  pipeline documents the alleles and IC50 < 50 nM threshold but not the tool).
- **Reconcile the 180 vs. "top 100 advanced" count.** `peptidecutter_batch_180.csv` suggests all
  180 GRAVY-survivors may have been cut, not just the top 100 — verify the CSV row count against
  the pipeline's "top 100 advanced" language and correct whichever is wrong.
- **α-Me-Arg labeling.** The uAA is α-carbon methylated; if sending to a vendor, relabel
  `N-Me-Arg` → `α-Me-Arg` to avoid ordering *N*-methylarginine.

---

## Tools

| Tool | Use |
|---|---|
| ProteinMPNN | Constrained de novo sequence generation |
| ProtParam | GRAVY / hydropathy |
| PeptideCutter | Protease cleavage site prediction |
| ColabFold (AF2-Multimer) | Complex structure prediction |
| MOE | Structure prep, energy minimization, visual QC, protein–protein docking |
| CEM LibertyBlue / Razor | Automated SPPS, cleavage/deprotection |

---

## Reference

Merle N.S. et al. *Complement-related CD109 averts autoimmunity but limits tumor control through
Th1 and Th17 restraint.*
