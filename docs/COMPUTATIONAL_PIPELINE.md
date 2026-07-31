# Computational Pipeline: CD109–CD26 Disruptor Peptide Optimization

*De novo* sequence generation and multi-stage in silico filtering to optimize a lead peptide disrupting the CD109–CD26 interaction at the CD109 MG4/bait-region interface.

---

## 1. System Definition

**Receptor**: CD109 (UniProt Q6YHK3); signal sequence (**residues 1–21**) excluded.

**Target residues (docking site)**:

| Region | Residues |
|---|---|
| Flexible loop, MG4 domain | E400, W402, S403, G404 |
| Bait region | Y650, D652, Y655 |

**Lead peptide (Peptide 1)**: `HIYTHMSHFIKQCFSLP` (17 aa)

---

## 2. Compute Environment

All generation, filtering, and structure-prediction steps (ProteinMPNN, biophysical triage, MHCnuggets immunogenicity screening, ColabFold) were run in **Google Colab Pro**, on a **high-RAM A100 GPU** runtime. MOE-based structure preparation, energy minimization, visual QC, and protein–protein docking were run locally/on-site (MOE is GUI-driven and not Colab-compatible).

---

## 3. Baseline Characterization

| Metric | Method | Value |
|---|---|---|
| Docking score (S<sub>avg</sub>) | MOE, average of top 10 poses | **−41.1158 kcal/mol** |
| GRAVY | ProtParam | **0.035** |
| Cleavage sites | PeptideCutter | **9** |
| Immunogenicity | Windowed MHC-I/II screen | **PASS** (0 strong binders) |

**Cleavage sites (9)**:

| Protease | Count | Positions |
|---|---|---|
| Chymotrypsin (high-sensitivity) | 3 | Y3, F9, F14 |
| Pepsin (pH > 2) | 5 | I2, H8, F9, F14, L16 |
| Trypsin | 1 | K11 |

---

## 4. Constrained Generation (ProteinMPNN)

| Constraint | Setting | Rationale |
|---|---|---|
| Frozen residues | I2, H5, H8, F9, Q12, S15, L16 | Anchors identified during MOE docking and pre-pipeline PyRosetta analysis |
| Sequences | 5,000 | Depth to survive filtering |
| Temperature | 0.5 | Native 0–1 "creativity" parameter; parity to survive filtering |
| Bias | Favor soluble, α-helix-driving residues; penalize acidic | Avoids excessive hydrophobic-but-stable analogs |
| Conditional mutations | Reduce bulky residues | Bulky side chains clash and shatter the backbone |
| Position 14 restriction | Ban trypsin/chymotrypsin targets | Initial runs showed repeated endopeptidase cleavage here |
| Net charge | +1 or +2 | Aqueous solubility; electrostatic anchoring |

**Yield**: 5,000 → **2,126**

---

## 5. Solubility Filter (GRAVY)

Candidates with GRAVY above the lead (0.035) were disqualified.

**Yield**: 2,126 → **180** (top 100 advanced to manual PeptideCutter passing)

---

## 6. Proteolytic Stability ([PeptideCutter](https://www.google.com/search?q=md+link+format&oq=md+link+format&gs_lcrp=EgZjaHJvbWUqDQgAEAAYkQIYgAQYigUyDQgAEAAYkQIYgAQYigUyBwgBEAAYgAQyCAgCEAAYFhgeMggIAxAAGBYYHjIICAQQABgWGB4yCAgFEAAYFhgeMggIBhAAGBYYHjIICAcQABgWGB4yCAgIEAAYFhgeMggICRAAGBYYHtIBCDY2OThqMGo3qAIAsAIA&sourceid=chrome&source=chrome.ob&ie=UTF-8))

Candidates with **≥7** cleavage sites removed (lead peptide: 9). Counts chymotrypsin high-sensitivity, pepsin (pH > 2), and trpsin cleavage sites.

