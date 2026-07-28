# KML/KMZ FTTH Network Parser — Spec

## Problem
KML/KMZ files from Omnicom (Huawei subcontractor) contain FTTH network design data:
FATs, Hub Boxes, Poles (8M/6M/6.5M), SFC/Distribution Cables, Feeder Cables,
CL (closure) points, OLT, and Buildings. However, different engineers structure
the XML differently — folder names, nesting depth, coordinate formats vary.

## Requirements
1. Accept any KML/KMZ file containing FTTH network data
2. Identify which folders contain: FATs, Hub Boxes, Poles, SFC cables, Feeder cables
3. Extract coordinates for all placemarks
4. Parse naming conventions (H\d+L\d+S\d+ for FATs, \d+M-[CS]-\d+ for poles)
5. Compute cable distances (haversine)
6. Categorize SFC links into: (a) HB→first FAT per line, (b) FAT→FAT cascade
7. Map feeder hierarchy: OLT → CL → HB → FAT
8. Detect co-mounted FATs (sub-10m distances = same pole)
9. Output structured data (JSON)
10. Handle completely different folder naming/structure via flexible classifier

## Approaches to Evaluate
| Approach | Description | Cost | Reliability |
|----------|-------------|------|-------------|
| A. Pure regex/heuristics | Hard-coded pattern matching for folder identification | Free | Brittle |
| B. Pure AI | LLM parses entire XML each time | $$$ | Flexible but slow/expensive |
| C. Hybrid: AI classifies → code extracts | AI identifies sections, code extracts & computes | $ | High |
| D. Hybrid: Heuristic + AI fallback | Try heuristics first, fall back to AI on mismatch | $ | High |

## Output
Structured JSON with: pole data, HB data, FAT data (with naming), cable links
(with distances per segment), topology tree, and metrics summary.