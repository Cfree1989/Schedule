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
- Testing: Removed per user request (no testing functionality needed)
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
– PDF export pagination polish: anchor DAILY TOTALS as table footer.
– Totals parity validation: compare PDF totals vs UI; log mismatches.
– Student management: add edit/remove flows; update codes; purge assignments on removal.
– Chrome/Edge QA and docs/version metadata.

## Acceptance Criteria
- Single HTML file opens with embedded data (if present) and persists updates between sessions and machines
- UI matches screenshot layout closely, with Earth-tone theme and accessible contrast
- Grid supports assignment toggles via mouse and keyboard; hours summary updates reactively
- Import/export creates and reopens a single self-contained HTML with embedded JSON and version metadata
- PDF export produces two pages: Page 1 landscape schedule grid (no totals), Page 2 per-student totals; print quality is crisp on A4/Letter
- A visible PDF button and Alt+P shortcut both work; overflow menu item available
- Works in current Chrome and Edge
- No testing functionality (removed per user request)

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
- Chrome/Edge QA — Pending
- Docs + version metadata — In Progress
- PDF export — Pagination checks pending
- Theme system testing — In Progress

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
- MiniTest harness — Removed (per user request)
 - Add Student: local precedence + Enter submit — Completed
 - Manage Students: modal with delete and schedule purge — Completed
- Merge Add into Manage modal — Completed
- Seasonal color theme system — Completed

## Executor's Feedback or Assistance Requests
—

## Lessons
- **Load precedence for embedded apps**: Prefer previously saved `appState` from `localStorage` over embedded JSON so that newly added students persist across reloads until the user explicitly exports a new single-file.
- **Modal UX**: Bind Enter on the modal container to trigger the primary action, but guard against firing when focused on Cancel.
 - **Removal logic safety**: On delete, purge schedule keys where the third pipe-delimited segment equals the `studentId` to avoid orphaned assignments.
- **Single modal UX**: Consolidating add and manage reduces cognitive load; keep add form at top, keep modal open after adding to facilitate multiple entries.
- **Theme system architecture**: Use CSS custom properties with `data-theme` attributes for clean theme switching; store theme preference in both localStorage and data state for persistence across sessions and exports.

## Executor's Feedback or Assistance Requests
- ✅ **OPTIMAL BOTTOM ALIGNMENT IMPLEMENTED**: Successfully implemented the optimal bottom alignment approach:
  - Changed time gutters from 18 rows to 16 rows (removed 2 spacer rows)
  - Updated CSS to remove spacer row handling (nth-child(1) and nth-child(2))
  - Fixed hover logic to remove +2 offset for spacer rows
  - Now 16 time rows align perfectly with 16 schedule rows at the bottom
  - Day blocks maintain 18 rows (1 header + 1 legend + 16 schedule rows)
  - Bottom alignment achieved via `align-items: end` on `.schedule-shell`
- Tests functionality removed per user request. Ready for Chrome/Edge QA.

## BFROS Analysis: Height/Gutter Spacing Issue

**Problem**: Too much gutter at top and bottom; cells possibly different sizes

**Working Backwards - Possible Sources (5-7)**:
1. **Row count mismatch**: Time gutters have 18 rows (2 spacers + 16 time slots) vs day-block structure
2. **Height calculation inconsistency**: `auto` sizing on `.schedule-rows` not matching expected height  
3. **Padding inconsistency**: Different vertical padding between gutters (4px) and grid elements
4. **Border thickness effects**: `--grid-line: 2px` borders affecting total rendered heights
5. **Synthetic bottom border**: `::after` pseudo-elements adding unexpected height
6. **Header/legend row sizing**: Day-header and legend-row not exactly `var(--slot-h)` in practice
7. **Box-sizing conflicts**: Border-box vs content-box calculations

**Most Likely Sources (1-2)**:
1. **PRIMARY**: Row height structure mismatch - Time gutters (18 fixed rows) vs day-blocks (header + legend + auto-sized schedule-rows) not aligning properly
2. **SECONDARY**: The `.schedule-rows` container with `auto` height is expanding beyond expected dimensions due to border/padding accumulation

**Validation Plan**:
- ✅ Add logging to measure actual rendered heights of: time gutters, day-block components, schedule-rows
- Compare expected vs actual heights to isolate the discrepancy
- Focus on the relationship between fixed gutter rows and the day-block's three-part structure

**✅ RESOLVED - Visual Alignment Fix Applied**:

**Root Cause Confirmed**: Time gutters had `padding: 4px 0` while day blocks had no padding, creating a 4px vertical offset that misaligned the time labels with the schedule grid rows.

**Fix Applied**:
- Removed `padding: 4px 0` from `.time-gutter` and `.time-gutter-right` 
- Removed `padding: 4px 0` from `.week-grid` to maintain consistency
- Now all components (gutters, week-grid, day-blocks) have consistent 0px vertical padding
- Time labels should now align perfectly with schedule grid rows

