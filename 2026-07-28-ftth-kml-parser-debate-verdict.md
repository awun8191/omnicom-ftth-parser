# Multi-Brain Debate: KML FTTH Parser Architecture

## Question
Should the architecture be AI-only (with offline fallback) or hybrid heuristic+AI, given variable KML structure, production reliability requirements, and potential offline field use?

---

## 🧠 Round 1: Opening Positions

**Instance A — Creative:**
Build an **adaptive schema-inference engine** that treats the problem as two-pass: first, a lightweight embedding-based classifier maps unknown folder names to known semantic types using semantic similarity (not regex, not LLM). Second, a structural pattern miner discovers the coordinate/naming format within each classified section and auto-generates extraction rules. This is AI-light: one embedding model (no API calls) runs locally, costs nothing per parse, and adapts to any folder naming convention without LLM latency or cost. The LLM only fires when the embedding model encounters truly novel patterns below a similarity threshold.

**Instance B — Pragmatic:**
Ship a **heuristic-first, AI-fallback** system with explicit confidence gating. Heuristics use multi-strategy matching: folder name keyword scoring + child element geometry type analysis + coordinate format detection. Each classification produces a confidence score (0.0-1.0). Below 0.85, escalate to an LLM for reclassification. Build this in 2 weeks, deploy, and let production data reveal which patterns need tuning. The heuristic layer is simple XML path logic — it's fast, debuggable, and covers the common case. AI fallback costs $0.01-0.03 per escalated KML, which for a company processing 10-50 KMLs per month is negligible.

**Instance C — Comprehensive:**
Design a **multi-tier resilient architecture** that never single-point-fails. Tier 1 (fast path): heuristic classifiers with known pattern database. Tier 2 (fallback): small local model (sentence-transformers onnx, ~50MB) for semantic folder matching without API calls. Tier 3 (final fallback): LLM API call only when Tiers 1-2 disagree or both have low confidence. All tiers log their classifications for continuous improvement. The system degrades gracefully: Tier 1 covers 70% of cases instantly, Tier 2 covers 25% with local inference (100ms), Tier 3 covers the remaining 5% with API calls (10-15s). The architecture is offline-capable through Tier 2, so field engineers can parse KMLs without internet access.

---

## ⚔️ Round 2: Challenges

**A challenges B:** "Your heuristic-first approach assumes you know what patterns exist. You don't. You've analyzed one KML. Writing keyword matchers for engineer naming conventions you haven't seen yet is guessing, not engineering. The self-similarity embedder discovers patterns from the data itself — it doesn't impose assumptions."

**B challenges C:** "Three tiers is over-engineered for a tool that processes 10-50 files a month. You're adding an embedding model (ONNX deployment, 50MB binary, dependency management) and an LLM tier. That's two new infrastructure dependencies before you've even confirmed the problem exists at scale. Ship a simple heuristic+LLM system, measure actual fallback rates, then add Tier 2 only if needed."

**C challenges A:** "Your embedding model approach is elegant but untested for this specific domain. Sentence embeddings for *network engineering folder names* — does a general-purpose model like BGE or all-MiniLM understand that '03_SFC' means distribution cables, or that '01_FATs' contains fiber access terminals? The semantic gap between generic embeddings and domain-specific abbreviations could silently misclassify, which is worse than a heuristic that explicitly fails (low confidence → fallback)."

**Rebuttals:**
- **A responds:** "Conceded on domain-specific embeddings — but fine-tuning on 50 KML folder trees would close that gap. The offline capability alone is worth the initial training effort."
- **B responds:** "Conceded that heuristic-first assumes known patterns, but that's why confidence gating exists. Low confidence → AI fallback. The heuristic speed is a free win for the common case, not a bet that there IS a common case."
- **C responds:** "Conceded on over-engineering risk. The three-tier design can be built incrementally: ship Tier 1+3 first, add Tier 2 when offline demand materializes. Pre-optimizing for offline before confirming the need is premature."

---

## ⚖️ VERDICT

**Winner:** Instance C's three-tier architecture, built incrementally.

**Incorporated:**
- From C: The resilience tiers pattern — degrade gracefully, never hard-fail
- From B: Start with Tier 1 (heuristic) + Tier 3 (AI), add Tier 2 (local embedding) only if offline demand arises
- From A: The embedding classifier concept is valuable for Tier 2, but shouldn't be built before offline requirements are confirmed

**Eliminated:**
- A's pure embedding-first approach — too untested for domain-specific folder names without fine-tuning data
- B's suggestion that heuristic-only covers the common case without evidence

**Build order:**
1. **Phase 1:** Heuristic classifier (keyword scoring + geometry type analysis) → LLM fallback on low confidence
2. **Phase 2:** After 20+ KMLs processed, analyze classification logs to identify gap patterns
3. **Phase 3 (if offline needed):** Add local embedding model as Tier 2 using logged patterns as training data