**Yield**: 100 → **51**

---

## 7. Immunogenicity Screening

Sliding-window scan via **MHCnuggets** (local, IC50-based). Strong binder = IC50 < 50 nM.

| MHC class | Window length | Alleles |
|---|---|---|
| MHC-I | 9-mer | HLA-A\*02:01, HLA-A\*01:01, HLA-B\*07:02, HLA-B\*44:02 |
| MHC-II | 15-mer | HLA-DRB1\*01:01, \*03:01, \*04:01, \*07:01, \*11:01, \*15:01 |

Each full-length candidate is decomposed into all windows of the given length and scored against each allele in its class.

| Tier | Definition | Action |
|---|---|---|
| FAILED | Strong binder to ≥2 distinct alleles | Discard |
| WARNED | Strong binder to 1 distinct allele | Retain; ranking penalty |
| PASSED | No strong binders | Retain |

**Yield**: 51 → **48**

---

## 8. Structure Prediction and Ranking (ColabFold)

PDBs generated per complex based on a template MSA adapted from the lead peptide's alignment — swapping in each candidate's sequence while preserving the CD109 receptor context — then ranked on a weighted composite:

| Component | Scoring | Weight |
|---|---|---|
| Solubility (GRAVY) | Min–max 0–1; lowest best | 30% |
| Cleavage sites | Min–max 0–1; lowest best | 40% |
| Immunogenicity | Binary: 0 = PASS, 1 = WARN | 20% |
| ColabFold | pLDDT 25% + ipTM 75%; min–max 0–1; highest best | 10% |

ColabFold weighted low due to minimal parity between different peptide folds.

---

## 9. Structural QC (MOE)

Output `.pdb` files inspected visually. Many pocket-template folds produced backbone breaks; candidates unrepairable via MOE's native **Quick Prep** function were discarded.

**Yield**: 48 → **11**

---

## 10. Protein–Protein Docking (MOE)

Candidates >20% worse than the lead were removed.

```
Peptide → Compute → Prepare → Structure Preparation → Correct
Peptide → Compute → Energy Minimize (Amber10)
Compute → Protein-Protein Dock
```

| Parameter | Setting |
|---|---|
| Receptor / Ligand | CD109 / entire peptide |
| Site | E400, W402, S403, G404; Y650, D652, Y655 |
| Refinement | Rigid Body |
| Scoring | Hydrophobic Patch Potential enabled |
| Poses | 10,000 pre-placement → 1,000 placement → 100 refinement |

**Acceptance**: score = mean of top 10 poses. Poses rejected if `e_conf > −50 kcal/mol` or `rmsd_refine > 2.5 Å`; candidate discarded if <5 of the top 10 qualify.

**Yield**: 11 → **8**

**Removed at this stage**:

| Sequence | Reason |
|---|---|
| `LISAHSSHFKSQSRSLP` | N-terminus backbone break from structure preparation |
| `IIIAHHSHFHKQAKSLS` | Positive `e_conf` |
| `SIAIHHSHFKKQALSLP` | Docking broke backbone |

---

## 11. Final Candidate Set (n = 8)

Ordered by `Final_Rank_Score`.

### 11.1 Primary metrics

| Rank | ID | Sequence | MOE S | GRAVY | Cleavage | Immuno | Synthesis |
|---|---|---|---|---|---|---|---|
| — | Lead | `HIYTHMSHFIKQCFSLP` | −41.1158 | 0.035 | 9 | PASS | Benchmark |
| 1 | 01 | `SISSHSSHFSSQSKSLK` | **−49.4678** | −0.812 | 6 | PASS | Synthesized |
| 2 | 11 | `LIAAHHAHFHKQSKSLP` | −49.3361 | −0.412 | 5 | WARN | Synthesized |
| 3 | 16 | `AISAHSSHFSSQSKSLP` | −48.2781 | −0.371 | **4** | WARN | Synthesized |
| 4 | 14 | `LIGSHSHHFHSQSRSLL` | −48.6115 | −0.382 | 6 | PASS | Synthesized |
| 5 | 28 | `SISSHSSHFSKQALSLP` | −47.1009 | −0.253 | 6 | PASS | Synthesized |
| 6 | 25 | `IISSHHSHFKKQALSLP` | −46.9472 | −0.265 | 6 | WARN | Synthesized |
| 7 | 19 | `SIASHSAHFKHQAKSLM` | −46.5495 | −0.335 | 6 | PASS | Synthesized |
| 8 | 15 | `LIASHHSHFSKQAKSLS` | −41.7256 | −0.376 | 6 | WARN | **Excluded — synthesis failed (3×)** |

