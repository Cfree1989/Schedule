# Schedule Masterplan

## 1) Executive Summary
This document is the authoritative guide to the Student Worker Schedule single-file web app. It covers current features, architecture, data model, UI/UX patterns, accessibility, and extension guidance, with code citations to `student-schedule-app.html`.

## 2) Feature Inventory (Current)
- Header actions (buttons): Save, Clear Times, PDF, Download (single-file HTML), Add Student
- Schedule grid: Monday–Friday, 8:30–4:30 in 30-minute slots; per-student columns with color chips; interactive cells
- Paint selection: click-drag to assign/erase contiguous ranges; Alt forces erase; column-locked painting; live hours indicator
- Hours summary: per-student daily and weekly totals + daily totals row
- Data bootstrap: embedded JSON inside the HTML; default seed if not present
- Persistence: localStorage cache (appState) and legacy migration support
- PDF export: Page 1 grid capture (html2canvas), Page 2 totals (jsPDF-AutoTable) with header/footer and parity check vs UI
- Single-file export: produce a self-contained HTML with current state embedded
- Accessibility/Keyboard: roving tabindex, arrow keys navigation, Home/End, PageUp/PageDown, Space/Enter activation; ARIA roles

## 3) System Architecture
- Single-file app: HTML + CSS + vanilla JS, no build step
- Modules:
  - DataManager: state loading, persistence, totals computation, legacy migration
  - UI: initial rendering, gutters, week grid, totals, student modal, event wiring
  - Public actions on `window`: save, clear, export to PDF, export HTML, add student
- CDN libraries: `jsPDF`, `html2canvas`, `jsPDF-AutoTable`

Code citations:
```504:880:student-schedule-app.html
// UI module and core rendering/event wiring
```
```393:502:student-schedule-app.html
// DataManager core (load/defaults/save/totals/migration)
```
```367:371:student-schedule-app.html
// UMD libraries: jsPDF, AutoTable, html2canvas
```

## 4) Data Model & Persistence
- Schema (embedded JSON in `<script type="application/json" id="embedded-data">`):
```350:365:student-schedule-app.html
{
  "version": "1.0.0",
  "metadata": {"createdAt": "${new Date().toISOString()}", "source": "embedded"},
  "settings": { "headerTitle": "STUDENT WORKER SCHEDULE | 2024–2025" },
  "students": [ {"id": "stu-1", "name":"Jane Doe", "code":"JD", "role":"PA", "colorIndex":0}, ... ],
  "schedule": {}
}
```
- Storage precedence when loading state:
  1) Embedded JSON (if present)
  2) Hybrid `localStorage` (legacy tracked format)
  3) Legacy keys: `studentWorkerSchedule`, `studentWorkerPeople` → migration
  4) App-state cache `studentScheduleAppState`
  5) Defaults (mirrors embedded)

```459:493:student-schedule-app.html
// DataManager.load(): embedded → hybrid → legacy → appState → defaults
```

- Totals computation (0.5h per slot, sparse schedule map keyed by day|slot|studentId):
```395:418:student-schedule-app.html
computeTotals(state) {
  // Builds per-student rows, daily totals, and grand total
}
```

## 5) UI/UX & Accessibility
### 5.1 Header layout and actions
HTML structure:
```276:286:student-schedule-app.html
<header class="app-header" role="banner">
  <div class="header-title" id="headerTitle" aria-live="polite">STUDENT WORKER SCHEDULE | 2024–2025</div>
  <div class="header-actions" role="toolbar" aria-label="Schedule actions">
    <button class="icon-btn" id="btnSave">Save</button>
    <button class="icon-btn danger" id="btnClear">Clear Times</button>
    <button class="icon-btn primary" id="btnPDF">PDF</button>
    <button class="icon-btn" id="btnDownload">Download</button>
    <button class="icon-btn" id="btnAddStudent">Add Student</button>
  </div>
</header>
```
Event wiring:
```516:525:student-schedule-app.html
document.getElementById('btnSave').addEventListener('click', window.saveSchedule);
document.getElementById('btnClear').addEventListener('click', window.clearTimes);
document.getElementById('btnPDF').addEventListener('click', window.exportToPDF);
document.getElementById('btnDownload').addEventListener('click', window.exportSchedule);
document.getElementById('btnAddStudent').addEventListener('click', () => this.openStudentModal());
```
Keyboard shortcut (Alt+P → PDF):
```539:545:student-schedule-app.html
window.addEventListener('keydown', (e) => {
  if (e.altKey && (e.key === 'p' || e.key === 'P')) {
    e.preventDefault();
    window.exportToPDF();
  }
});
```

