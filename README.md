# Omnicom FTTH Parser

FTTH KML/KMZ network data parser — extracts fiber access terminals, hub boxes, poles, SFC/distribution cables, and feeder routes from variable-format KMZ design files produced by Huawei subcontractor field engineers.

## Problem

KML/KMZ files from Omnicom engineers carry identical FTTH network data (FATs, HBs, Poles, SFC cables, feeders, closures, OLT) but the XML folder structure, naming, nesting depth, and description formatting vary per engineer. The tool must handle this variation reliably in production — wrong extractions lead to real-world fiber deployment errors (wrong cable lengths, misassigned poles, incorrect FAT counts).

## Architecture (After UltraThink Analysis)

```
KML/KMZ → [1] AI Classification → [2] Code Extraction → [3] Naming Parser → [4] Computation → JSON
```

| Layer | What | Method | Failure Risk |
|-------|------|--------|-------------|
| **1. Section Classifier** | Identify which folder contains FATs, Poles, SFCs, HBs, Feeders | **AI** (LLM with structured output) | Low — AI handles naming variation |
| **2. Data Extractor** | Pull coordinates, geometries, descriptions from known XML paths | **Code** (lxml ElementTree) | Near-zero — KML is always valid XML |
| **3. Naming Parser** | Parse conventions like `H10L1S3`, `8M-C-83` | **Regex** (confined to stable CAD patterns) | Low — naming from CAD export is stable across engineers |
| **4. Computation** | Haversine distances, roll-ups, topology tree | **Code** (math, graph traversal) | Near-zero |

### Naming Convention Patterns (from analyzed KML)

```
FATs:     EWU_C1_Z1_H{hub}L{line}S{sub}     e.g. EWU_C1_Z1_H10L1S3
Poles:    EWU-C1_Z1-{size}M-{type}-{num}    e.g. EWU-C1_Z1-8M-C-83
HBs:      EWU_C1_Z1_H{num}                   e.g. EWU_C1_Z1_H10
SFC/DC:   {SOURCE} - {DEST}                  e.g. EWU_C1_Z1_H10 - H10L1S1
Feeders:  From {SOURCE} To {DEST}            e.g. From T0468 Rack1 ODU3 To EWU_C1_Z1_H19
```

### Data Extracted per KML

| Element | Count (This Site) |
|---------|-------------------|
| Hub Boxes | 19 |
| FATs | 286 |
| SFC/DC Cables | 286 (72 HB→FAT + 214 FAT cascade) |
| Poles (8M/6M/6.5M) | 283 (71 + 199 + 13) |
| Existing Buildings | 1,036 |
| Total DC length | 12.67 km |
| Total Feeder | 6.46 km (2 from OLT, 3 from CL) |

System topology: OLT → Feeder Cable → Hub Box → DC Cable → FAT (on Pole) → Customer

## Build Phases

### Phase 1: Core Pipeline
- AI classifier that reads KML folder tree → maps to known section types
- Code extractor: pull all Placemarks + coordinates from classified paths
- Naming parser: extract FAT/HB names, pole types, cable source/destination
- Distance calculator: haversine per cable segment
- Output: structured JSON with all extracted data

### Phase 2: Production Hardening
- Classification confidence scoring + logging
- Human-in-the-loop: show AI classification for user confirmation on first-run KMLs
- Cached pattern maps: once AI classifies a folder structure, cache the mapping for fast re-parsing

### Phase 3: Scale & Monitor (office-only, always-online)
- Confidence tracking across all processed KMLs
- Build pattern cache from AI classifications to accelerate re-parsing of same-site KMLs
- Continual improvement: flag low-confidence classifications for human review

## Site Data: Ewutuntun Cluster

The first KML analyzed is an FTTH design for **Ewutuntun**:

```
OLT (T0468)
  ├── 96C (3.28km) → H13
  │   └── CL_1, CL_2 → H10, H11, H16
  └── 48C (2.36km) → H19
        └── Remaining HBs (H1-H12, H14-H18)
              ├── Line 1 → S1 → S2 → S3 → S4
              ├── Line 2 → S1 → S2 → S3 → S4
              ├── Line 3 → S1 → S2 → S3 (or S4)
              └── Line 4 → S1 → S2 → S3 → S4
```

Each HB feeds 2-4 lines via DC cables (avg 70m to first FAT, then 36m between cascading FATs). ~11 HBs co-locate one line within 1m of the HB itself (same pole). Co-mounted FATs (sub-10m) share a pole.

## Reference

See `2026-07-28-ftth-kml-parser-*.md` for the full UltraThink analysis pipeline (Spec, Council Synthesis, Debate Verdict, Deep Thinker Review).