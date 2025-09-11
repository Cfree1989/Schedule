# System Audit Report

## 🚨 URGENT ISSUES
- No critical security, data loss, or production-breaking issues identified in the current single-file implementation. Notable considerations documented in pass summaries (e.g., memory pressure on large PDF captures; CDN dependency for exported files).

## Executive Summary
- This is a single-file HTML application implementing a student scheduling UI with embedded data, local persistence, single-file export, and a two-page PDF export (grid + totals). Architecture is clear and modular within the constraints of a single file: a `DataManager` for state/persistence, a `UI` module for rendering and interaction, and public functions for user actions.
- Accessibility and keyboard navigation are thoughtfully addressed; totals sorting is persistent and mirrored in PDF. State precedence is sane: local app state → embedded JSON → legacy/hybrid fallbacks → defaults.
- Primary risks are operational rather than security: PDF capture can be memory-intensive; exported HTML depends on external CDNs for libraries when opened offline. Both are mitigated with fallbacks or acceptable trade-offs.

## Pass 1: Flask Backend Analysis
This project has no backend. For completeness, this pass audits data/state management and business logic implemented in JavaScript (client runtime) instead.

### Health Snapshot
- Complexity Score: 4/10
- Maintainability: Moderate; clear modularization within a single file
- Overall Risk: Low
- Bus Factor: 3/5 (logic readable; single-file concentration helps)

### Technical Findings
- Data ownership and persistence are centralized in `DataManager` with load/save and migration paths.
- Sorting state for totals is stored in `settings.totalsSort`, ensuring consistent UI/PDF ordering.
- Embedded bootstrap JSON is respected and replaced on export.
- Legacy/hybrid migrations are guarded with safe JSON parsing.

Code references:
```580:1866:c:\Users\cfree3\OneDrive - Louisiana State University\Lab\Admin\Templates\Schedule\student-schedule-app.html
const DataManager = {
  state: null,
  computeTotals(state) { /* ... */ },
  getTotalsSort() { /* ... */ },
  setTotalsSort(next) { /* ... */ },
  ensureOrderPerDay(studentsArr) { /* ... */ },
  getEmbedded() { /* ... */ },
  migrateLegacy(legacySchedule, legacyPeople) { /* ... */ },
  defaults() { /* ... */ },
  load() { /* ... */ },
  save() { /* ... */ }
};
```

### Issues by Severity
- Critical: None
- High: None
- Medium:
  - Exported single-file depends on CDN libraries; offline usage may degrade PDF/export functionality.
- Low:
  - `orderPerDay` persistence relies on re-render after drag-end; acceptable for size, but could be optimized.

### Cross-Cutting Concerns Identified
- Robust safe JSON parsing via `parseJSONSafe` used across loaders.
- Consistent state precedence and schema evolution hooks present (migration and defaults).

## Pass 2: React Frontend Analysis
There is no React here. For completeness, this pass audits the UI layer (DOM rendering, accessibility, interaction, and PDF export).

### Health Snapshot
- Complexity Score: 5/10
- Maintainability: Moderate; UI is cleanly separated from data; strong keyboard/accessibility patterns
- Overall Risk: Low–Medium (memory considerations for canvas rendering)
- Bus Factor: 3/5

### Technical Findings
- UI initialization wires theme system, gutters, week grid, totals, buttons, and modals. Roving tabindex with arrow-key navigation is implemented. Sorting UI uses persistent state and `aria-sort` attributes.
- Per-day column reordering uses SortableJS on legend chips; `orderPerDay` updates on drop.
- PDF export: Page 1 uses html2canvas to rasterize the grid with a fallback scale; Page 2 uses jsPDF-AutoTable with header/footer via `didDrawPage`. UI totals and PDF totals parity are considered.

Code references:
```842:1580:c:\Users\cfree3\OneDrive - Louisiana State University\Lab\Admin\Templates\Schedule\student-schedule-app.html
const UI = {
  init() { /* theme, gutters, week, totals, actions */ },
  renderTotals() { /* sortable headers, tfoot totals */ },
  renderWeek(students) { /* grid, painting, keyboard nav, SortableJS */ },
  openManageStudentsModal() { /* list + edit/delete flows */ },
  setTheme(themeName) { /* data-theme + persist */ }
};
```

```1600:1766:c:\Users\cfree3\OneDrive - Louisiana State University\Lab\Admin\Templates\Schedule\student-schedule-app.html
window.exportToPDF = async function exportToPDF() {
  const { jsPDF } = window.jspdf || {};
  // html2canvas capture with fallback scale
  // AutoTable with header/footer and totals row
};
```

```1769:1796:c:\Users\cfree3\OneDrive - Louisiana State University\Lab\Admin\Templates\Schedule\student-schedule-app.html
window.exportSchedule = function exportSchedule() {
  // Clone document, replace embedded JSON with current state, download single-file HTML
};
```

### Issues by Severity
- Critical: None
- High: None
- Medium:
  - html2canvas can trigger high memory usage on very wide/tall grids; a 1.5x fallback is implemented, which mitigates most cases.
