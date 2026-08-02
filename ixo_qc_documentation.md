# LightCycler 480 `.ixo` QC Export — Documentation & Bug Report

## 1. What This Website Does

This is a **single-file HTML application** (`ixo_qc_export.html`, ~3,400 lines, ~200 KB) that reads **Roche LightCycler 480 `.ixo` experiment files** (and ZIP archives containing them) directly in the browser, extracts all qPCR data, and presents it in multiple review tabs:

| Tab | Purpose |
|-----|---------|
| **Values** | Raw Cq, concentration, Tm, melt-curve, genotype and relative-quantification tables |
| **Plate Map** | Heat-map visualisation of every well by result type |
| **Comparison** | Side-by-side comparison of two runs |
| **Statistics** | Within-run inhibition test, replicate CV, outlier flags |
| **Controls** | Levey-Jennings charts, Westgard rules for control series |
| **Curves** | Amplification and melt-curve rendering with export |
| **Instrument** | Run-level metadata, temperature log, forensic events |
| **Exports** | RDML, CSV, R script, Python script, QC bundle download |

The app is **completely client-side**: no server upload, no external dependencies. It parses the proprietary `.ixo` binary/XML hybrid format using only the browser's `DOMParser`, `DecompressionStream`, and `atob`.

---

## 2. The `.ixo` File Format

`.ixo` files are **XML object-streams** produced by Roche LightCycler 480 software. The file you supplied (`QT_MON810_PT_BIPEA_12.01.2026_VV.ixo`, ~1.8 MB) has this top-level structure:

```xml
<objectstream signature="IXOS" version="1">
  <obj name="root" class="HTCExperiment" version="5">
    <prop name="name">QT_MON810_PT_BIPEA_12.01.2026_VV</prop>
    <prop name="UID">632F09B8F4724114AC803CF62948BA3F</prop>
    ...
    <obj name="run" class="HTCRun" version="7">
      <obj name="Protocol" class="HTCProtocol" version="1">
        <obj name="Programs" class="HTCRunProgramList" version="5">
          <!-- thermal programs & segments -->
        </obj>
        <obj name="Container" class="HTCBlockType" version="1">
          <!-- plate geometry (96/384-well) -->
        </obj>
        <obj name="DetectionFormats" class="DetectionFormats" version="2">
          <!-- channel definitions (FAM, Hex, etc.) -->
        </obj>
      </obj>
      <obj name="TemperatureLog" class="TemperatureLog" version="2">
        <!-- run temperature trace -->
      </obj>
      ...
      <prop name="AcquisitionStore">eJzkvWuzqki3rftXdrxxvvlB...
        <!-- base64-encoded, deflate-compressed binary acquisition data -->
      </prop>
      ...
      <!-- Analysis objects: Absolute Quantification, Relative Quantification, Tm Calling, Genotyping, etc. -->
    </obj>
  </obj>
</objectstream>
```

### 2.1 Key Data Regions Inside an `.ixo`

| Region | XML Path / Class | What It Contains |
|--------|------------------|------------------|
| **Experiment metadata** | `HTCExperiment` | Name, UID, creation info, software version |
| **Run metadata** | `HTCRun` | Start/end time, instrument name, technician, plate ID |
| **Protocol** | `HTCProtocol` | Thermal programs, segments, channels, block type |
| **Plate model** | `GenSampleEditName`, `HTCSampleID`, `QuantSampleTypeProperty`, etc. | Sample names, types, concentrations per well/channel |
| **Subsets** | `SampleSubset` | User-defined well groups for analysis/reporting |
| **Acquisition store** | `prop[name="AcquisitionStore"]` | Base64 → deflate → raw fluorescence `Float32Array` per cycle/channel |
| **Analyses** | Various `*Analysis*` / `*Genotyping*` / `*Quantification*` classes | Cq, concentration, Tm, genotype calls, relative quant ratios |
| **Temperature log** | `TemperatureLog` | Instrument-reported block temperature over time |

### 2.2 The Acquisition Store (Raw Fluorescence)

This is the most complex part. The `AcquisitionStore` property holds a **base64 string** that, when decoded and inflated, becomes a flat XML stream of acquisition records:

```xml
<obj name="root" class="HTCAcquisitionStore" version="1">
  <prop name="ChannelCount">2</prop>
  <prop name="SampleCount">96</prop>
  <list name="Cycles" count="45">
    <obj name="Item" class="TCycle" version="1">
      <prop name="Program">1</prop>
      <prop name="Segment">2</prop>
      <prop name="Cycle">0</prop>
      <list name="Acquisitions" count="2">
        <obj name="Item" class="THTCFloAcquisition" version="3">
          <prop name="Temp">$4051F33333333333</prop>   <!-- hex-BE double -->
          <prop name="Channel">0</prop>
          <prop name="ScalingFactor">20000</prop>
          <prop name="FloPoints">K9PmRDXG5kSSt+lE...</prop>  <!-- base64 Float32Array -->
        </obj>
        ...
      </list>
    </obj>
  </list>
</obj>
```

