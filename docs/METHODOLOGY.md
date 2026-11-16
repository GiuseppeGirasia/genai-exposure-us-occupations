# Methodological documentation

This document records the methodological choices of the project. Every step is implemented — and can be re-run — in `main_analysis.ipynb`; section numbers below refer to the notebook. All counts are those produced by the current data files.

---

## 1. Sources and their verification (Notebook §2)

| Index | File | Original classification | Value used | Verification |
|---|---|---|---|---|
| Gmyrek et al. (2025), ILO WP 140 | `data/raw/ai_exposure/ilo/ILO_WP140.pdf` (Table A1) | ISCO-08, 4 digits (427 unit groups) | `Mean` = mean task-level GenAI exposure score (0–1) | Table A1 re-extracted from the PDF with `pypdf` and compared with the earlier spreadsheet `ILO_WP140_exposure_ISCO08.xlsx`: 427/427 groups, identical mean, SD and gradient. Extraction saved in `data/intermediate/ilo_table_a1_from_pdf.csv`. |
| Massenkoff and McCrory (2026) | `data/raw/ai_exposure/anthropic_observed/job_exposure.csv` | SOC 2018, 6 digits (756 codes) | `observed_exposure` (0–1) | All codes are valid SOC 2018 detailed codes; top values match Figure 3 of the paper (Computer Programmers 0.75; Customer Service Representatives 0.70; Data Entry Keyers 0.67). |
| Dingel and Neiman (2020) | `data/raw/telework/dingel_neiman/occupations_workathome.csv` | O\*NET-SOC 2010, 8 digits (968 codes) | `teleworkable` (0/1) | Codes unique, values in {0, 1}. |

ISCO-08 has 436 unit groups; the ILO does not score the three armed-forces groups and six residual groups (1439, 3139, 3435, 5249, 7319, 8189). No other index is used in the analysis.

## 2. Crosswalks and harmonisation to SOC 2018 (Notebook §3)

**Crosswalks (official BLS files in `data/raw/crosswalk/`).**
* SOC 2010 → SOC 2018 (900 links; also the reference list of the 867 SOC 2018 detailed occupations).
* ISCO-08 → SOC 2010 (1,125 links).
* O\*NET-SOC 2019 → NEM codes (used only to identify the 12 OEWS composite codes, §4).

**Link tables.** For each index a table of unique pairs *(original code, SOC 2018 code)* is built by chaining crosswalks; chains are *composed* into one direct table before any averaging. The link tables are saved in `data/intermediate/` (`links_dn_onet2010_soc2018.csv`, `links_ilo_isco08_soc2018.csv`).

* **Dingel–Neiman:** O\*NET-SOC 2010 code truncated to 6 digits (SOC 2010) → SOC 2010 → SOC 2018 crosswalk.
* **ILO:** ISCO-08 → SOC 2010 → SOC 2018 (composed; the "partial" flags of the ISCO crosswalk are ignored, consistent with the unweighted rule).
* **Massenkoff–McCrory:** identity.

**Correspondence problems found and solved.**
* O\*NET `19-1020.01` *Biologists* has a SOC 2010 *broad* prefix (`19-1020`) that is not in the crosswalk. It is mapped to `19-1029` (*Biological Scientists, All Other*), where O\*NET-SOC 2019 places Biologists (`19-1029.04`). Without the fix the occupation would be lost.
* SOC crosswalk titles carry footnote markers `(#)`, `(##)`, which are removed.
* ISCO codes are read as numbers by Excel (110): the leading zero is restored (`0110`).

## 3. Aggregation rule (Notebook §3.1, §5.1)

Whenever several original codes are linked to one SOC 2018 code, the harmonised value is the **simple, unweighted arithmetic mean over the distinct original codes**; when one original code is linked to several SOC 2018 codes, each receives its value. The same rule is applied to all three indices, and again when SOC 2018 codes are combined into the 12 OEWS composite codes (mean over the components that have a value). No employment or other weights are used in the construction of the indices.

**Harmonisation report (Table T01).**

| | ILO | M&M | D&N |
|---|---|---|---|
| Original codes | 427 | 756 | 968 |
| Original codes not matched (lost) | 0 | 0 | 0 |
| Original codes split over several SOC 2018 codes | 292 | 0 | 95 |
| SOC 2018 codes obtained | 840 | 756 | 800 |
| … from one original code | 651 | 756 | 667 |
| … averaging several original codes | 189 | 0 | 133 |
| … receiving the value of a split code | 749 | 0 | 76 |

**Coverage of the 867 SOC 2018 occupations (Tables T02–T04).** All three indices: 747 (86.2%); exactly two: 56; exactly one: 43; none: 21 (19 military). Missing: ILO 27, D&N 67 (mostly residual "All Other" categories, plus Data Scientists), M&M 111 (including all 28 components of the OEWS composite codes).

## 4. OEWS data and the 2019 occupational codes (Notebook §4)

**Variables.** National OEWS estimates for May 2019, 2022 and 2025; detailed occupations only. Employment = `TOT_EMP`; wage = mean annual wage `A_MEAN` (not top-coded, unlike the median). Values shown as `*` (annual wage not published for some performing-arts occupations) are left missing — no economic value is imputed.

**Outcomes.** Δ log employment and Δ log mean wage for 2019→2022 (placebo) and 2022→2025 (main). A change is computed only when both years are observed. Both periods are three years long, so the cumulative changes are directly comparable (dividing a coefficient by three gives a yearly rate).

**OEWS composite codes.** OEWS (2019, 2022, 2025) publishes 12 codes that combine 28 SOC 2018 detailed occupations (e.g. 13-1020 = 13-1021 + 13-1022 + 13-1023); their composition is read from the BLS NEM crosswalk and verified against the OEWS files.

