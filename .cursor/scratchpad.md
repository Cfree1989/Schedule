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

## High-level Task Breakdown (Implementation Outline)
1. Scaffold new single-file app `student-schedule-app.html` (HTML/CSS/JS inline). Header, grid container, totals table, modal overlay, footer badges. Placeholders for John/Jane Doe.
2. Implement `DataManager` with schema (metadata, students, schedule, settings), reactive updates, autosave, and storage strategy: embedded JSON → localStorage → defaults. Add migration hook.
3. Implement Earth-tone theme CSS (variables), responsive grid, and accessible color contrast.
4. Implement `ScheduleGrid` rendering Mon–Fri and 8:30–16:30 at 30-min intervals. Color-coded assignments, click/keyboard toggle, drag or quick assign.
5. Implement `StudentManager` (modal form) to add/edit students, generate initials/code, assign deterministic earth-tone color.
6. Compute and render Hours Summary: daily totals per student and weekly totals; daily totals row.
7. Implement `FileManager` single-file export/import: clone current HTML, embed JSON in `<script id="embedded-data" type="application/json">` and version metadata; import merges/replaces with validation.
8. Implement PDF export using html2canvas + jsPDF: Page 1 landscape schedule grid only; Page 2 per-student totals. High resolution (scaled canvas). Expose via header button and Alt+P (and overflow menu).
9. Add keyboard navigation (grid arrows, Enter/Space toggle), ARIA roles/labels; focus management.
10. Add MiniTest harness to validate core modules and file ops (dev only), plus a simple in-app test runner panel.
11. Cross-browser checks (Chrome/Edge), fix layout differences;
12. Embed version metadata and lightweight usage docs within the file.

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

## Project Status Board (initial)
- Scaffold single-file app shell — Completed
- DataManager with storage bootstrap — In Progress
- Earth-tone theme + layout — Pending
- ScheduleGrid core rendering — Pending
- StudentManager modal — Pending
- Hours Summary table — Pending
- Single-file export/import with embedded JSON — Pending
- PDF export (2 pages, button + Alt+P) — Pending
- A11y + keyboard nav — Pending
- MiniTest harness — Pending
- Chrome/Edge QA — Pending
- Docs + version metadata — Pending

## Executor's Feedback or Assistance Requests
- None at this time. Next up: flesh out ScheduleGrid interactions and totals.

## Lessons
- Context7: Using `/parallax/jspdf`, `/niklasvh/html2canvas`, and `/simonbengtsson/jspdf-autotable` CDN UMD builds aligns with our two-page PDF export goal and avoids bundling complexity.

## Decision History
- PDF control revised: include a visible header button and Alt+P (overflow menu also)
- Embedded JSON is the primary data source; localStorage is a fallback

## Quick Reference (for Executor)
- Work week/time: Mon–Fri, 8:30–16:30, 30-min slots
- Roles: PA, WS, GA (no minimum coverage)
- PDF: Page 1 = schedule grid landscape; Page 2 = per-student totals
- Access PDF: Alt+P or header button; overflow menu option available

## Planner Analysis — Template Structure (Student Worker Schedule Template.html)
- **Header**: `STUDENT WORKER SCHEDULE | 2024-2025` top banner.
- **Buttons (Admin Controls)**:
  - **Quick Save**: `saveSchedule()` → `saveHybridSchedule()` + persist `studentWorkerPeople`.
  - **Clear Times**: `clearTimes()` resets all assignments (keeps students).
  - **PDF**: `exportToPDF()` uses html2canvas + jsPDF to generate 2 pages: Page 1 (header + schedule grid), Page 2 (header + totals). Scale 3.0, Letter landscape.
  - **Download**: `exportSchedule()` creates a self-contained HTML, embedding current `schedule` (into `sampleSchedule`) and `people` inline.
  - **Add Student**: opens modal → `addStudentFromModal()` generates initials, assigns earth-tone color, persists list, re-renders.
  - Hidden import input: `#scheduleFileInput` with `handleScheduleFileSelect(...)` parses embedded `sampleSchedule` and `people` from an HTML file. Helper `importSchedule()` triggers the chooser (no visible button currently).

- **Schedule Grid and Time Slot Area**:
  - Layout is a 3-column CSS grid: left time gutter, central week grid, right time gutter.
  - **Time gutters (left and right)**: 12-hour labels; top two empty rows to align with day header and legend; then 16 rows from 8:30–9:00 through 16:00–16:30. Gutters are sticky to edges.
  - **Week grid**: 5 `day-block`s (Monday–Friday). Each `day-block` has:
    - `day-header` (day name),
    - `legend-row` (top row of student initials per day),
    - 16 `schedule-row`s (one per 30-min slot).
  - Column count per day is dynamic: `grid-template-columns: repeat(numStudents, 1fr)`.
  - **Cells**: Each `schedule-cell` has `data-day`, `data-slot`, `data-person`. Click toggles assignment; applies the student color class (earth-tone). Key format: `${day}-${slot}-${personCode}`.

- **Legend (Initials Row)**:
  - One legend cell per student at the top of each day (`legend-row`). Displays `person.code` (initials); background via per-student color class.

- **Totals Table (separate, below grid)**:
  - Section `totals-section` → table `#totalsTable` built by `renderTotalsTable()`.
  - For each student: Monday–Friday daily totals computed by counting scheduled slots × 0.5h; weekly total is the row sum.
  - Rows are draggable to reorder students; name cell opens `editStudent(...)` (rename, role change, or removal). Grand-total row labeled `DAILY TOTALS` shows per-day sums across all students and a weekly grand total.

- **Hour Calculation Details**:
  - `timeSlots` = 16 half-hour ranges; each scheduled slot contributes 0.5h.
  - `schedule` is a sparse map of `${day}-${slot}-${personCode}` → `true`.
  - Editing student code updates all schedule keys to maintain assignments.

- **Persistence & State (Hybrid)**:
  - Autosave via `MutationObserver` on class attribute changes within `#weekGrid` (debounced 500ms).
  - Hybrid structure `hybridData` keeps `trackedState` (schedule), `domVerification` snapshot, and `metadata` (checksum, timestamp).
  - Storage keys: `labSchedule` (hybrid), `studentWorkerSchedule` (legacy schedule), `studentWorkerPeople` (students).
  - Load order: embedded `sampleSchedule`/`people` (if present) → `labSchedule`/legacy → defaults.

- **Libraries (Context7 references)**:
  - jsPDF: `/parallax/jspdf` (CDN v2.5.1 UMD) — PDF generation.
  - jsPDF-AutoTable: `/simonbengtsson/jspdf-autotable` (CDN 3.5.29) — loaded, not currently used in `exportToPDF()`.
  - html2canvas: `/niklasvh/html2canvas` (CDN v1.4.1) — DOM rasterization for PDF images.

- **Accessibility Note**:
  - No explicit ARIA roles/keyboard navigation on the grid yet; planned under A11y milestone.
