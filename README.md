# LC480 ixo QC Export

A browser-based quality-control viewer for **Roche LightCycler 480** `.ixo` experiment files.

## What it does

Open `ixo_qc_export.html` in any modern browser, drag-and-drop your `.ixo` (or ZIP archive containing `.ixo`) files, and instantly get:

| Tab | Content |
|-----|---------|
| **Values** | Cq, concentration, Tm, melt-curve, genotype, and relative-quantification tables |
| **Plate Map** | Heat-map of every well by result type |
| **Comparison** | Side-by-side run comparison |
| **Statistics** | Inhibition test, replicate CV, outlier flags |
| **Controls** | Levey-Jennings charts with Westgard rules |
| **Curves** | Amplification and melt-curve rendering |
| **Instrument** | Run metadata, temperature log, forensic events |
| **Exports** | RDML, CSV, R script, Python script, QC bundle |

Everything is **client-side** — no data leaves your computer.

## File overview

| File | Description |
|------|-------------|
| `ixo_qc_export.html` | The single-file web application (~3,400 lines, ~200 KB) |
| `ixo_qc_documentation.md` | Full documentation of the parsing pipeline and known bugs |

## Usage

1. Open `ixo_qc_export.html` in Chrome, Firefox, Edge, or Safari.
2. Drag your `.ixo` file onto the drop zone (or click **Select files**).
3. Click **Read**.
4. Browse the tabs.

## Supported formats

- `.ixo` — Roche LightCycler 480 experiment files
- `.zip` — archives containing `.ixo` files
- Pasted CSV/TSV — for cross-instrument inhibition and replicate checks

## Documentation

See [`ixo_qc_documentation.md`](ixo_qc_documentation.md) for:
- Complete `.ixo` format breakdown
- Parsing pipeline architecture
- Function reference
- **3 real bugs identified** in the JavaScript parser

## License

This is research tooling. Use at your own risk for QC and data-review purposes.