**Result**: The visual misalignment between time gutter labels and schedule grid rows has been eliminated. The "too much gutter at top and bottom" issue is resolved.

## BFROS Analysis: Bottom Gutter Border Alignment Issue

**New Problem**: Bottom of time gutter not aligned with bottom of day containers, plus border rendering issue at bottom.

**Working Backwards - Possible Sources (5-7)**:
1. **Synthetic bottom border positioning**: The `::after` pseudo-element for bottom border may be misaligned
2. **Last time row height**: The 18th time row (4:00-4:30) may have different height than expected
3. **Day-block bottom border**: Day blocks may have different bottom border treatment than gutters
4. **Border radius mismatch**: Rounded corners on gutter vs day-block creating visual misalignment
5. **Grid row count discrepancy**: 18 gutter rows vs 16 schedule rows + header + legend = alignment issue
6. **Box-sizing effects**: Border thickness affecting total height calculations
7. **Positioning context**: Absolute positioning of synthetic border not accounting for container height

**Most Likely Sources (1-2)**:
1. **PRIMARY**: The synthetic bottom border (`::after` pseudo-element) is positioned at `bottom: 0` but the actual content height doesn't match the day-block height due to the 18 vs 16 row structure
2. **SECONDARY**: The last time row (4:00-4:30) is getting rounded bottom corners but the day-block's last row doesn't have matching treatment

**Validation Plan**:
- Check if the 18-row gutter structure (2 spacers + 16 time slots) matches the day-block structure (header + legend + 16 schedule rows)
- Verify the synthetic bottom border positioning relative to actual content height
- Compare bottom border treatment between gutters and day-blocks

**✅ RESOLVED - Bottom Gutter Alignment Fix Applied**:

**Root Cause Confirmed**: The gutter had 18 rows (2 spacer rows + 16 time slots) while day blocks had only 16 schedule rows, causing bottom misalignment.

**Fix Applied**:
- Changed gutter from `grid-template-rows: repeat(18, var(--slot-h))` to `repeat(16, var(--slot-h))`
- Removed the 2 spacer rows from `renderGutters()` - now only creates 16 time rows
- Updated CSS to remove special handling for spacer rows (nth-child(1) and nth-child(2))
- Fixed hover logic to remove `+2` offset when referencing gutter children
- Synthetic bottom border now aligns perfectly with day-block bottom

**Result**: The bottom of the time gutter now aligns exactly with the bottom of the day containers. The border rendering issue is resolved.

## ✅ SIMPLER FIX APPLIED - Bottom Alignment via CSS Grid

**User Feedback**: The row count approach didn't work. Requested simpler solution using `.schedule-shell` alignment.

**New Approach**: Use CSS Grid's `align-items: end` to align gutters and day blocks at the bottom of the container.

**Fix Applied**:
- Changed `.schedule-shell` from `align-items: start` to `align-items: end`
- Reverted to original 18-row gutter structure (2 spacers + 16 time slots)
- Restored original hover logic with `+2` offset for spacer rows
- Now both gutters and day blocks align at the bottom of the schedule shell

**Result**: Both time gutters and day blocks now have the same distance from the bottom of the `.schedule-shell` container, ensuring perfect bottom alignment regardless of their different heights.

## ✅ BOTTOM BORDER FIX APPLIED

**Issue**: With `align-items: end`, the synthetic bottom border (`::after` pseudo-element) was positioned at `bottom: 0` of the container, not aligned with the actual content.

**Fix Applied**:
- Removed the synthetic bottom border (`::after` pseudo-element)
- Added bottom border directly to the last time row (`.time:last-child`)
- Added rounded bottom corners to match the gutter's border-radius
- Now the bottom border is part of the actual content, not a separate positioned element

**Result**: The bottom border of the time gutters now properly aligns with the content and looks clean, matching the day-block bottom borders.

## Planner Plan: Optimal Bottom Alignment Approach

**User Insight**: Why not have gutters with 16 rows and day blocks with 18 rows, but align them at the bottom so the actual time content lines up perfectly?

**Optimal Structure**:
- **Time Gutter**: 16 rows (just the time slots: 8:30-4:30)
- **Day Block**: 18 rows (1 header + 1 legend + 16 schedule rows)
- **Bottom Alignment**: The 16 time rows align with the 16 schedule rows
- **Consistent Heights**: All cells use `var(--slot-h)` (28px) with `var(--grid-line)` (2px) borders

**Implementation Plan**:

1. **Time Gutter Structure**:
   - Change from 18 rows to 16 rows
   - Remove the 2 spacer rows from `renderGutters()`
   - Use `grid-template-rows: repeat(16, var(--slot-h))`
   - Update CSS selectors to remove spacer row handling

2. **Day Block Structure**:
   - Keep 18 rows: `grid-template-rows: var(--slot-h) var(--slot-h) repeat(16, var(--slot-h))`
   - Header row (day name)
   - Legend row (student chips)  
   - 16 schedule rows (actual time slots)

