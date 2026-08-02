# LC480 .ixo QC Analyzer

A **browser-based, client-side quality-control analyzer** for Roche LightCycler 480 `.ixo` experiment files. Drag-and-drop your files, define sample roles and target-specific acceptance rules, and get instant QC verdicts with full traceability — no data ever leaves your computer.

**Live site:** [https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/)

---

## Table of contents

1. [What it does](#what-it-does)
2. [Quick start](#quick-start)
3. [File overview](#file-overview)
4. [Supported formats](#supported-formats)
5. [How the QC pipeline works](#how-the-qc-pipeline-works)
   - [Step 1 — Load runs](#step-1--load-runs)
   - [Step 2 — Assign sample roles](#step-2--assign-sample-roles)
   - [Step 3 — Set target-specific acceptance rules](#step-3--set-target-specific-acceptance-rules)
   - [Step 4 — Review results](#step-4--review-results)
6. [Sample-name matching explained](#sample-name-matching-explained)
7. [Target-name matching explained](#target-name-matching-explained)
8. [Efficiency calculation](#efficiency-calculation)
9. [Replicate agreement](#replicate-agreement)
10. [Evidence bundle](#evidence-bundle)
11. [Theoretical background](#theoretical-background)
12. [Deployment](#deployment)
13. [Browser compatibility](#browser-compatibility)
14. [License and disclaimer](#license-and-disclaimer)

---

## What it does

The analyzer reads Roche LightCycler 480 `.ixo` result files directly in the browser, decodes the raw fluorescence curves, extracts stored crossing points (Cq) and call codes, and evaluates them against user-configurable QC rules. It is designed for GMO screening laboratories that need:

- **Sample classification** by name (CN = Certified Negative, NTC = No-Template Control, etc.)
- **Target-specific acceptance rules** (e.g. "CN PORUMB must amplify HMG at Cq 20–27 and must stay clean for 35S and T-NOS")
- **Per-well efficiency estimation** using the window-of-linearity (WoL) method
- **Replicate agreement checking** with configurable maximum Cq spread
- **Evidence bundle export** with SHA-256 hashes, CSV tables, RDML, and R scripts

All computation happens in the browser. No server, no upload, no cloud dependency.

---

## Quick start

1. **Open the live site**  
   [https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/)

2. **Load your data**  
   Drag-and-drop `.ixo` files or a `.zip` archive onto the drop zone, or click to select files. ZIP entry CRC-32 is verified automatically.

3. **Configure your lab profile**  
   Fill in the profile name, analyst, reviewer, replicate scheme, and maximum replicate spread.

4. **Define sample classification rules**  
   Use the dropdown to choose **exactly**, **hyphen exactly**, **every target**, or **custom regex** for each sample-name pattern. These rules assign roles (Positive, Negative, NTC, Blank, etc.).

5. **Define target-specific QC acceptance rules**  
   For each sample role, specify which targets must amplify (with Cq min/max) and which must stay clean.

6. **Run the analysis**  
   Click **Analyze**. The tool reports PASS / FAIL / MISSING per well, with full evidence.

7. **Export**  
   Download the evidence bundle (CSV, RDML, R script, PNG graphs, manifest with SHA-256).

---

## File overview

| File | Description |
|------|-------------|
| `index.html` | The single-file web application (~95 KB). Contains the full UI, parser, QC engine, graph renderer, and export logic. |
| `ixo_qc_export.html` | Previous-generation LC480 viewer (~208 KB). Provides plate maps, melt curves, Levey-Jennings charts, and multi-tab inspection. Kept for reference. |
| `ixo_qc_documentation.md` | Technical documentation of the `.ixo` parsing pipeline and known bugs in the previous parser. |

---

## Supported formats

| Format | Support | Notes |
|--------|---------|-------|
| `.ixo` | ✅ Full | Roche LightCycler 480 experiment result files. XML-based with Base64-zlib-encoded binary fluorescence. |
| `.zip` | ✅ Full | Archives containing `.ixo` files. CRC-32 verified. RAR is **not** supported. |
| `.rdml` | ✅ Export only | Generated as output for cross-validation with LinRegPCR. |

---

## How the QC pipeline works

### Step 1 — Load runs

The analyzer parses each `.ixo` file in two stages:

1. **XML extraction** — Reads the `<objectstream>` root, traverses `<obj>`, `<prop>`, and `<list>` nodes to recover well-level results.
2. **Binary decoding** — The `AcquisitionStore` property holds raw fluorescence as a Base64 → zlib → inner XML → little-endian float32 pipeline.

From each file the analyzer extracts:
- Sample names and well positions
- Target/assay names
- Stored `CrossingPoint` (Cq) and `Call` codes  
  (`0 = Negative`, `1 = Uncertain`, `2 = Positive`, `3 = NotCalculated`)
- Quality flags (`CpUncertain`, `CpState`, `IsIncluded`, `WarnCodes`)
- Raw fluorescence curves for efficiency calculation

### Step 2 — Assign sample roles

Sample roles are assigned by the **first matching rule** in the sample-classification table. Each rule has:

| Column | Meaning |
|--------|---------|
| **Assigned sample role** | The role to assign (e.g. `Positive`, `Negative`, `NTC`, `Blank`, `Sample`) |
| **Sample-name pattern** | The sample name to match (e.g. `CN PORUMB`, `NTC`, `415b`) |
| **Match mode** | `exactly` / `hyphen exactly` / `every target` / `custom regex` |
| **Action** | (reserved) |

**Important:** Enter **sample/control names** here (e.g. `CN PORUMB`, `NTC`), not target names (e.g. `HMG`, `35S`).

### Step 3 — Set target-specific acceptance rules

After role assignment, each well is tested against the target-specific QC table:

| Column | Meaning |
|--------|---------|
| **Sample role** | Which role this rule applies to |
| **Target-name pattern** | Which target to match (e.g. `HMG`, `35S`, `LEC`, `T-NOS`) |
| **Match mode** | `exactly` / `hyphen exactly` / `every target` / `custom regex` |
| **Expected** | `Amplify` (Cq must fall in range) or `Clean` (must be Negative / NotCalculated) |
| **Cq min** | Lower bound (inclusive) for amplification |
| **Cq max** | Upper bound (inclusive) for amplification |
| **Required** | `yes` = rule must be satisfied; `no` = advisory only |
| **Evidence label** | Free-text note printed in the report |

**Important:** Enter **target names** here (e.g. `HMG`, `35S`), not sample names.

### Step 4 — Review results

The analyzer produces per-well and per-sample verdicts:

| Verdict | Meaning |
|---------|---------|
| **PASS** | All required rules satisfied, replicates agree |
| **FAIL** | A required rule is violated, or replicates disagree |
| **MISSING** | A required rule has no matching wells (evidence not found) |
| **WARNING** | Advisory rule triggered, or quality flags present |

---

## Sample-name matching explained

The **Sample-name pattern / regex** dropdown offers four modes:

| Mode | Behaviour | Example input | Matches |
|------|-----------|---------------|---------|
| `exactly` | Full-string match, case-insensitive | `CN PORUMB` | `CN PORUMB`, `cn porumb` |
| `hyphen exactly` | Matches the name with or without a hyphen | `T-NOS` | `T-NOS`, `TNOS` |
| `every target` | Wildcard `.*` — matches any sample name | *(none)* | All samples |
| `custom regex` | Full JavaScript regex, `^` = start, `$` = end | `^(CN PORUMB\|NTC)$` | `CN PORUMB` or `NTC` |

Rules are evaluated **top to bottom**; the first match wins. Unmatched samples keep the default role **Sample**.

---

## Target-name matching explained

The **Target-name pattern / regex** dropdown uses the same four modes, but matches against assay/target names stored inside the `.ixo`:

| Mode | Example input | Matches |
|------|---------------|---------|
| `exactly` | `HMG` | `HMG`, `hmg` |
| `hyphen exactly` | `T-NOS` | `T-NOS`, `TNOS` |
| `every target` | *(none)* | All targets (useful for NTC/blank rules) |
| `custom regex` | `^(HMG\|LEC)$` | `HMG` or `LEC` |

Regex tips (for `custom regex` mode):
- `^HMG$` — exactly HMG
- `^(HMG|LEC)$` — exactly HMG or LEC; `|` means OR
- `^(35S|T-?NOS)$` — 35S, TNOS or T-NOS; `?` makes the hyphen optional
- `^IPC$` — exactly IPC
- `.*` — every target

---

## Efficiency calculation

The browser computes a **single-curve efficiency** for every well the instrument called **Positive**, using the original thesis window-of-linearity (WoL) index:

1. Grid-search the fluorescence baseline `β` just below the curve minimum.
2. Select the **window of linearity** — contiguous cycles whose baseline-subtracted signal lies between **2 %** and **25 %** of the curve amplitude.
3. Fit `log10(Fc − β)` against cycle `c` by ordinary least squares.
4. Keep the baseline/window combination with the highest R² (reject if R² < 0.98).
5. Report **E = (10^slope − 1) × 100 %**.

A perfect per-cycle doubling gives **E = 100 %**. The analyzer gates results with:
- **R² ≥ 0.98**
- **30 % < E < 140 %**

> **Note:** This is a **per-reaction kinetic index**, not the MIQE-grade assay efficiency from a dilution series. It is reproducible enough to compare like-with-like, but not a substitute for a standard-curve validation. See the *Theoretical background* section below.

Two efficiency back-ends are selectable in the UI:

| Option | Description |
|--------|-------------|
| **Thesis WoL index** | The original Chapter XII.1 implementation. |
| **Exploratory sliwin-style proxy** | A browser approximation of the qpcR `sliwin` approach (not the full qpcR package). |

Exact LinRegPCR and qpcR remain the desktop references; the evidence bundle contains RDML input and an R script for local reproduction.

---

## Replicate agreement

Wells sharing the same **root sample name** and **target** are grouped as replicates.

| Condition | Check |
|-----------|-------|
| All replicates **Positive** | Stored Cq spread ≤ **Maximum replicate spread** (configured in the lab profile) |
| All replicates **Negative** | Automatically consistent |
| **Mixed calls** (Positive + Negative) | Reported as discordant-replicate failure |

The replicate scheme can be set to `2`, `3`, or other configurations.

---

## Evidence bundle

Every analysis run produces a cryptographically traceable evidence bundle containing:

- Source `.ixo` files
- Decoded XML/JSON representations
- SHA-256 checksums of all inputs
- PNG amplification and melt-curve graphs
- CSV well tables and QC summary
- RDML file for LinRegPCR cross-validation
- qpcR-ready R script
- Report HTML
- Manifest with cryptographic fingerprints

> The printed/saved PDF is signature-ready but is **not** itself a qualified electronic signature unless signed with an approved external system.

---

## Theoretical background

This section summarises the thesis-level theory behind every quantity the analyzer reports.

### 1. The `.ixo` format and decoding pipeline

A LightCycler 480 `.ixo` file is a single XML document rooted at `<objectstream>`. Binary data appear only inside leaf elements, always wrapped in text-safe encoding (Base64 → zlib → inner XML → float32). The same three primitives — `<obj>`, `<prop>`, `<list>` — recur at every level, so a single generic reader traverses the whole file.

### 2. Per-well results: crossing points and calls

The quantitative result for every well is stored as a `QuantSampleB` object. `CrossingPoint` (Cp) is the instrument's name for Cq; a stored value of `0.0` means "no crossing" and must be treated as non-amplification. Call codes: **0 = Negative**, **1 = Uncertain**, **2 = Positive**, **3 = NotCalculated**. Quality fields (`CpUncertain`, `CpState`, `IsIncluded`, `WarnCodes`) are propagated to the final verdict.

### 3. Amplification efficiency: window-of-linearity

In the exponential phase `Fc = F0(1 + E)^c`, so `log10(Fc − β)` is linear in cycle `c` with slope `log10(1 + E)`. The WoL estimator grid-searches `β`, selects the linear window (2 %–25 % of amplitude), fits by least squares, and reports `E = (10^slope − 1) × 100 %`.

### 4. Two quantities called "efficiency"

- **Assay efficiency** (dilution series): MIQE-grade. `E = 10^(−1/slope) − 1` from Cq vs. log(concentration). Perfect doubling: slope = −3.32, E = 100 %. ENGL window: slope ∈ [−3.6, −3.1] → E ∈ [89.6 %, 110.2 %].
- **Single-curve efficiency** (one reaction): `E` from the shape of one curve. Needs no dilution series, but is noisier and best read as a **relative kinetic index**.

### 5. Inhibition detection

PCR inhibitors (polysaccharides, polyphenols, humic acids, EDTA, ethanol) lower efficiency and delay Cq. The routine plate monitors this through:
1. **Endogenous reference target** (HMG/LEC) — delayed reference Cq or depressed efficiency flags the extract.
2. **Per-well efficiency screen** — any reaction falling > 2 robust standard deviations below its target's median is flagged for the confirmatory ENGL/EURL dilution inhibition test.

The ENGL/EURL test requires: slope ∈ [−3.6, −3.1], R² ≥ 0.98, and ΔCq(undiluted vs. extrapolated) < 0.5 cycle.

### 6. Cross-validation: LinRegPCR, qpcR and the instrument

Recovered curves are exported as RDML and fed unchanged to:
- **LinRegPCR** (Untergasser et al., 2021) via RDML-Python
- **qpcR** (Ritz & Spiess, 2008) four-parameter log-logistic fit

Agreement: Cq difference SD < 0.2 cycle; efficiency agreement within a few percentage points at R² ≈ 1.0.

### 7. QC rules, sample roles and acceptance framework

Sample roles are assigned by the first matching regex; unmatched samples remain **Sample**. Target-specific rules then decide, per role, whether a target must amplify in a stored Cq interval or must stay clean. A required rule with no matching wells = **missing evidence**; a matching well that fails = **QC failure**.

### 8. Provenance and integrity

Every input is SHA-256 hashed at load time; ZIP entries have CRC-32 verified. The evidence bundle contains source inputs, decoded files, checksums, graphs, CSVs, RDES/RDML, R script, report HTML, and a manifest with cryptographic fingerprints.

### References (thesis chapters)

- **Chapter V** — The .ixo Format: An Open Specification
- **Chapter X** — A Multi-Run Corpus: Scaling, Controls Validation and Provenance
- **Chapter XI** — Derived Quantities: Efficiency, Call Uncertainty and Reference-Gene Stability
- **Chapter XII** — Amplification Efficiency across the Corpus
- **Chapter XIII** — Longitudinal Quality Control: Control Charts for Run Controls
- **Chapter XIV** — Inhibition, Efficiency and the Limits of a Single Curve
- **Chapter XVI** — Quantitative Screening Verdicts: Direct-Ct and ΔCt

---

## Deployment

This repository is published with **GitHub Pages** from the `gh-pages` branch.

| Setting | Value |
|---------|-------|
| Source | Deploy from a branch |
| Branch | `gh-pages` |
| Folder | `/` (root) |
| URL | [https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/) |

To run locally, simply open `index.html` in any modern browser. No build step, no server, no dependencies.

---

## Browser compatibility

| Browser | Status |
|---------|--------|
| Chrome / Edge | ✅ Recommended |
| Firefox | ✅ Fully supported |
| Safari | ✅ Supported |
| Internet Explorer | ❌ Not supported |

---

## License and disclaimer

This is research tooling developed for GMO screening quality control. Use at your own risk for QC and data-review purposes. The software is provided as-is, without warranty of any kind. Always verify critical results against your laboratory's validated procedures and regulatory requirements.

---

*README generated for the `lc480-ixo-qc-export` repository — `gh-pages` branch.*