- Low:
  - Last column styles in AutoTable are minimal (intentional); cosmetic only.

### Cross-Cutting Concerns Identified
- Accessibility consistently addressed: roles, aria attributes, keyboard model, and focus management.
- Sorting logic is shared in concept between UI and PDF (consistent order).

## Pass 3: Infrastructure Analysis
Single-file app with CDN dependencies; no Docker/build tooling involved here.

### Health Snapshot
- Complexity Score: 3/10
- Maintainability: Easy (single file)
- Overall Risk: Low
- Bus Factor: 4/5

### Technical Findings
- External libraries loaded via CDN:
  - jsPDF UMD, jsPDF-AutoTable, html2canvas, SortableJS.
- Exported HTML will retain CDN references; fully offline environments may lack these resources.
- Local persistence uses `localStorage`; size is modest for this use-case.

### Issues by Severity
- Critical: None
- High: None
- Medium:
  - Availability risk if CDNs are unreachable when opening exported HTML offline. Consider documenting this trade-off.
- Low:
  - No build-time integrity checks (SRI) on CDN links. Consider adding `integrity`/`crossorigin` for additional safety.

### Cross-Cutting Concerns Identified
- Version pinning for CDN URLs is explicit (good for reproducibility).

## Pass 4: File Handling Analysis
Focus on state storage, export flow, and PDF generation.

### Health Snapshot
- Complexity Score: 5/10
- Maintainability: Moderate
- Overall Risk: Low–Medium
- Bus Factor: 3/5

### Technical Findings
- State precedence and persistence:
  - Load order: `appState` (local) → embedded JSON → hybrid/legacy → defaults.
  - Save writes to `localStorage` with try/catch; failures are warned, not fatal.
- Export flow:
  - Clones DOM, injects current state into `#embedded-data`, serializes to single-file HTML, downloads with date-based filename.
- PDF flow:
  - Page 1: Raster capture of the grid with scaling and fallback.
  - Page 2: Programmatic totals table with footer row and page header/footer using AutoTable.

Code references:
```170:207:c:\Users\cfree3\OneDrive - Louisiana State University\Lab\Admin\Templates\Schedule\student-schedule-app.html
<script id="embedded-data" type="application/json">{ /* bootstrap state */ }</script>
```

```1769:1796:c:\Users\cfree3\OneDrive - Louisiana State University\Lab\Admin\Templates\Schedule\student-schedule-app.html
exportSchedule() { /* replace embedded JSON with current state and download */ }
```

### Issues by Severity
- Critical: None
- High: None
- Medium:
  - Exported file’s reliance on CDNs for libraries can limit portability in offline contexts.
- Low:
  - No compression of embedded JSON (file size trade-off). Acceptable at current scale.

### Cross-Cutting Concerns Identified
- Defensive JSON parsing across all load paths.
- Consistent 0.5-hour slot accounting between UI and PDF.

## Overall Health Assessment
- System-wide complexity score: 4.25/10 (average of passes)
- Maintainability rating: Easy–Medium
- Breaking point warning: Very large numbers of students/slots could stress html2canvas during PDF capture; mitigated by fallback scale.
- Top cross-cutting issues:
  1) CDN availability for exported HTML (Medium). 
  2) Memory pressure on PDF grid capture for very large content (Medium).
  3) Minor styling consistency for PDF column widths (Low).
  4) JSON size uncompressed in export (Low).
- Architecture coherence: Strong for a single-file app—clear separation of data vs UI, consistent patterns, and thoughtful accessibility.

## Task Board
### 🚨 CRITICAL (Do Immediately)
- [ ] No critical items.

### 🔥 HIGH PRIORITY (This Week)
- [ ] Document CDN trade-off in a short “Offline Usage” note in the UI or README-equivalent section. | Risk: Medium | Effort: S | Files: `student-schedule-app.html` | Why: Avoid confusion when opening export without internet.

### 📋 MEDIUM PRIORITY (Next 2 Weeks)
- [ ] Add SRI `integrity` and `crossorigin` attributes to CDN script tags. | Risk: Medium | Effort: S | Files: `student-schedule-app.html` | Dependencies: none
- [ ] Optional: Provide an inline-libs export mode (embed library code) for fully offline use. | Risk: Medium | Effort: M | Files: `student-schedule-app.html` | Dependencies: design decision

### 🔧 LOW PRIORITY (Technical Debt)
- [ ] Tweak AutoTable columnStyles to include the last column explicitly if desired. | Risk: Low | Effort: XS | Files: `student-schedule-app.html` | Nice-to-have because: visual polish
- [ ] Optionally minify embedded JSON on export for smaller files. | Risk: Low | Effort: XS | Files: `student-schedule-app.html`

## Success Metrics
- [ ] Exported HTML’s behavior is clearly communicated for offline vs online contexts.
- [ ] PDF export remains stable across typical grid sizes with no crashes; fallback path exercised.
- [ ] Consistent ordering and values between UI totals and PDF totals for representative test cases.
- [ ] Accessibility remains intact: keyboard navigation and `aria-sort` indicators verified.


