# CD109–CD26 Disruptor Peptide Optimization

Computational optimization of a lead peptide that disrupts the **CD109–CD26** interaction, a checkpoint axis that restrains Th1/Th17 immunity and limits CAR T-cell antitumor activity. Starting from the lead peptide `HIYTHMSHFIKQCFSLP`, a multi-stage in silico pipeline reduced 5,000 *de novo* sequences to **8** viable analogs, of which **7** were synthesized; **5** additional analogs were then designed by hand using unnatural amino acids (uAAs).

**Biological context**: CD109 is an activation-induced checkpoint that engages the co-stimulator CD26 to restrain proinflammatory CD4 T cell responses. Transient disruption of this interaction enhances CAR T cell manufacturing and potency against neuroblastoma. This project optimizes the disruptor peptide for stability, solubility, and reduced immunogenicity.

Full method: **[`docs/COMPUTATIONAL_PIPELINE.md`](docs/COMPUTATIONAL_PIPELINE.md)**

---

## Pipeline at a glance

| Stage | Method | In | Out |
|---|---|---|---|
| *De novo* generation | ProteinMPNN | 5,000 | 2,126 |
| Solubility filter | ProtParam (GRAVY) | 2,126 | 180 (top 100 advanced) |
| Proteolytic stability | PeptideCutter | 100 | 51 |
| Immunogenicity | Windowed MHC-I/II | 51 | 48 |
| Structural QC | MOE visual | 48 | 11 |
| Docking | MOE | 11 | **8** |
| Synthesis | Automated SPPS | 8 | **7** |

Lead docking benchmark: **−41.1158 kcal/mol** (MOE, mean of top 10 poses). Top analog (`SISSHSSHFSSQSKSLK`) improves on this by **20.3%**; analog `AISAHSSHFSSQSKSLP` reduces the lead's 9 protease-liable motifs to **4**.

---

## Repository layout

```
├── chemdraw/
│   ├── lead_structure.cdxml            # Lead peptide structure
│   ├── natural_structures.cdxml        # 7 synthesized natural analogs
│   └── uAA_structures.cdxml            # 5 rationally designed uAA analogs
├── data/
│   ├── 01_post_solubility.fasta        # GRAVY-filtered candidates
│   ├── 02_post_peptidecutter.txt       # Post-PeptideCutter survivors
│   ├── 03_post_immunogenicity_*.csv    # MHCnuggets screen results (tabular)
│   ├── 03_post_immunogenicity.txt      # MHCnuggets screen results (raw log)
│   ├── 04_post_colabfold_ranked_*.csv  # ColabFold composite ranking
│   └── 05_post_moe_ranked.csv          # Final MOE-docked ranking (the 8)
├── docs/
│   └── COMPUTATIONAL_PIPELINE.md       # Full method, parameters, candidate tables
├── figures/                            # Poster figures — outstanding, not yet uploaded
├── notebooks/
│   ├── 01_Generation_BiologicalFunnel.ipynb   # ProteinMPNN → GRAVY → PeptideCutter → immunogenicity
│   └── 02_ManualFold.ipynb                    # ColabFold structure prediction + composite ranking
├── structures/
│   ├── background/
│   │   ├── cd109_cd26_dimer_complex.pdb    # AF2 prediction, CD26 dimer (no interaction)
│   │   └── cd109_cd26_monomer_complex.pdb  # AF2 prediction, CD26 monomer (binding-competent)
│   ├── colabfold/
│   │   ├── complexes/                  # Predicted complexes, 11 passing MOE QC
│   │   ├── peptide_solo/               # Solo peptide extracted from each complex
│   │   └── raw_outputs/                # Unfiltered ColabFold run outputs
│   ├── lead/
│   │   ├── lead_complex.a3m            # MSA / template source for all ColabFold docks
│   │   ├── lead_complex.pdb            # Lead CD109–peptide complex
│   │   ├── lead_dock.mdb               # MOE docking session (lead)
│   │   ├── lead_peptide.pdb            # Separated lead peptide
│   │   ├── lead_poses.pdb              # Lead docking poses
│   │   └── lead_receptor.pdb           # Separated CD109
│   ├── moe_docks/                      # Per-candidate MOE docking sessions (.mdb)
│   └── poses/                          # Per-candidate docking poses (.pdb)
└── README.md
```

---

## Compute environment

All ProteinMPNN generation, biophysical triage, MHCnuggets immunogenicity screening, and ColabFold structure prediction were run in **Google Colab Pro** on a **high-RAM A100 GPU** runtime (see `notebooks/`). MOE structure preparation, energy minimization, visual QC, and protein–protein docking (`.mdb` files under `structures/`) were run locally — MOE is a licensed, GUI-driven application and does not run in Colab.

---

## Where to find what

| You want… | Look in |
|---|---|
| The full method and every parameter | `docs/COMPUTATIONAL_PIPELINE.md` |
| The final 8 candidates + scoring matrix | `data/05_post_moe_ranked.csv` (mirrored in pipeline doc §11) |
| Per-stage attrition (auditable filter outputs) | `data/01_...` through `data/05_...` |
| Predicted complexes | `structures/colabfold/complexes/` |
| Docked poses | `structures/moe_docks/` (sessions), `structures/poses/` (poses) |
| How the docking templates were built | `structures/lead/` (`.a3m` + complex `.pdb`) |
| Why CD26 monomer (not dimer) was targeted | `structures/background/` |
| uAA / natural analog chemistry | `chemdraw/` |

---

## Data provenance and known gaps

This repo begins at the ProteinMPNN generation stage (`01_post_solubility.fasta`). Earlier intermediate pools (the full 5,000-sequence and ~2,126 charge-filtered sets) are not archived as standalone files but are **regenerable** from the ProteinMPNN configuration documented in the pipeline (frozen residues, temperature 0.5, solubility/charge bias, position-14 restriction).

**Structures are MOE and ColabFold outputs.** MOE sessions (`.mdb`) were generated interactively in a licensed GUI and cannot be re-run from a notebook; the pipeline documents every setting(site residues, refinement, pose counts, acceptance thresholds) so results are reconstructable even without re-running MOE itself.

### Outstanding

- **Figures not yet uploaded.** `figures/` is currently empty; poster figures 1 and 2 remain to
  be added.

---

## Tools

| Tool | Use |
|---|---|
| ProteinMPNN | Constrained de novo sequence generation |
| ProtParam | GRAVY / hydropathy |
| PeptideCutter | Protease cleavage site prediction |
| MHCnuggets | Local IC50-based immunogenicity screening |
| ColabFold (AF2-Multimer) | Complex structure prediction |
| MOE | Structure prep, energy minimization, visual QC, protein–protein docking |
| CEM LibertyBlue / Razor | Automated SPPS, cleavage/deprotection |

---

## Reference

Merle N.S. et al. *Complement-related CD109 averts autoimmunity but limits tumor control through
Th1 and Th17 restraint.*