Ranking is composite-driven, so the MOE column is not monotonic. All natural analogs, like the lead, are **C-terminally amidated** (Rink Amide resin); this is omitted from the sequence strings above for readability.

- **Analog 01** — best docking score, **20.3%** over lead; most hydrophilic (GRAVY −0.812), no immunogenicity warning.
- **Analog 16** — lowest proteolytic liability (9 → **4** cleavage motifs), 17.4% docking improvement.
- **Analog 15** — cleared all computational filters but failed SPPS 3×. Retained for completeness of the in silico record; **not carried forward**. Effective synthesized set = **7**.

### 11.2 Scoring components

| ID | ipTM | pLDDT | score_gravy | score_cleavage | score_immuno | norm_ipTM | norm_pLDDT | score_cf | Composite | MOE scaled | Final |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 01 | 0.8694 | 70.434 | 1.0000 | 0.25 | 0.0 | 0.9110 | 0.1515 | 0.7211 | 47.21 | 100.00 | **86.80** |
| 11 | 0.7400 | 70.849 | 0.5277 | 0.50 | 0.0 | 0.4722 | 0.3775 | 0.4485 | 40.32 | 98.42 | **83.90** |
| 16 | 0.6358 | 70.297 | 0.4793 | 0.75 | 0.0 | 0.1191 | 0.0768 | 0.1085 | 45.47 | 85.76 | **75.68** |
| 14 | 0.6127 | 71.183 | 0.4923 | 0.25 | 0.0 | 0.0407 | 0.5591 | 0.1703 | 26.47 | 89.75 | **73.93** |
| 28 | 0.7330 | 71.588 | 0.3400 | 0.25 | 0.0 | 0.4484 | 0.7796 | 0.5312 | 25.51 | 71.66 | **60.12** |
| 25 | 0.8443 | 71.496 | 0.3542 | 0.25 | 0.0 | 0.8259 | 0.7296 | 0.8019 | 28.64 | 69.82 | **59.53** |
| 19 | 0.7653 | 71.355 | 0.4368 | 0.25 | 0.0 | 0.5581 | 0.6527 | 0.5818 | 28.92 | 65.06 | **56.02** |
| 15 | 0.6318 | 71.047 | 0.4852 | 0.25 | 0.0 | 0.1054 | 0.4852 | 0.2003 | 26.56 | 7.30 | **12.12** |

### 11.3 Data dictionary

| Column | Definition |
|---|---|
| `moe_s_score` | MOE docking score, mean of top 10 accepted poses (kcal/mol) |
| `gravy` | Grand average of hydropathy; lower = more hydrophilic |
| `cleavage_sites` | PeptideCutter total (chymotrypsin high-sens., pepsin pH > 2, trypsin) |
| `immunogenicity` | `0.0` = PASS, `1.0` = WARN (one allele, IC50 < 50 nM) |
| `iptm`, `plddt` | Raw ColabFold metrics |
| `score_*`, `norm_*` | Min–max normalized 0–1, higher = better |
| `score_cf` | ColabFold composite (ipTM 75% + pLDDT 25%) |
| `Composite_Score` | GRAVY 30% + cleavage 40% + immuno 20% + ColabFold 10% |
| `Score_MOE_Scaled` | Docking score min–max scaled across the surviving set, 0–100 |
| `Final_Rank_Score` | Composite combined with scaled docking performance |

