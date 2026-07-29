# Engineering Review — FTTH Closure Automation Tool

Generated: 2026-07-28
Branch: master
Status: COMPLETE

---

## Architecture

| Layer | Technology |
|-------|------------|
| Frontend | React (Vercel) |
| API | FastAPI (Google Cloud Run) |
| Auth | Firebase Auth |
| Storage | Firebase Storage (KMLs, photos, templates) |
| Database | Firestore (components, tasks, projects, deliverables) |
| Background Jobs | Cloud Tasks → FastAPI worker (60-min timeout, auto-retries) |
| Real-time Progress | Firestore `onSnapshot()` |
| AI Model | **Gemini 3.5 Flash-Lite** (all tasks) |

---

## Data Model (Firestore)

```
/companies/{companyId}
  - name, createdAt, plan, ownerUid
  /projects/{projectId}
    - name, kmlRef, status, createdAt
    /components/{componentId}     # FATs, hubs, poles, cables from KML
      - type: "fat" | "hub" | "pole" | "cable" | "feeder"
      - name: "EWU_C1_Z1_H10L1S1"
      - geometry: { lat, lng } | { from, to }
      - expectedUnits, actualUnits?, actualLengthM?
      - source: "kml" | "manual"
    /photos/{photoId}
      - componentRef, type: "pole" | "opm"
      - storagePath, mimeType, sizeBytes
      - classification: { poleType: "concrete"|"steel", confidence }
      - matchedComponentId?
    /templates/{templateId}
      - name, pdfStoragePath, boundingBoxes[]
    /tasks/{taskId}
      - type: "kml_parse" | "photo_classify" | "pdf_generate" | "excel_generate"
      - status, progress, lastCheckpoint, error
    /deliverables/{deliverableId}
      - type: "excel" | "pdf"
      - storagePath, generatedAt
  /users/{uid}
    - role: "admin" | "engineer"
    - projectIds[]
```

---

## KML Classification Pipeline (AI-First, Batched)

```
Step 0 — Clean KML (pre-processing)
  → Remove all Style, StyleMap, IconStyle, LineStyle, PolyStyle elements
  → Remove GroundOverlay, ScreenOverlay, NetworkLink
  → Remove gx:Tour, gx:Playlist, gx:MultiTrack
  → Remove extended data not needed for extraction
  → Keep only: Document, Folder, Placemark, name, Point, LineString, Polygon, coordinates, description
  → Output: cleaned KML (~80% size reduction)

Step 1 — Parse KML structure (lxml)
  → Extract all Placemarks with coordinates, names, folders
  → Group by folder/layer type (heuristic: folder name patterns)
  → Output: sections = [{ type: "sfc_cables", items: 283 }, ...]

Step 2 — For each section, process in batches of 50
  → Batch 1: items 0-49 → Gemini 3.5 Flash-Lite with structured output schema
  → Batch 2: items 50-99 → ...
  → Each batch: checkpoint saves { section, batchIndex, extractedCount }
  → On failure: resume from last completed batch

Step 3 — Merge all batches → write to /components
  → Each component: { type, name, geometry, expectedUnits, source: "kml", batchRef }
```

**Cost per site (Ewutuntun scale): ~$0.15-0.20**

---

## Photo Pipeline (Hybrid Batch + AI)

1. Engineer uploads batch: "these 15 are OPM tests for these FATs"
2. AI classifies within batch:
   - Pole type (concrete/steel) — confidence gated
   - OPM reading extraction — confidence gated
3. Low confidence → flagged for manual review in UI
4. Matched component ID written to photo doc

---

## PDF Generation (Bounding Box UX)

- User uploads Huawei PDF template
- UI: draw bounding boxes per component type per page
- System: auto-fits uploaded photos into those boxes
- Template-agnostic — works with any Huawei template version

---

## Excel Generation

- Template-based from KML pre-population
- Columns: Deliverable Type, Name, Expected Units, Actual Units, Cable From, Cable To, Length (m), Notes
- Engineer fills only actuals; expected comes from KML

