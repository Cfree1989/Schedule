# Student Worker Schedule Management System — Planner Plan

## Background and Motivation
Build a portable, single-file Student Worker Schedule app that mirrors the provided screenshot’s structure and interaction model. The file must embed its data so it can be opened on any computer/browser (Chrome/Edge) and resume state. Include a high-quality PDF export with two pages: (1) schedule grid in landscape, (2) per-student weekly totals.

## Key Constraints & Decisions
- Work week/time: Monday–Friday, 8:30–16:30, 30-minute intervals (as spec)
- Roles: PA, WS, GA; no minimum coverage constraints
- Theme: Earth-tones from the spec; accessible contrast
- UI layout: Adhere closely to the screenshot’s header, grid, totals table, and footer badges
- Export/Import: Single-file HTML with embedded JSON; on load read embedded data first, else fall back to localStorage, else bootstrap defaults
- PDF Export: Two pages; Page 1 landscape schedule grid only; Page 2 per-person totals. Provide a visible header PDF button and keyboard shortcut Alt+P (also accessible via overflow menu)
- Browsers: Chrome and Edge primary; test for parity
- Testing: Include MiniTest harness (dev mode) covering DataManager, ScheduleGrid, StudentManager, File operations
- Header: "STUDENT WORKER SCHEDULE | 2024–2025" (configurable)
- Seed: Start with 5 placeholder students
- Export filename: `student_schedule_YYYY-MM-DD.html`

## Milestones (Incremental Delivery)
1) Core shell and data layer
2) Interactive grid + student management + totals
3) Import/export (single-file) + embedded data bootstrap
4) PDF export (2-page, high-res) + accessibility/keyboard nav
5) Cross-browser QA and polish

## High‑level Task Breakdown — Remaining
- MiniTest harness: validate core modules and file ops; simple in‑app runner.
- PDF export pagination polish: anchor DAILY TOTALS as table footer.
- Totals parity validation: compare PDF totals vs UI; log mismatches.
- Student management: add edit/remove flows; update codes; purge assignments on removal.
- Chrome/Edge QA and docs/version metadata.

## Acceptance Criteria
- Single HTML file opens with embedded data (if present) and persists updates between sessions and machines
- UI matches screenshot layout closely, with Earth-tone theme and accessible contrast
- Grid supports assignment toggles via mouse and keyboard; hours summary updates reactively
- Import/export creates and reopens a single self-contained HTML with embedded JSON and version metadata
- PDF export produces two pages: Page 1 landscape schedule grid (no totals), Page 2 per-student totals; print quality is crisp on A4/Letter
- A visible PDF button and Alt+P shortcut both work; overflow menu item available
- Works in current Chrome and Edge
- MiniTest reports all green for core modules

## Risks & Mitigations
- Large canvas memory when exporting PDF → use measured capture regions and a safe scale (2–3x) with fallback to 1.5x if memory caps are detected
- Embedded file size growth → compress JSON (whitespace-stripped), and optionally base64-gzip if needed (phase 1 keeps plain JSON)
- Accessibility complexity → prioritize focus order and keyboard nav for grid; defer advanced ARIA to minor update if needed
- Time complexity of totals calc → cache per-day per-student totals and recompute incrementally on change

## Planner — Paint Selection Plan (v1)
- Interaction model:
  - Primary: Mouse/touch drag to assign across contiguous cells. Action is determined by the first cell’s state (if assigned → erase mode; if unassigned → assign mode).
  - Modifier: Hold Alt to force erase while dragging regardless of first cell state.
  - Hover feedback: During drag, apply a temporary highlight class to show affected cells.
- Event handling (Pointer Events):
  - Use `pointerdown` on `schedule-cell` to start painting; call `setPointerCapture(evt.pointerId)` on the origin cell.
  - On `pointermove`, if painting is active, compute entered cell via `evt.target.closest('.schedule-cell')` and apply action if not already processed.
  - On `pointerup`/`pointercancel`, release capture and finalize.
  - Maintain a `visitedCellKeys` Set for the current gesture to avoid redundant work.
- Data updates and performance:
  - Debounce autosave to 300–500ms after last mutation during painting.
  - Batch DOM/class updates via requestAnimationFrame to avoid layout thrash.
  - Write through to `schedule` state per cell; totals recompute incrementally per affected student/day.