---

## 12. Pipeline Yield Summary

| Stage | Method | In | Out |
|---|---|---|---|
| De novo generation | ProteinMPNN | 5,000 | 2,126 |
| Solubility filter | ProtParam (GRAVY) | 2,126 | 180 (top 100 advanced) |
| Proteolytic stability | PeptideCutter | 100 | 51 |
| Immunogenicity | Windowed MHC-I/II | 51 | 48 |
| Structural QC | MOE visual | 48 | 11 |
| Docking | MOE | 11 | **8** |
| Synthesis | Automated SPPS | 8 | **7** |

---

## 13. Rationally Designed Synthetic Analogs (n = 5)

Designed manually by substituting unnatural amino acids (uAAs) into synthesized analogs at positions carrying known in vivo liabilities.

> **No computational scores reported.** ProteinMPNN, ProtParam, PeptideCutter, and ColabFold are parameterized on the 20 canonical L-amino acids and reject uAA-containing sequences. These designs are justified mechanistically and require empirical characterization.

### 13.1 Sequences

| # | Sequence | Parent | Substitutions |
|---|---|---|---|
| S1 | `[Ac]-(D-Ser)-ISSHSSH-(F5-Phe)-S-(D-Lys)-QA-(tLeu)-SLP-[NH2]` | 28 — `SISSHSSHFSKQALSLP` | S1→D-Ser, F9→F5-Phe, K11→D-Lys, L14→tLeu |
| S2 | `[Ac]-AISAHSSH-(F5-Phe)-SSQS-(Orn)-SLP-[NH2]` | 16 — `AISAHSSHFSSQSKSLP` | F9→F5-Phe, K14→Orn |
| S3 | `[Ac]-IISSHHSH-(F5-Phe)-(D-Lys)-(Orn)-QALS-(tLeu)-P-[NH2]` | 25 — `IISSHHSHFKKQALSLP` | F9→F5-Phe, K10→D-Lys, K11→Orn, L16→tLeu |
| S4 | `[Ac]-LIGSHSHH-(F5-Phe)-HSQS-(α-Me-Arg)-S-(tLeu)-(tLeu)-[NH2]` | 14 — `LIGSHSHHFHSQSRSLL` | F9→F5-Phe, R14→α-Me-Arg, L16→tLeu, L17→tLeu |
| S5 | `[Ac]-SIASHSAH-(F5-Phe)-(Orn)-HQA-(D-Lys)-SL-(D-Nle)-[NH2]` | 19 — `SIASHSAHFKHQAKSLM` | F9→F5-Phe, K10→Orn, K14→D-Lys, M17→D-Nle |

All length-matched to their parents; frozen anchors (I2, H5, H8, F9, Q12, S15, L16) retained in identity or isosteric substitution.

The cationic positions use three different modifications (Orn, D-Lys, α-Me-Arg) deliberately: all improve proteolytic stability, but spreading minor side-chain/backbone variations tests which best preserves or improves binding affinity.

### 13.2 uAA rationale

| uAA | Replaces | Rationale |
|---|---|---|
| **Ornithine (Orn)** | Lys, Arg | Keeps the positive charge for electrostatic anchoring; shorter side chain is a poor trypsin substrate. |
| **D-Lysine (D-Lys)** | Lys | Retains lysine's charge and length exactly, but the D-center is not recognized by trypsin — removes the cleavage site with minimal steric change to the anchor. |
| **α-Methylarginine (α-Me-Arg)** | Arg | Preserves the guanidinium charge; α-carbon methylation adds a quaternary center that sterically hinders protease recognition and rigidifies toward α-helix. |
| **L-tert-Leucine (tLeu)** | Leu | tert-butyl sterically blocks chymotrypsin/pepsin access and rigidifies toward α-helix; isosteric enough to preserve the L16 anchor. |
| **Pentafluorophenylalanine (F5-Phe)** | Phe9 | Phe9 is both anchor and dual chymotrypsin/pepsin site, so cannot be deleted. Perfluorination keeps ring geometry and π-stacking; electron-poor ring resists cleavage, C–F bonds metabolically inert. |
| **D-Norleucine (D-Nle)** | Met17 | D-center blocks carboxypeptidase trimming; also removes the oxidation-prone Met thioether. |
| **D-Serine (D-Ser)** | Ser1 | Blocks N-terminal aminopeptidase attack, complementing acetylation. S1 only. |