---

## API Contracts (FastAPI)

### Upload & Processing
```
POST   /projects/{projectId}/kml/upload
POST   /projects/{projectId}/photos/upload
GET    /tasks/{taskId}
POST   /tasks/{taskId}/retry
```

### Data Access
```
GET    /projects/{projectId}/components
PATCH  /components/{componentId}
GET    /projects/{projectId}/photos
POST   /photos/{photoId}/match
```

### Deliverables
```
POST   /projects/{projectId}/excel/generate
POST   /projects/{projectId}/pdf/generate
GET    /deliverables/{deliverableId}/download
```

### Templates
```
POST   /templates/upload
POST   /templates/{templateId}/bounding-boxes
```

---

## Auth / Multi-Tenant

- Firebase Auth (email/password + Google)
- Firestore Security Rules:
  ```
  match /companies/{companyId}/{document=**} {
    allow read, write: if request.auth.uid == resource.data.ownerUid
                        || request.auth.uid in resource.data.memberUids;
  }
  ```
- FastAPI middleware: verifies Firebase ID token, attaches `companyId` to request state

---

## Billing / Metering

| Layer | Approach |
|-------|----------|
| Infrastructure | GCP billing alerts at $50, $100, $200 |
| AI API calls | Internal logging per company (ops only, not customer-facing) |
| Customer pricing | **Flat ₦100-150K/mo** — unlimited sites, KMLs, reasonable WhatsApp volume |
| Abuse protection | Soft caps: 20 KMLs/mo, 500 WhatsApp extractions/mo per company (configurable) |

---

## Model Selection (Locked)

| Task | Model | Provider | Cost/1M output |
|------|-------|----------|----------------|
| KML classification (batched) | **Gemini 3.5 Flash-Lite** | Google AI Studio / Vertex | $2.50 |
| Photo classification (pole/OPM) | Gemini 3.5 Flash-Lite | Same | $2.50 |
| OPM reading OCR | Gemini 3.5 Flash-Lite | Same | $2.50 |
| WhatsApp extraction (v2) | Gemini 3.5 Flash-Lite | Same | $2.50 |

**Rationale**: Cheap, fast, free credits in AI Studio for dev, structured output support, 1M context window.

---

## Testing Strategy

| Layer | What | Tool | Target |
|-------|------|------|--------|
| Unit | KML parsing, distance calc, Excel row gen | pytest | 90%+ |
| Integration | Cloud Task → worker → Firestore | pytest + Firebase emulators | Happy + failure paths |
| Integration | Photo upload → classify → match | pytest + mocked Gemini | 80%+ |
| Integration | PDF gen with bounding boxes | pytest + pdfplumber | Template fidelity |
| Contract | OpenAPI schema validation | schemathesis | All endpoints |
| E2E | Full closure flow: KML → Excel + PDF | Playwright | 3 critical paths |
| Load | 50 concurrent KML parses | Locust | < 2 min p95 |

**Test fixtures**: Ewutuntun KML (2,280 pts), sample Huawei PDF, pole/OPM photos, WhatsApp exports

---

## Open Questions for Implementation

1. **KML folder variance** — AI-first classification handles this, but initial folder grouping heuristics need tuning
2. **WhatsApp v2** — deferred; data model ready (`/messages` subcollection)
3. **Security rules debugging** — start with application-level checks, migrate to rules
4. **AI structured output schema** — finalize JSON schema for each task type

---

## Next Steps (When Building)

1. Scaffold FastAPI + React monorepo
2. Implement KML parser (lxml) + Cloud Task worker
3. Build React upload UI + Firestore listener for progress
4. Add Gemini 3.5 Flash-Lite client with structured output
5. Implement batched classification with checkpoint resume
6. Build photo upload → classification → matching flow
7. PDF template upload + bounding box drawer
8. Excel template generation from KML data
9. Auth middleware + security rules
10. E2E test with Ewutuntun KML