### 5.2 Layout and styling system
- Color tokens and sizing:
```8:29:student-schedule-app.html
:root { /* theme colors, slot height, radius, grid line thickness */ }
```
- App shell and main grid:
```41:52:student-schedule-app.html
.app { display: grid; grid-template-rows: auto 1fr auto; min-height: 100dvh; }
.app-main { display: grid; grid-template-rows: auto auto; gap: 8px; padding: 8px 16px; }
```
- Schedule shell (3 columns: left gutter, week grid, right gutter) with bottom alignment:
```94:101:student-schedule-app.html
.schedule-shell { display: grid; grid-template-columns: minmax(60px,80px) 1fr minmax(60px,80px); gap: 8px; align-items: end; max-height: 45vh; overflow: auto; }
```
- Time gutters (16 rows; sticky; aligned to grid rows):
```103:121:student-schedule-app.html
.time-gutter, .time-gutter-right { /* sticky, 16 rows, borders, rounded corners */ }
```
- Week grid and day blocks:
```138:157:student-schedule-app.html
.week-grid { grid-template-columns: repeat(5, 1fr); gap: 4px; }
.day-block { grid-template-rows: var(--slot-h) var(--slot-h) auto; border: var(--grid-line) solid var(--border); border-radius: var(--corner-r); }
```
- Legend and schedule rows/cells with single-line borders and hover/focus:
```176:226:student-schedule-app.html
.legend-row { grid-auto-flow: column; grid-auto-columns: minmax(20px,1fr); }
.schedule-rows { grid-template-rows: repeat(16, var(--slot-h)); }
.slot-cell { border-top: var(--grid-line) solid var(--border); border-right: var(--grid-line) solid var(--border); cursor: pointer; }
.slot-cell:focus { box-shadow: inset 0 0 0 2px var(--focus); }
```

### 5.3 Grid interaction & painting
- Click toggles assignment for that student/day/slot; totals update
- Painting: pointerdown determines mode (assign/erase) based on starting cell or Alt; column-locked; indicator shows live hours; save and update totals on end

```745:822:student-schedule-app.html
// Pointer events: setPointerCapture, elementFromPoint hit-testing, visited Set, live indicator, save at end
```

### 5.4 Accessibility and keyboard model
- Roles: `role="grid"` on week grid, `role="row"` on slot rows, `role="gridcell"` on cells
- Roving tabindex: first cell tabbable; arrows move focus; Home/End to row ends; PageUp/Down switch days; Space/Enter toggles
```825:861:student-schedule-app.html
// Keyboard navigation and activation handlers
```

## 6) Feature Deep Dives
### 6.1 Header Buttons & Actions
- Save: writes `DataManager.state` to `localStorage` and re-renders totals
```884:888:student-schedule-app.html
window.saveSchedule = function saveSchedule() {
  DataManager.save(); UI.renderTotals(); alert('Saved current state locally. Export to persist into a file.');
};
```
- Clear Times: confirm, clear `state.schedule`, save, re-render grid and totals
```890:898:student-schedule-app.html
window.clearTimes = function clearTimes() { /* confirm; clear schedule; save; re-render */ };
```
- PDF: two-page PDF with grid screenshot + totals table
```900:1049:student-schedule-app.html
window.exportToPDF = async function exportToPDF() { /* html2canvas grid, add image; add table page with AutoTable; parity check */ };
```
- Download: export single-file HTML with embedded state
```1051:1074:student-schedule-app.html
window.exportSchedule = function exportSchedule() { /* clone document, replace embedded JSON, serialize, download */ };
```
- Add Student: open modal → save new student to state → re-render grid & totals
```1079:1093:student-schedule-app.html
window.addStudentFromModal = function addStudentFromModal() { /* build id/code/colorIndex; push; save; render */ };
```