**The 2019 problem.**
* *Codes affected:* 31 codes appear only in 2019 and 72 only in 2022. Thirty 2019 codes are "hybrid" codes created during the SOC 2010 → 2018 transition (the May 2019 estimates pool panels collected under both classifications); each is the union of 2–8 SOC 2018 detailed occupations (72 in total). The remaining 2019-only code is 29-1024 *Prosthodontists*, not published in 2022. (Between 2022 and 2025 the only differences are 11-1031 *Legislators*, missing in 2025, and 29-1024, missing in 2022.)
* *Verification:* the composition (BLS documentation, written in the notebook) is checked against the data: every 2019-only code except 29-1024 is a hybrid; every 2022-only code belongs to exactly one hybrid; all components exist in 2022 and 2025.
* *Procedure evaluated:* split each hybrid's 2019 employment with the components' 2022 shares (wages: hybrid 2019 mean wage × component's 2022 wage relative to the group's employment-weighted mean). Algebraically, each component then receives exactly the **observed group-level change**, Δlog E_c = Δlog E_h. The procedure is a correspondence between codes; it creates no occupation-specific information.
* *Assumptions:* (i) the 2019 hybrid and the 2022 group cover the same jobs; (ii) components grow at the same rate within a group (constant shares).
* *Diagnostics (Tables T05, T06):* shares are stable when observable (largest 2022–2025 change in a component share < 10 p.p. in every group, median 4); but some groups show reclassification artefacts (Data Scientists group ×5.3, Health Information Technologists group ×1.75, Project Management Specialists group ×1.5). Hybrid components grow by +10.6 log points on average in 2019–2022, against −4.8 for occupations unchanged since SOC 2010, and are more dispersed (SD 0.33 vs 0.20).
* *Decision:* the correspondence is implemented (72 occupations, 2019 total employment preserved), so that the main and placebo regressions use the same occupations. The reclassification noise affects only the placebo period and only the hybrid components (48 of the 743 occupations of the final sample, about 5% of its employment); it is listed among the limitations.

## 5. Missing observations and sample construction (Notebook §5, Tables T07–T08)

| Step | Occupations | Dropped |
|---|---|---|
| OEWS occupations (union 2019–2025) | 831 | – |
| Employment observed in all three years | 829 | 2 (Legislators, Prosthodontists) |
| All three indices available | 746 | 83 |
| Mean wage published in all three years | **743** | 3 (Actors, Dancers, Musicians and Singers) |

The analysis sample covers 133.9 million employees in May 2022 (90.5% of OEWS employment). The binding index is Massenkoff–McCrory (75 occupations, 13.3 million employees missing). A single common sample is used for all 12 regressions so that coefficients are comparable.

## 6. Standardisation (Notebook §6)

Each index is converted into a z-score, Z = (X − mean)/SD, with mean and SD (n − 1 denominator) computed on the **common analysis sample of 743 occupations**:

| Index | Mean | SD |
|---|---|---|
| ILO | 0.295 | 0.136 |
| M&M | 0.077 | 0.133 |
| D&N | 0.353 | 0.467 |

Standardisation makes coefficient *sizes* comparable but not the concepts. Table T11 translates 1 SD into original units and into a movement in percentile rank, with example occupations (for the almost binary D&N index the examples are the largest occupations at 0 and at 1): D&N is almost binary (1 SD ≈ half the jump from 0 to 1; the 0→1 jump = 2.14 SD), M&M has 54% zeros, ILO has only 121 distinct values.

## 7. Correlations (Notebook §7)

Pearson and Spearman correlations for the three pairs of indices, with the p-values of the test of zero correlation (`scipy.stats.pearsonr` and `scipy.stats.spearmanr`), on the common sample (N = 743 for every pair). Both are shown in one heatmap (Figure 2): Spearman below the diagonal, Pearson above. Spearman 0.55–0.64, Pearson 0.52–0.65. No bootstrap.

## 8. Econometric specification (Notebook §8)

**Main specification (12 regressions).** For each index j, outcome Y and period p:

Δ log Y_ip = α + β Z_ij + ε_ip

estimated by OLS, **one index per regression** (enforced by assertions), no controls, no weights, on the common sample. Standard errors are clustered by SOC minor group (first four characters of the code; for 2019 hybrid components, the minor group of the hybrid code), 93 clusters; inference with t(G − 1).

Results: Table T12 (full statistics), Table T13 (paper-style table), Figures 3–4 (binned scatter plots).

## 9. Comparison of the coefficients (Notebook §9)

The twelve coefficients are compared descriptively (Table T14, Figure 5): across indices for a given outcome and period, and between the placebo and the main period for a given index and outcome. Employment and wage coefficients are not compared with each other. No formal tests of equality between coefficients are run; this, together with weighted and controlled specifications, is listed among the limitations (Notebook §10).

## 10. Output map

| Output | Content |
|---|---|
| `data/intermediate/ilo_table_a1_from_pdf.csv` | ILO Table A1 extracted from the PDF |
| `data/intermediate/links_*.csv` | crosswalk link tables (original code → SOC 2018) |
| `data/intermediate/indices_soc2018.csv` | harmonised indices on the 867 SOC 2018 occupations |
| `data/intermediate/oews_panel_2019_2022_2025.csv` | OEWS panel with reconstructed 2019 codes, logs, changes, flags |
| `data/final/analysis_dataset.csv` | analysis sample (743 occupations), with the original and standardised indices |
| `output/tables/T01–T14*.csv` | all tables (CSV) |
| `output/figures/F01–F05*.png` | all figures (300 dpi) |