### 13.3 Terminal Capping

All five are N-terminally acetylated and C-terminally amidated.

- **`[Ac]`** — neutralizes the free α-amino group, the primary aminopeptidase handle.
- **`[NH2]`** — blocks carboxypeptidase and removes a destabilizing negative charge at the helix C-terminus. Also matches the lead peptide, synthesized on Rink Amide AM resin.

Caps close both termini to exopeptidases; internal uAAs address endopeptidase sites. Neither is sufficient alone.

---

## 14. Synthesis & Workup

Identical protocol for all 12 peptides (7 natural, 5 synthetic).

### 14.1 Resin swelling

Rink Amide resin swelled in **10 mL total: 5 mL DCM + 5 mL DMF**. Swirled by hand, incorporated at room temperature — **no stir bar**.

### 14.2 Automated SPPS (CEM Liberty Blue)

Default instrument settings; Fmoc-protected amino acids in DMF.

| Parameter | Value |
|---|---|
| Resin | Rink Amide, **0.1283 g** |
| Resin loading | **0.78 mmol/g** (not preloaded) |
| Synthesis scale | **0.1 mmol** |
| C-terminus | Amide |
| Final deprotection | Enabled |

### 14.3 Cleavage and deprotection (CEM Razor)

Product dried, washed with **DCM** to remove residual DMF/DCM, then cleaved and deprotected **20 min at 42 °C**.

For synthetic analogs, perform N-terminal acetylation by adding 10 equivalents (1.0 mmol) of acetic anhydride (0.102 g, 0.094 mL) and DIPEA (0.129 g, 0.175 mL). Let gently stir in Razor cleavage tube for 30 minute. Repeat (2x total) to ensure reaction completion, as acetylated and non-acetylated variants of the same sequence are difficult to separate.

**Cocktail (10 mL total):**

| Component | Amount |
|---|---|
| TFA | 8.8 mL |
| TIPS | 0.2 mL |
| Water | 0.5 mL |
| Phenol | 540 mg |

**6–8 mL per peptide.** Final vacuum after deprotection directs product into diethyl ether.

### 14.4 Precipitation and isolation

1. Cool at **0 °C** in an ice bath to crash out peptide.
2. Reincorporate by vortex mixing.
3. Centrifuge in diethyl ether: **4 °C, 4,000–5,000 rcf, 5 min — 3 cycles.**
4. Between cycles, decant and replace with fresh **0 °C** diethyl ether, vortexing each round.
5. Air dry after the final spin.

### 14.5 Validation and storage

Crude product confirmed by **LC-MS** (target mass), stored in the dedicated peptide freezer, and transferred to the analytical team for purification.

---

## 15. Software and Tools

| Tool | Use |
|---|---|
| ProteinMPNN | Constrained *de novo* sequence generation |
| ProtParam | GRAVY / hydropathy |
| PeptideCutter | Protease cleavage site prediction |
| MHCnuggets | IC50-based immunogenicity screening |
| ColabFold | Complex structure prediction |
| MOE | Structure prep, energy minimization, visual QC, protein–protein docking |
| Amber10 | Force field for peptide energy minimization |
| Google Colab Pro (A100, high-RAM) | Compute platform for all generation/filtering/ColabFold steps |