### 6.2 Schedule Grid & Painting
- Rendering pipeline: `UI.renderWeek(students)` builds day blocks, legend chips (student codes), and 16 slot rows; attaches click, hover, focus, pointer, and keyboard handlers
```603:869:student-schedule-app.html
// renderWeek: build day headers, legend chips per student, 16 slot rows, attach events
```
- Column lock ensures painting only affects the starting student’s column; Alt always erases
```768:787:student-schedule-app.html
// applyToCell respects painting.lockStudentId and toggles class/aria accordingly
```

### 6.3 Weekly Totals Computation
- UI derives table from `DataManager.computeTotals(state)`; shows per-student days and total, then a DAILY TOTALS row
```565:601:student-schedule-app.html
// UI.renderTotals(): builds table body from computeTotals output
```

### 6.4 Student Management (Add)
- Modal open/close
```871:879:student-schedule-app.html
UI.openStudentModal / UI.closeStudentModal
```
- Creation from modal values: deterministic 1–3 letter code from initials; rotating color swatches (0–4)
```1079:1093:student-schedule-app.html
// addStudentFromModal: compute code, colorIndex, id; persist; re-render
```

### 6.5 Save/Clear (Local Persistence)
- `saveSchedule()` delegates to `DataManager.save()` (stores `appState` JSON)
- `clearTimes()` resets `state.schedule` and re-renders

### 6.6 Single-file Export (HTML)
- Clones the live DOM, replaces embedded JSON with current state, serializes, downloads `student_schedule_YYYY-MM-DD.html`
```1056:1073:student-schedule-app.html
// XMLSerializer + Blob + a.download + URL.createObjectURL
```

### 6.7 PDF Export (2 pages)
- Page 1: `html2canvas` capture of the grid at high scale with fallback; landscape Letter; title/date header
- Page 2: AutoTable of per-student totals with daily totals footer; parity check compares UI DOM totals to PDF data
```948:1037:student-schedule-app.html
// AutoTable config with head/body/foot, styles, column widths, didDrawPage header/footer
```

## 7) Performance Considerations
- Canvas memory: try scale 2, fallback to 1.5 if capture fails
- Painting: `visited` Set avoids repeat work; DOM updates are minimal; save once per gesture end
- Grid size: 5 days × 16 rows × N students; prudent use of CSS grid and class toggles keeps it responsive

## 8) Quality & Testing Approach
- Manual testing in Chrome/Edge
- PDF parity check: warns if PDF totals mismatch the UI totals table
- Alerts for major error paths (e.g., missing libraries)

```957:996:student-schedule-app.html
// Parity check block: compares UI totals to computed PDF rows
```

## 9) Extension Roadmap
- Student edit/remove (update codes and purge assignments on removal)
- Coverage/constraints (e.g., min coverage by hour or role)
- Import flow (controlled source) if reintroduced
- Theming options and print stylesheet tweaks for direct browser print

## 10) Maintenance Guide
- Versioning: `APP_VERSION` constant; embed `metadata.exportedAt`
```374:381:student-schedule-app.html
const APP_VERSION = '1.0.0';
const STORAGE_KEYS = { hybrid:'labSchedule', legacySchedule:'studentWorkerSchedule', legacyPeople:'studentWorkerPeople', appState:'studentScheduleAppState' };
```
- CSS tokens: adjust `:root` variables for theme/spacing; avoid changing grid semantics
- Data precedence: preserve load order; ensure exporters always write a fresh embedded JSON
- Accessibility: retain ARIA roles and roving tabindex; keep keyboard parity with pointer features