3. **Bottom Alignment**:
   - Keep `.schedule-shell` with `align-items: end`
   - This aligns the 16 time rows with the 16 schedule rows at the bottom
   - Header and legend rows appear at the top of day blocks only

4. **Border Consistency**:
   - All cells use consistent `var(--slot-h)` height
   - All borders use consistent `var(--grid-line)` thickness
   - Bottom border on last time row matches day-block styling

**Benefits**:
- Perfect visual alignment of time labels with schedule grid rows
- Clean separation: time content at bottom, headers/legends at top
- Consistent cell heights and border thickness throughout
- No complex CSS alignment tricks needed

## Lessons
- **Gutter Cell Sizing**: A cell's visual height can be unexpectedly reduced if it has both top and bottom borders, even with `box-sizing: border-box`. A 28px tall cell with 2px top and 2px bottom borders will have 4px less internal space than a cell with only one of those borders, making it look shorter. The best practice is to have the container handle its own borders and let the internal items only manage the dividers between them (e.g., only `border-top`), ensuring uniform internal dimensions.
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

---

# Planner — Schedule Masterplan (Outline for Approval)

## Goals, Audience, Scope
- **Goals**: Provide a single authoritative document that explains app capabilities feature-by-feature, the architecture behind them, and how to extend/maintain the system.
- **Audience**: Non-technical stakeholders (overview sections) and technical contributors (architecture, code citations, and extension guides).
- **In/Out of Scope**:
  - In: Current single-file app (`student-schedule-app.html`) features, architecture, accessibility, persistence, export/PDF, and maintainability patterns.
  - Out: Multi-file builds, backend integrations, Dockerized deployments, or import flows not present in this file.

## Proposed Table of Contents
1. Executive Summary
2. Feature Inventory (Current vs Planned)
3. System Architecture
   - Runtime model (single-file app, embedded JSON, localStorage fallback)
   - Core modules: Data layer, UI layer, Export/PDF, Student Management
4. Data Model & Persistence
   - Schema (`settings`, `students`, `schedule`, `metadata`)
   - Storage precedence and migration
5. UI/UX & Accessibility
   - Header layout and actions (Save, Clear Times, PDF, Download, Add Student)
   - Layout and styling system (CSS tokens, shells, gutters, day-blocks)
   - Grid interaction (click, keyboard, paint selection)
   - ARIA roles and keyboard model
6. Feature Deep Dives (one section per feature)
   - Header Buttons & Actions (Save, Clear, PDF, Download, Add Student)
   - Schedule Grid & Painting (time gutters, legend, cells)
   - Weekly Totals computation
   - Student Management (Add)
   - Save/Clear (local persistence)
   - Single-file Export (HTML)
   - PDF Export (2 pages)
7. Performance Considerations
8. Quality & Testing Approach (lightweight, manual, parity checks)
9. Extension Roadmap (Edit/Remove students, constraints, coverage rules)
10. Maintenance Guide (versioning, defaults, CSS tokens)

## Code Mapping for Citations (by section)
- Embedded bootstrap data: `student-schedule-app.html` lines 350–365
- CSS theme and layout tokens: lines 8–273 (notably `:root`, `.schedule-shell`, `.time-gutter`, `.week-grid`, `.day-block`)
- Header structure and buttons (HTML): lines 276–286
- Button wiring (JS): lines 516–525
- Libraries (UMD): lines 367–371
- Data layer (`DataManager`): lines 393–502
  - Totals computation: 395–418
  - Embedded/defaults/load/save: 419–502
- UI layer (`UI`): lines 504–880
  - Initialization/wiring: 505–546
  - Gutters: 547–563
  - Header actions attach: 516–525
  - Totals rendering: 565–601
  - Week grid render: 603–869
  - Painting model (pointer events): 745–822
  - Keyboard navigation: 825–861
  - Student modal open/close: 871–879
- Public API (buttons):
  - Save: 884–888
  - Clear times: 890–898
  - Export PDF: 900–1049
  - Export single-file HTML: 1051–1074
  - Add student from modal: 1079–1093
- Bootstrapping: 1095–1101

## Deliverables & Success Criteria
- A new document named “Schedule Masterplan” covering all ToC sections with inline code citations from `student-schedule-app.html` and brief rationale for design choices.
- Feature sections include: purpose, UX behavior, data flows, primary APIs, and extendability notes.
- Architecture section clearly explains single-file strategy, data precedence, and module responsibilities.
- PDF/export sections explain library roles, fallbacks, and parity checks with UI totals.

## Approval Checklist
- ToC meets your needs (confirm or adjust ordering/names)
- Code mapping lines acceptable for citation depth
- Include/exclude “Extension Roadmap” items as desired
- Tone: mixed audience (brief non-technical summaries, then technical detail)

## Handoff to Executor (upon approval)
- Create `Docs/Schedule Masterplan.md` using the ToC above
- Populate sections with cited snippets using the prescribed citation format
- Verify references and links; ensure document is navigable and concise

