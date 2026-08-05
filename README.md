# LC480 .ixo Tools Suite

A collection of **browser-based, client-side tools** for Roche LightCycler 480 `.ixo` experiment files. Every tool runs entirely in your browser — no server, no upload, no cloud dependency.

---

## 🌐 Live sites

| Path | Tool | Description | Best for |
|------|------|-------------|----------|
| [**/**](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/) | **QC Analyzer** (12PM) | GMO screening QC with sample roles, target-specific acceptance rules, per-well efficiency (WoL), replicate agreement, and evidence bundle export. | Routine GMO screening, plate QC, SOP compliance |
| [**/mpa/**](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/mpa/) | **Multiple Plate Analyzer** | Multi-plate study analysis: absolute & relative quantification, genotyping, Tm calling, melt curves, standard curves, t-tests, Hardy-Weinberg, signed PDF reports. | Research studies, method validation, cross-plate comparisons |
| [**/qc-export/**](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/qc-export/) | **QC Export Viewer** | Previous-generation LC480 viewer with plate heat-maps, Levey-Jennings control charts, Westgard rules, melt-curve rendering, and multi-tab inspection. | Historical reference, control-chart monitoring |

---

## Differences at a glance

### QC Analyzer (`/`)
- **Single-plate focus** with simple, fast workflow
- **Sample classification** by name (dropdown: exactly / hyphen exactly / every target / custom regex)
- **Target-specific QC rules** (must amplify at Cq X–Y, or must stay clean)
- **Efficiency** via window-of-linearity (WoL) with R² ≥ 0.98 gate
- **Replicate agreement** with configurable max Cq spread
- **Evidence bundle** with SHA-256 hashes, CSV, RDML, R script
- **Theoretical background** from thesis (Chapters V, X–XIV, XVI)

### Multiple Plate Analyzer (`/mpa/`)
- **Multi-plate studies** — load several plates and compare them
- **Quantification modes** — absolute quant, relative quant, Tm calling, genotyping, gene scanning
- **Standard curves** — dilution-series assay efficiency (MIQE-grade)
- **Statistics** — mean/SD or median/range, t-tests (paired, equal, Welch)
- **Advanced grouping** — sample-name regex grouping with property rules
- **PDF report generation** — signature-ready printed report
- **Hardy-Weinberg equilibrium** and χ² tests for genotyping
- **Melt-curve and amplification-curve** rendering per well

### QC Export Viewer (`/qc-export/`)
- **Legacy tool** kept for reference and backward compatibility
- **Plate map** heat-map visualization
- **Levey-Jennings** control charts with Westgard rules
- **Multi-tab inspection** — Values, Plate Map, Comparison, Statistics, Controls, Curves, Instrument, Exports
- **RDML, CSV, R and Python script** exports

---

## Supported formats

| Format | All tools | Notes |
|--------|-----------|-------|
| `.ixo` | ✅ | Roche LightCycler 480 experiment result files. XML-based with Base64-zlib-encoded binary fluorescence. |
| `.zip` | ✅ | Archives containing `.ixo` files. CRC-32 verified. RAR is **not** supported. |
| `.rdml` | Export only | Generated as output for cross-validation with LinRegPCR. |

---

## Quick start (any tool)

1. **Open the live site** — pick the tool from the table above.
2. **Load your data** — drag-and-drop `.ixo` files or a `.zip` archive onto the drop zone.
3. **Configure** — fill in study name, analyst, reviewer, and choose a preset (or use Advanced settings).
4. **Analyze** — click **Analyze** and review results.
5. **Export** — download evidence bundle, CSV, graphs, RDML, or R script.

---

## Theoretical background

All three tools share the same decoding pipeline and theoretical foundation:

- **`.ixo` format** — Single XML document rooted at `<objectstream>`; binary fluorescence in `AcquisitionStore` as Base64 → zlib → inner XML → float32.
- **Crossing points** — Stored `CrossingPoint` (Cp = Cq); `0.0` means non-amplification. Call codes: 0 = Negative, 1 = Uncertain, 2 = Positive, 3 = NotCalculated.
- **Efficiency** — Window-of-linearity (WoL) estimator: grid-search baseline β, select linear window (2 %–25 % amplitude), fit log₁₀(Fc − β) vs. cycle, report E = (10^slope − 1) × 100 % with gates R² ≥ 0.98 and 30 % < E < 140 %.
- **Inhibition** — Monitored via endogenous reference (HMG/LEC) Cq delay and per-well efficiency depression; confirmatory ENGL/EURL dilution test when flagged.
- **Cross-validation** — Recovered curves exported as RDML for LinRegPCR and qpcR; agreement SD < 0.2 cycle on Cq, within a few % on efficiency.

For the full theoretical background, open the **Help & methods** panel inside the **QC Analyzer** or **Multiple Plate Analyzer**.

---

## Browser compatibility

| Browser | Status |
|---------|--------|
| Chrome / Edge | ✅ Recommended |
| Firefox | ✅ Fully supported |
| Safari | ✅ Supported |
| Internet Explorer | ❌ Not supported |

---

## Deployment

This repository is published with **GitHub Pages** from the `gh-pages` branch.

| Setting | Value |
|---------|-------|
| Source | Deploy from a branch |
| Branch | `gh-pages` |
| Folder | `/` (root) |
| Base URL | [https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/](https://mihaiionitabio-alt.github.io/lc480-ixo-qc-export/) |

To run any tool locally, simply open its `index.html` in any modern browser. No build step, no server, no dependencies.

---

## File layout (`gh-pages` branch)

```
/
├── index.html              # QC Analyzer (12PM)
├── README.md               # This file
├── ixo_qc_documentation.md # Technical documentation
├── ixo_qc_export.html      # QC Export Viewer (source)
├── mpa/
│   └── index.html          # Multiple Plate Analyzer
└── qc-export/
    └── index.html          # QC Export Viewer (deployed)
```

---

## License and disclaimer

This is research tooling developed for GMO screening quality control and qPCR data analysis. Use at your own risk for QC and data-review purposes. The software is provided as-is, without warranty of any kind. Always verify critical results against your laboratory's validated procedures and regulatory requirements.

---

*README for the `lc480-ixo-qc-export` repository — `gh-pages` branch.*