- Edge cases:
  - Leaving the grid while dragging: captured pointer preserves events; still finalize on `pointerup`.
  - Scrolling: tolerate vertical scroll; do not start a text selection (`user-select: none` on grid while painting).
  - Multi-touch: ignore additional pointers; only track the first active pointerId.
  - Student removal mid-gesture: guard by resolving `personCode` at start and skip if missing on move.
- Accessibility:
  - Keep existing keyboard toggle behavior unchanged. Painting is an enhancement for mouse/touch users; keyboard users still toggle per cell.
  - Announce large-range changes via an ARIA live region summary after gesture ends (e.g., "Assigned 6 slots to Jane on Tuesday").
- Tasks:
  1) Add painting state machine to grid controller (idle → painting(assign|erase) → idle).
  2) Implement pointer event handlers and pointer capture on `.schedule-cell`.
  3) Track `visitedCellKeys` per gesture; apply assign/erase consistently.
  4) Debounce persistence and incremental totals recompute.
  5) Add `user-select: none` during painting and temporary hover class.
  6) Add optional Alt modifier to force erase.
  7) QA across Chrome/Edge; verify touch on Edge/Windows.
- Acceptance criteria:
  - Dragging across cells assigns or erases contiguous ranges smoothly without lag.
  - Action is consistent based on the first cell or Alt override.
  - No text selection occurs while painting; autosave is debounced; totals update correctly.
  - Works in Chrome/Edge with mouse and touch; keyboard behavior unchanged.

## Executor — Next Actions (PDF Page 2)
1. Build table data from state
   - Derive `head = [['Student','Mon','Tue','Wed','Thu','Fri','Total']]`.
   - For each student, sum assigned slots per day (0.5h each) → two-decimal strings; compute row total.
   - Compute trailing row: `['DAILY TOTALS', monSum, ..., friSum, grand]`.
2. Generate Page 2 and render table
   - After Page 1 image, call `doc.addPage('letter','landscape')` (or `doc.addPage()` if already landscape).
   - Invoke AutoTable with configured `theme`, `styles`, `columnStyles`, and margins from the plan.
3. Add header/footer via hooks
   - Use `didDrawPage` to draw title (left), date (center), and `Page X` (right).
4. Wire into existing `exportToPDF`
   - Replace the placeholder Page 2 text with the AutoTable call; keep error handling.
5. Validate totals
   - Cross-check a few interactive toggles vs. UI totals; ensure PDF values match UI `renderTotals()`.
6. Performance fallback
   - If Page 1 html2canvas capture throws, retry with `scale: 1.5` and continue with Page 2.

## Planner — Next Steps (Post-PDF v1)
Priority order:
1) PDF polish — anchor daily totals as table footer
   - Move the "DAILY TOTALS" row into AutoTable `foot`.
   - Set `showFoot: 'lastPage'` (or `everyPage` if repetition across pages is preferred) per AutoTable docs.
   - Keep `didDrawPage` for title/date/page numbers.
   - Success: Footer row is pinned to bottom on the last page without squeezing body rows; header repeats on page breaks.
2) Totals parity validation
   - Add a quick parity check in `exportToPDF` that recomputes PDF numbers and compares to UI `renderTotals()` values; log mismatches.
   - Success: 0 mismatches for a representative set of toggles and students.
3) Accessibility & keyboard navigation
   - Roles: `role="grid"` on the week grid, `role="row"` on slot rows, `role="gridcell"` on cells; keep `aria-pressed` true/false.
   - Focus: roving tabindex (only one `tabindex="0"` cell at a time; others `-1`), visible focus outline.
   - Keys: Arrow keys move focus (Up/Down/Left/Right); Space/Enter toggles the focused cell; Home/End jump to row ends; PageUp/PageDown move days.
   - Announcements: `aria-label` on cells like "Monday 9:00–9:30 • Jane Doe"; header has live region title.
   - Success: Navigate cells by keyboard alone; toggle assignments; screen readers announce context.
4) Student management (edit/remove)
   - Add edit/remove flow to the current modal or a simple inline editor.
   - Success: Rename updates initials/code and schedule keys; remove purges assignments.