The JavaScript parser (`decodeAcq`) scans this with a **single global regex** that extracts:
- `Program`, `Segment`, `Cycle` indices
- `Temp` as a 16-hex-char big-endian IEEE-754 double
- `Channel` number
- `ScalingFactor`
- `FloPoints` as a base64 blob decoded into `Float32Array` (one value per well)

The function then separates:
- **Amplification segment**: the segment with the most distinct cycles (typically the quantification segment)
- **Melt segments**: segments with temperature ramps (continuous temperature change > 2 °C or acquisition mode "2")

---

## 3. Parsing Pipeline (Data Flow)

```
User drops .ixo file(s)
         │
         ▼
┌─────────────────┐
│  intake()       │  ← accepts .ixo or .zip; reads as Uint8Array
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  decodeIxo()    │  ← main orchestrator
│  ├─ decodePlateModel()   → plate[pos] = {name, sampleId, channels:{...}}
│  ├─ decodeSubsets()      → subset definitions
│  ├─ decodeProtocol()     → programs, channels, block geometry
│  ├─ decodeAnalyses()     → list of analysis objects
│  ├─ decodeAcq()          → raw fluorescence channels + melt curves
│  └─ analysisResults()    → per-well result records per analysis
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  assignRoles()  │  ← marks wells as standard, unknown, positive/negative control
└────────┬────────┘
         │
         ▼
   Render tabs (Values, Plate, Statistics, Curves, etc.)
```

### 3.1 Key Functions

| Function | Lines | Responsibility |
|----------|-------|----------------|
| `decodeIxo(source, geomPref)` | ~1332–1648 | Master parser; returns a structured run object |
| `decodePlateModel(dom)` | ~952–1014 | Extracts sample names, types, concentrations, genotypes per well |
| `decodeProtocol(dom)` | ~1027–1078 | Extracts thermal programs, channel definitions, block geometry |
| `decodeAnalyses(dom)` | ~1205–1298 | Finds all analysis objects; determines kind (quant, tm, relquant, geno, etc.) |
| `analysisResults(an)` | ~1308–1331 | Extracts per-position result records from one analysis |
| `decodeAcq(raw, protocol)` | ~856–941 | Parses the binary acquisition store into cycle/channel/well matrices |
| `decodeRelQuant(an)` | ~1142–1192 | Parses relative-quantification gene groups, pairings, result groups |
| `readZip(ab, archive, filter)` | ~773–798 | Manual ZIP parser (no external library) |

---

## 4. Real Bugs Found

### 🐛 Bug 1: `analysisResults()` Silently Drops All Letter-Format Well Positions

**Location:** `ixo_qc_export.html`, inside `analysisResults(an)` (~lines 1308–1331)

**Code:**
```javascript
const posTxt = d.Pos !== undefined ? d.Pos : d.Position;
let pos;
if (/^[A-Za-z]/.test(posTxt)) {
    const m = posTxt.match(/^([A-Za-z]+)(\d+)$/);
    pos = m ? (m[1].toUpperCase().charCodeAt(0) - 65) * 24 + (parseInt(m[2], 10) - 1) : NaN;  /* refined by caller */
    pos = NaN;  // ← BUG: unconditionally overwrites the computed value
} else {
    pos = parseInt(posTxt, 10);
}
if (!Number.isFinite(pos)) return;  // ← NaN causes the result to be skipped entirely
```

**What happens:**
- Some `.ixo` files store well positions as text like `"A1"`, `"B12"` instead of numbers.
- The code correctly computes the 0-indexed numeric position on the first line.
- **The very next line unconditionally sets `pos = NaN`.**
- `!Number.isFinite(NaN)` is `true`, so the result is discarded with `return`.
- **Every single result with a letter-format position is silently lost.** No error is thrown; the data simply disappears.

**Secondary issue in the same block:** Even if the `pos = NaN` line were removed, the computation hardcodes `24` columns (384-well geometry). For a 96-well plate, `"B1"` would become `1×24+0 = 24` instead of `12`, placing it at `"C1"`.

**Fix:**
```javascript
if (/^[A-Za-z]/.test(posTxt)) {
    pos = wellToPos(posTxt, cols);  // use the existing wellToPos helper
} else {
    pos = parseInt(posTxt, 10);
}
```
*(Note: `cols` would need to be passed down to `analysisResults`, or `wellToPos` should be called inside `decodeIxo` after `cols` is known.)*

**Impact:** Low for your specific file (all positions are numeric `0`, `1`, `2`…), but **critical** for `.ixo` exports that use lettered positions (common in some LIMS integrations and older software versions).

---

### 🐛 Bug 2: `decodeSubsets()` Dedup Key Is Too Weak — Can Drop Distinct Subsets

**Location:** `ixo_qc_export.html`, inside `decodeSubsets(dom)` (~lines 1015–1026)

**Code:**
```javascript
const seen = new Set();
return out.filter(s => {
    const k = s.name + "|" + s.subsetId + "|" + s.items.length;
    if (seen.has(k)) return false;
    seen.add(k);
    return true;
});
```

**What happens:**
- The deduplication key is `name|subsetId|itemCount`.
- If two subsets have the **same name, same subsetId, and same number of items, but different wells**, the second one is incorrectly discarded.
- Example: Subset "Replicates" with ID "3" and 8 items `{0,1,2,3,4,5,6,7}` vs. another "Replicates" with ID "3" and 8 items `{48,49,50,51,52,53,54,55}`. The second would be lost.

**Fix:**
```javascript
const k = s.name + "|" + s.subsetId + "|" + s.items.join(",");
```

**Impact:** Low in practice (duplicate subsets with identical metadata but different items are rare), but it is a data-loss bug.

---

### 🐛 Bug 3: `decodePlateModel()` `posOf()` Returns `NaN` for Letter Positions, but Callers Only Guard Against `null`

**Location:** `ixo_qc_export.html`, inside `decodePlateModel()` (~lines 952–1014)

**Code:**
```javascript
const posOf = el => {
    const t = pv(el, "Position");
    return t === "" ? null : parseInt(t, 10);
};
// ...
dom.querySelectorAll('[class="GenSampleEditName"]').forEach(el => {
    const p = posOf(el);
    if (p == null) return;   // ← NaN != null, so this guard FAILS
    const v = pv(el, "name");
    if (v && !get(p).name) get(p).name = v;   // ← creates plate[NaN]
});
```

**What happens:**
- If a plate-model element (sample name, sample ID, etc.) has a letter-format `Position` like `"A1"`, `parseInt("A1", 10)` returns `NaN`.
- The caller checks `if (p == null) return;`, but **`NaN == null` is `false`**.
- The code then calls `get(NaN)`, which creates `plate[NaN]` — a well entry with a non-integer key that will never be matched by any analysis result.

**Fix:**
```javascript
const posOf = el => {
    const t = pv(el, "Position");
    if (t === "") return null;
    const n = parseInt(t, 10);
    return Number.isFinite(n) ? n : null;
};
```

**Impact:** Same as Bug 1 — only manifests with letter-format positions in the plate model. Your file uses numeric positions, so not triggered.

---

## 5. Verified-Working Areas (No Bugs Found)

| Component | Status | Notes |
|-----------|--------|-------|
| **ZIP parsing** | ✅ Correct | Manual EOCD scan, CRC-32 verification, deflate-raw decompression all verified |
| **Acquisition store decoding** | ✅ Correct | Regex parsing matches the file structure perfectly (tested by decoding your `.ixo`) |
| **Base64 / hex-BE double decoding** | ✅ Correct | `hexBEToDouble` and `base64ToBytes` produce valid numbers |
| **Channel indexing (active-only)** | ✅ Correct | Matches Roche's documentation: inactive channels do not consume an index |
| **Protocol / block geometry detection** | ✅ Correct | Falls back to 96-well when block type is missing |
| **Efficiency calculations** | ✅ Correct | Standard slope-based and sliding-window efficiency indices |
| **RDML / CSV / R / Python export** | ✅ Correct | Output schemas are well-formed |
| **QC statistics (replicate CV, inhibition, Westgard)** | ✅ Correct | Standard statistical formulas |

---

## 6. Summary Table

| # | Bug | Severity | Trigger | Fix Complexity |
|---|-----|----------|---------|----------------|
| 1 | `analysisResults()` discards letter-format positions | **High** | `.ixo` with `"A1"`-style positions | Low (3-line change) |
| 2 | `decodeSubsets()` weak dedup key loses distinct subsets | Medium | Two subsets with same name+ID+count but different wells | Low (1-line change) |
| 3 | `decodePlateModel()` `NaN` position leaks through `== null` guard | Medium | `.ixo` with letter-format plate-model positions | Low (2-line change) |

---

## 7. File Structure Reference

```
C:\Users\Ana si Mihai\Desktop\webpage\ixo_qc_export.html   ← the single-file web app
C:\Users\Ana si Mihai\Desktop\SOFT\Data\Institutul Seminte\Experiments\PROBE GM25\
  └─ QT_MON810_PT_BIPEA_12.01.2026_VV.ixo                   ← Roche LC480 experiment file
```

To use the app: open `ixo_qc_export.html` in any modern browser, drag-and-drop the `.ixo` file onto the page, and click **Read**.