5) Cross-browser QA and docs
   - Verify Chrome/Edge; note any rendering differences; finalize README section embedded in file.
   - Success: No layout regressions; export works identically.

6) Paint selection (click-drag time assignment)
   - Mouse-down to start selection; drag over cells to assign; release to commit. Hold a modifier (e.g., Alt) to erase while dragging.
   - Success: Dragging across cells assigns/clears contiguous ranges smoothly without lag or accidental selections.
   - Impl notes: Use pointer events; on first cell determine action (assign vs erase) from initial state; during drag, apply that action to entered cells; debounce saves; add `user-select: none` while painting.

## Project Status Board
### Active
- MiniTest harness — Pending
- Chrome/Edge QA — Pending
- Docs + version metadata — In Progress
- PDF export — Pagination checks pending

### Completed
- Scaffold single-file app shell — Completed
- DataManager with storage bootstrap — Completed (v1 minimal)
- Earth-tone theme + layout — Completed
- ScheduleGrid core rendering — Completed
- StudentManager modal — Completed (add-only; edit/remove pending)
- Hours Summary table — Completed
- Single-file export/import with embedded JSON — Completed
- PDF export (2 pages, button + Alt+P) — Completed (v1); footer anchored
- A11y + keyboard nav — Completed (baseline implemented)
- Paint selection (column-locked) — Completed

## Executor's Feedback or Assistance Requests
- No current blockers or assistance requests.

## Lessons
- **PDF Export**: CDN UMD builds (jsPDF, html2canvas, jsPDF-AutoTable) avoid bundling complexity while enabling two-page export.
- **Accessibility**: Roving tabindex pattern with programmatic focus via arrows provides clean keyboard navigation without excessive tab stops.

## Decision History
- PDF control revised: include a visible header button and Alt+P (overflow menu also)
- Embedded JSON is the primary data source; localStorage is a fallback

## Completed Features Archive
- Single-file app shell — Implemented header, grid container, totals table, modal, and footer badges with Earth-tone theme and accessible contrast.
- DataManager — Schema with metadata, students, schedule, settings; autosave and storage strategy (embedded JSON → localStorage → defaults); migration hook stubbed.
- Earth-tone theme & layout — Responsive grid, sticky time gutters, accessible color palette, close adherence to screenshot layout.
- ScheduleGrid core — Mon–Fri, 8:30–16:30 at 30-minute intervals; color-coded assignments; click/keyboard toggle; dynamic columns per number of students.
- StudentManager (add-only) — Modal add flow; deterministic initials/code and color assignment; persistence integrated; edit/remove deferred.
- Hours Summary table — Per-student daily and weekly totals; draggable rows; trailing DAILY TOTALS across all students.
- Single-file export/import — Export creates self-contained HTML with embedded JSON and version metadata; import validates and merges/replaces state.
- PDF export v1 — Two pages (Page 1 landscape grid, Page 2 per-student totals); header button and Alt+P; footer anchored; pagination checks pending.
- Accessibility baseline — Roving tabindex and ARIA roles (`grid`, `row`, `gridcell`); keyboard navigation (arrows, Home/End, PageUp/PageDown, Space/Enter).
- Paint selection — Drag to assign/erase contiguous cells; pointer events with capture and elementFromPoint hit-test; Alt forces erase; column locked to starting student to prevent accidental cross-student painting; save at gesture end.

## Quick Reference (for Executor)
- Work week/time: Mon–Fri, 8:30–16:30, 30-min slots
- Roles: PA, WS, GA (no minimum coverage)
- PDF: Page 1 = schedule grid landscape; Page 2 = per-student totals
- Access PDF: Alt+P or header button; overflow menu option available

## Architecture Reference
- **Layout**: 3-column CSS grid (time gutters + week grid); Mon–Fri, 8:30–16:30, 30-min slots
- **Data Model**: Sparse map `${day}-${slot}-${personCode}` → `true`; 0.5h per slot
- **Storage**: Embedded JSON → localStorage → defaults; autosave via MutationObserver
- **Libraries**: jsPDF, html2canvas, jsPDF-AutoTable via CDN UMD
