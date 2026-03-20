# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.11.1] - 2026-03-20

### Fixed
- **CRITICAL: Gantt Chart Row Height Alignment**: Fixed misalignment between task column rows and timeline rows
  - **Root Cause**: Task column row height was updated to 70px in v2.11.0, but timeline row height remained at 60px
  - **Symptom**: Border lines between task names and task bars became increasingly misaligned as users scrolled down
  - **Solution**: Updated `.timeline-row` height from 60px to 70px (task-manager.html:3656)
  - **Result**: Border lines now perfectly align between task column and timeline, maintaining alignment regardless of scroll position

### Technical Details
- **CSS Change** (task-manager.html:3656):
  ```css
  /* Before */
  .timeline-row { height: 60px; }

  /* After */
  .timeline-row { height: 70px; }
  ```
- **Synchronization**: Both `.task-row` (task column) and `.timeline-row` (timeline) now use identical 70px height
- **Grid Alignment**: CSS Grid automatically maintains row alignment when both columns have matching heights

### Visual Impact
```
Before (Misaligned):
┌─────────────┬─────────────┐
│ Task 1 (70) │ Bar 1 (60)  │
├─────────────┤ Gap         │ ← Misalignment starts
│ Task 2 (70) ├─────────────┤
│             │ Bar 2 (60)  │
├─────────────┤ Gap grows   │ ← Gets worse
│ Task 3 (70) ├─────────────┤

After (Aligned):
┌─────────────┬─────────────┐
│ Task 1 (70) │ Bar 1 (70)  │
├─────────────┼─────────────┤ ← Perfect alignment
│ Task 2 (70) │ Bar 2 (70)  │
├─────────────┼─────────────┤ ← Perfect alignment
│ Task 3 (70) │ Bar 3 (70)  │
```

## [2.11.0] - 2026-03-20

### Added
- **Gantt Chart: Resizable Task Column**: Task column width can now be manually adjusted by dragging
  - Excel-like column resizer on the right edge of task column
  - Drag to resize between 150px and 500px
  - Width preference saved to localStorage and persists across sessions
  - Visual feedback: Blue highlight on hover, stronger blue during drag
  - Smooth cursor change to `col-resize` during interaction

### Fixed
- **Gantt Chart: Task Row Text Clipping**: Fixed task name text being cut off at bottom
  - Increased `.task-row` height from 60px to 70px (task-manager.html:3575)
  - Reduced vertical padding from 0.75rem to 0.5rem (task-manager.html:3572)
  - Provides adequate space for both WBS number and task name display

### Technical Implementation
- **CSS Variables** (task-manager.html:3441-3443):
  - Added `:root { --task-column-width: 250px; }` for dynamic width control
  - Changed `grid-template-columns: 250px max-content` to `var(--task-column-width) max-content` (line 3548)

- **Resizer UI** (task-manager.html:3561-3579):
  - `.column-resizer`: 8px wide absolute-positioned element on right edge
  - Transparent background, blue highlight on hover, stronger blue during drag
  - Z-index: 25 (above task content, below headers)

- **JavaScript Resizer Logic** (task-manager.html:4019-4087):
  - localStorage restoration on page load (lines 4024-4030)
  - Mouse event handlers for drag interaction (lines 4037-4084)
  - Width constraints: 150px ≤ width ≤ 500px
  - Real-time CSS variable update during drag
  - Persistent storage on mouseup

### UI/UX Improvements
- **Better Text Readability**: Task names and WBS numbers fully visible without clipping
- **Flexible Layout**: Users can adjust task column width based on task name length
- **Visual Feedback**: Clear indication of draggable area with hover and drag states
- **Persistent Preferences**: Column width remembered across browser sessions

## [2.10.1] - 2026-03-20

### Fixed
- **CRITICAL: Assignee data export/import**: Assignee configurations (custom IDs like DEV1, DEV2) are now properly included in JSON exports and restored on import
  - Fixed `saveToFile()` and `exportJSON()` to sync assignee data from localStorage to projectData before saving
  - Fixed `loadFromFile()` to restore assignee data from imported JSON to localStorage
  - Resolved data loss issue when transferring projects between browsers/PCs

### Changed
- **Export Dialog** (task-manager.html:1302, 1307):
  - JSON description: Added "(タスク・担当者等のシステムデータ含む)" to clarify system data inclusion
  - CSV description: Added "(担当者等のシステムデータは含まれません)" to clarify CSV limitations
- **Import Dialog** (task-manager.html:1343, 1348):
  - JSON description: Added "(タスク・担当者等のシステムデータを含む完全データ)" for clarity
  - CSV description: Added "(担当者等のシステムデータは含まれません・既存データは置き換えられます)"
- **Settings Modal Hint** (task-manager.html:1415):
  - Changed from "データはローカルストレージに保存されます"
  - To "データはJSON形式エクスポートに含まれます（CSV形式には含まれません）"

### Technical Details
- Added assignee sync logic in `saveToFile()` (lines 2257-2261)
- Added assignee sync logic in `exportJSON()` (lines 2387-2391)
- Assignee restore logic in `loadFromFile()` already present (lines 2337-2342)
- JSON structure now includes `"assignees": ["PM", "ENG", ...]` field
- Backward compatible: Old JSON files without assignees field continue to work

## [2.10.0] - 2026-03-20

### Fixed
- **CRITICAL: Complete Gantt Chart Scrolling Architecture Overhaul**: Resolved all three critical issues simultaneously
  - **Issue 1**: Task column now properly fixed during horizontal scroll (Excel "Freeze Panes" behavior)
  - **Issue 2**: Date headers now visible during vertical scroll AND follow horizontal scroll correctly
  - **Issue 3**: Horizontal scrollbar now appears properly when timeline exceeds viewport width

### Root Cause Analysis (External Expert Consultation)
- Consulted three external AI experts (Opus, GPT, Gemini) who identified two fundamental architectural flaws:
  1. **CSS Grid `auto` keyword constraint**: `grid-template-columns: 250px auto` caused timeline column to be clamped to parent's available width (~950px), preventing overflow and scrollbar
  2. **Split scroll container architecture**: Separate `.gantt-container` (vertical) and `.gantt-scroll` (horizontal) wrappers prevented `position: sticky` from working on both axes simultaneously

### Changed
- **HTML Structure** (line 3758-3761):
  - **REMOVED**: `.gantt-scroll` wrapper div
  - **SIMPLIFIED**: Single `.gantt-container` → `.gantt-table` hierarchy
  - Eliminates scroll context fragmentation

- **CSS: Scroll Container** (`.gantt-container`, lines 3507-3515):
  - Changed from `overflow-y: auto` to `overflow: auto` (handles both axes)
  - Added `position: relative` for sticky positioning context
  - Single unified scroll container for all scrolling behavior

- **CSS: Grid Layout** (`.gantt-table`, lines 3517-3521):
  - Changed from `grid-template-columns: 250px auto` to `250px max-content`
  - Changed from `min-width: 800px` to `min-width: max-content`
  - Timeline column now expands to full calculated width (e.g., 3000px)
  - Grid no longer constrains timeline to parent's remaining width

- **CSS: Task List Column** (`.task-list`, lines 3523-3529):
  - **ADDED**: `position: sticky; left: 0; z-index: 20`
  - Entire column now sticky to left edge (not individual rows)
  - Simplifies architecture and improves performance

- **CSS: Task List Header** (`.task-list-header`, lines 3531-3544):
  - Changed `z-index` from 20 to 30 (highest - top-left corner position)
  - Maintains dual-sticky position: `left: 0; top: 0`

- **CSS: Task Rows** (`.task-row`, lines 3546-3555):
  - **REMOVED**: `position: sticky; left: 0; z-index: 10`
  - Individual rows no longer need sticky (parent `.task-list` handles it)
  - Cleaner CSS architecture

### Removed
- **CSS Rule**: `.gantt-scroll` class entirely deleted (lines 3516-3519 removed)
- **Redundant Sticky Positioning**: Individual task row sticky positioning (handled by column-level sticky)

### Technical Implementation
- **Grid Track Sizing**: `max-content` allows grid column to respect child element's explicit width (3000px)
- **Single Scroll Context**: Both axes scroll within same container, enabling dual-axis sticky positioning
- **Z-Index Hierarchy**: 30 (corner) > 20 (task column) > 15 (timeline header) > 10 (content)
- **Column-Level Sticky**: More efficient than per-row sticky positioning

### JavaScript
- **No Changes Required**: Existing width calculation logic remains unchanged
- `.timeline` element still receives explicit width: `style="width: ${totalTimelineWidth}px;"`
- CSS `max-content` automatically calculates `.gantt-table` width from children

### Documentation
- Generated 7 comprehensive documentation files for external AI expert consultation:
  - `00_PROMPT_FOR_AI_EXPERT.md` - Structured question format
  - `01_CURRENT_ISSUE_SUMMARY.md` - Three critical issues with user feedback history
  - `02_HTML_STRUCTURE.md` - Complete HTML hierarchy and generation code
  - `03_CSS_RULES.md` - All relevant CSS rules with annotations
  - `04_PROBLEM_ANALYSIS.md` - Root cause analysis of `auto` vs `max-content`
  - `05_PROPOSED_SOLUTIONS.md` - Three solution approaches with comparison matrix
  - `06_EXPECTED_BEHAVIOR.md` - Excel-like "Freeze Panes" specification with ASCII diagrams

### UI/UX Improvements
- **Excel-Like Freeze Panes**: Task names always visible during horizontal scroll
- **Proper Header Behavior**: Date headers scroll horizontally with timeline, stay fixed during vertical scroll
- **Natural Scrolling**: Horizontal scrollbar appears and functions correctly for wide timelines
- **Performance**: Column-level sticky positioning more efficient than per-row sticky
- **Architectural Simplicity**: Single scroll container simplifies event handling and debugging

### Breaking Changes
- None - Purely internal architectural improvements

### References
- External expert feedback files in: `/mnt/c/dev/ppt-transfer/docs/session/archive/11_ganttchart_ui/feedback/`
  - `01_opus.md` - Recommended Flexbox approach
  - `02_gpt_Gantt Chart UI 技術レビュー（批判的・建設的見解）.md` - Agreed on root causes
  - `03_gemini_Gantt Chart Architecture Critique & Implementation Guide.md` - **Winning recommendation**: Keep Grid, use `max-content`, single scroll container

## [2.9.0] - 2026-03-20

### Fixed
- **CRITICAL: Horizontal Scrollbar Display**: Fixed timeline horizontal scrollbar not appearing
  - Root cause: `.timeline` element (grid column 2) was set to `auto`, causing it to shrink to parent's remaining width (~950px)
  - Child elements had explicit widths (e.g., 3000px), but parent element was still shrinking
  - Solution: Set explicit width on `.timeline` element itself: `style="min-width: ${totalTimelineWidth}px; width: ${totalTimelineWidth}px;"`
  - Timeline now properly expands beyond viewport width, triggering horizontal scrollbar

### Removed
- **Task Bar Text Display**: Removed all text display inside task bars on timeline
  - Deleted `displayText` calculation logic (lines 3710-3721)
    - Previously showed task name for bars >= 15% width
    - Previously showed WBS number for bars >= 5% width
  - Removed `${displayText}` from task bar HTML (line 3753)
  - Task bars now display as color-only rectangles
  - Task information still available via tooltip on hover
  - Eliminates visual issues like "業務要件ヒアリング・整理" and "1.2" appearing on timeline

### Technical
- **Width Calculation Order**: Moved `totalTimelineWidth` calculation before `.timeline` element creation (line 3688)
- **Explicit Width Setting**: `.timeline` element now has inline style with calculated width
- **Grid Column Behavior**: Grid's `auto` column now forced to expand by explicit child width

### UI/UX
- **Cleaner Timeline**: No text overlap or visual clutter on task bars
- **Proper Scrolling**: Horizontal scrollbar appears when timeline width exceeds viewport
- **Color-Only Bars**: Task status indicated purely by color (green/blue/gray/orange/red)
- **Tooltip Information**: All task details available on hover

## [2.8.9] - 2026-03-20

### Removed
- **Task Name Labels**: Removed task name labels from timeline area
  - Deleted `.task-name-label` CSS class and `.align-right` variant (lines 3544-3564 removed)
  - Removed JavaScript generation of task name labels (lines 3736-3743)
  - Labels were difficult to implement properly and caused visual clutter
  - Task information still available via tooltip on hover

### Changed
- **Gantt Chart Header Icon**: Replaced "ガントチャート" text with SVG icon (line 3594)
  - Changed from `<h1>ガントチャート</h1>` to gantt chart SVG icon
  - Icon size: 32x32px
  - Cleaner, more compact header design

- **Refresh Button Icon**: Replaced "更新" text with SVG refresh icon (line 3606)
  - Changed from `<button>更新</button>` to icon-only button with tooltip
  - Icon size: 20x20px
  - More consistent with modern UI patterns

### Fixed
- **Horizontal Scrollbar Display**: Fixed timeline horizontal scrollbar not appearing
  - Added explicit width calculation: `totalTimelineWidth = cells.length * cellWidth` (line 3694)
  - Set timeline-grid width: `min-width: ${totalTimelineWidth}px; width: ${totalTimelineWidth}px` (line 3697)
  - Set timeline-cell width: `width: ${cellWidth}px; flex: none` (line 3699)
  - Set timeline-bg width: `width: ${totalTimelineWidth}px` (line 3729)
  - Set timeline-bg-cell width: `width: ${cellWidth}px; flex: none` (line 3735)
  - Removed `right: 0` from `.timeline-bg` CSS (line 3493)
  - Removed `flex: 1` from `.timeline-cell` and `.timeline-bg-cell` CSS
  - Timeline now properly expands beyond viewport width, triggering horizontal scroll

- **Timeline Header Horizontal Scroll**: Confirmed timeline date headers scroll with timeline content
  - Timeline header has `position: sticky; top: 0` (sticks vertically, scrolls horizontally)
  - Task list header has `position: sticky; left: 0; top: 0` (fixed to top-left corner)
  - Proper synchronization between headers and scrolling content

### Technical
- **Width Calculation**: Explicit width calculation prevents flexbox from shrinking timeline
- **CSS Cleanup**: Removed flex properties that interfered with horizontal scrolling
- **Icon Integration**: SVG icons embedded directly in HTML for better performance

### UI/UX
- **Cleaner Header**: Icon-only design reduces visual clutter
- **Proper Scrolling**: Horizontal scrollbar appears when timeline is wide
- **Simplified Timeline**: Removed overlapping labels for cleaner appearance
- **Consistent Behavior**: Timeline scrolls naturally with date headers following

## [2.8.8] - 2026-03-20

### Fixed
- **CRITICAL: Timeline Header Horizontal Scroll**: Fixed timeline date headers to scroll horizontally with timeline content
  - v2.8.7 incorrectly made date headers completely fixed, breaking fundamental gantt chart functionality
  - Reverted HTML structure back to single `.gantt-table` container (from separated header/body)
  - Timeline header now `position: sticky; top: 0` ONLY (no left) - sticks vertically but scrolls horizontally
  - Task list header now `position: sticky; left: 0; top: 0; z-index: 20` - fixed to top-left corner
  - Task rows `position: sticky; left: 0; z-index: 10` - fixed to left, scrolls vertically
  - Timeline header placed back INSIDE `.timeline` div so it scrolls with timeline content
  - Date headers now properly synchronized with timeline horizontal scrolling

### Changed
- **HTML Structure**: Reverted to simpler grid-based layout (lines 3653-3659)
  - Removed `.gantt-header-row` and `.gantt-body` containers from v2.8.7
  - Restored single `.gantt-table` container with grid layout
  - Timeline header moved back inside timeline column for horizontal scroll sync

- **CSS Structure**: Fixed sticky positioning for proper scrolling behavior
  - `.gantt-container` (lines 3383-3390): Removed flexbox, restored `overflow-y: auto`
  - `.gantt-scroll` (lines 3392-3395): `overflow-x: auto; overflow-y: visible`
  - `.gantt-table` (lines 3397-3401): Restored grid layout `250px auto`
  - `.task-list-header` (lines 3408-3421): `position: sticky; left: 0; top: 0; z-index: 20`
  - `.task-row` (lines 3423-3435): `position: sticky; left: 0; z-index: 10`
  - `.timeline-header` (lines 3459-3467): `position: sticky; top: 0; z-index: 15` (NO left property)

- **JavaScript**: Reverted to single-container population
  - `renderGantt()` (lines 3668-3779): Populates single `ganttTable` container
  - Timeline header generated inside timeline column (lines 3703-3708)
  - Fixed `ganttTable.innerHTML` assignment (line 3779)

- **Media Queries**: Updated for restored structure
  - Changed `.gantt-header-row, .gantt-body` back to `.gantt-table` (lines 3588, 3600)

### Technical
- **Z-index Layering**: Proper stacking for overlapping sticky elements
  - Task list header: z-index 20 (highest - top-left corner)
  - Timeline header: z-index 15 (sticks to top, scrolls horizontally)
  - Task rows: z-index 10 (sticks to left, scrolls vertically)
  - Task name labels: z-index 7 (above task bars)
  - Task bars: z-index 5

- **Sticky Positioning Strategy**:
  - Task list header: Both `left: 0` and `top: 0` keeps it in top-left corner
  - Timeline header: Only `top: 0` allows horizontal scroll while sticking to top
  - Grid layout ensures proper alignment between sticky header and scrolling content

### UI/UX
- **Restored Gantt Chart Functionality**: Date headers now correctly move with timeline
- **Improved Navigation**: Task list header always visible in top-left corner
- **Proper Synchronization**: Timeline dates scroll horizontally with task bars
- **Maintains Vertical Sticky**: Both headers stick to top during vertical scroll
- **All Task Labels Display**: Task name labels appear above all task bars (not just one)

## [2.8.7] - 2026-03-20

### Fixed
- **Gantt Chart Sticky Headers**: Fixed task/date headers to actually stick during scroll
  - Restructured HTML: Separated header row from body content
  - Header row now independent with `position: sticky; top: 0; z-index: 100`
  - Headers remain visible when scrolling vertically through tasks
  - Removed ineffective sticky from `.task-list-header` and `.timeline-header` (they're now children of sticky parent)

- **Horizontal Scrollbar**: Confirmed grid-template-columns uses `auto` (was already correct from v2.8.4)
  - Timeline expands to full content width
  - Horizontal scrollbar appears when needed

- **Task Name Labels**: Confirmed labels are generated and positioned correctly (was already correct from v2.8.4/v2.8.5)
  - Labels display above task bars: `WBS_No: Task_Name`
  - Smart right-edge alignment when bar position > 80%
  - Labels overlay task bars with z-index: 7

### Changed
- **HTML Structure**: Separated header and body for sticky functionality
  - Added `.gantt-header-row` container (lines 3650-3652)
  - Added `.gantt-body` container (lines 3654-3656)
  - Removed old `.gantt-table` container

- **CSS Structure**: New layout for sticky headers
  - `.gantt-container` (lines 3383-3391): Changed to flexbox column, added height: calc(100vh - 60px)
  - `.gantt-header-row` (lines 3393-3401): New sticky header with grid layout
  - `.gantt-scroll` (lines 3403-3407): Overflow auto with flex: 1
  - `.gantt-body` (lines 3409-3413): Grid layout for body content
  - `.task-list-header` (lines 3420-3428): Removed sticky properties (parent handles it)
  - `.timeline-header` (lines 3466-3471): Removed sticky properties (parent handles it)

- **JavaScript**: Split rendering logic for header and body
  - `renderGantt()` (lines 3670-3789): Now populates two separate containers
  - Header content: `.gantt-header-row` (task list header + timeline header)
  - Body content: `.gantt-body` (task rows + timeline rows)

- **Media Queries**: Updated for new structure
  - Changed `.gantt-table` references to `.gantt-header-row, .gantt-body` (lines 3592, 3604)

### Technical
- Header separation enables proper sticky behavior (sticky doesn't work inside overflow containers)
- Grid layout maintained for both header and body with matching column widths
- Vertical scrolling now works correctly with sticky headers
- Horizontal scrolling works independently for header and body (synchronized by grid columns)

### UI/UX
- Task and date headers always visible during vertical scroll
- Horizontal scrollbar appears when timeline is wide
- Task name labels visible above task bars
- Seamless navigation with sticky headers
- More intuitive timeline reference while scrolling

## [2.8.6] - 2026-03-20

### Fixed
- **Gantt Chart Header Layout**: Minimized and unified header into single row
  - Combined title, view mode buttons, refresh button, and status legend into one row
  - Reduced header padding: 1rem 1.5rem → 0.5rem 1rem
  - Reduced title font size: 1.5rem → 1.2rem
  - Removed emoji from title
  - Removed status title ("ステータス")
  - Status legend now inline on the right side of header

- **Task/Date Headers Sticky**: Made task and date headers always visible during scroll
  - Added `top: 0` to `.task-list-header` (z-index: 50)
  - Confirmed `top: 0` on `.timeline-header` (z-index: 49)
  - Headers remain visible when scrolling vertically
  - Date labels always visible for timeline reference

### Changed
- **Header Layout**: Single-row compact design
  - `.header` (lines 3324-3334): Reduced padding, changed to flex with gap
  - Title, buttons, and legend all in one row with `gap: 1rem`
  - Legend positioned with `margin-left: auto` to push to right

- **Status Legend**: Inline horizontal layout
  - `.legend` (lines 3565-3570): Changed to inline flex layout
  - `.legend-item` (lines 3572-3578): Reduced font size to 0.8rem, gap to 0.3rem
  - `.legend-color` (lines 3580-3584): Reduced size 20px → 12px
  - Removed `.legend-title` completely

- **Button Styles**: Minimized for compact layout
  - `.view-mode-btn` (lines 3349-3357): Reduced padding 0.5rem 1rem → 0.35rem 0.75rem
  - `.refresh-btn` (lines 3369-3377): Reduced padding and font size to 0.8rem
  - Border radius reduced: 4px → 3px

- **Gantt Container**: Removed extra spacing
  - `.gantt-container` (lines 3383-3389): margin: 1rem → 0, removed border-radius and box-shadow
  - Full-width layout without margins

### Technical
- Modified generateGanttHTML() function (lines 3261-3900):
  - Updated header HTML structure (lines 3615-3647)
  - Combined header, controls, and legend into single div
  - Removed separate legend div
  - Modified CSS for compact layout

### UI/UX
- Much more compact gantt chart header (single row vs multi-row)
- More screen space for actual gantt chart content
- Task and date headers always visible during vertical scroll
- Cleaner, more professional appearance
- Legend always visible without taking extra vertical space
- Easier timeline navigation with sticky date headers

## [2.8.5] - 2026-03-20

### Fixed
- **Task Name Label Positioning**: Fixed task name labels to overlay task bars instead of creating separate space
  - Labels now positioned directly above task bars (top: -18px, was -22px)
  - Removed extra space: timeline rows reduced from 85px to 60px
  - Removed padding-top: 25px from .timeline-row
  - Task list rows also reduced from 85px to 60px for alignment
  - Labels now properly overlay the task bars instead of occupying independent space

### Changed
- **Timeline Row Height**: Reduced from 85px to 60px
  - `.timeline-row` (lines 3483-3488): height: 85px → 60px, removed padding-top: 25px
  - `.task-row` (line 3426): height: 85px → 60px

- **Task Name Label Position**: Adjusted to overlay task bars
  - `.task-name-label` (lines 3541-3556): top: -22px → -18px, z-index: 6 → 7
  - Labels render on top of task bars instead of in separate space
  - More compact and cleaner visual appearance

### Technical
- Modified CSS: .timeline-row, .task-row, .task-name-label
- Label positioning now uses z-index: 7 to ensure proper layering above task bars (z-index: 5)
- Labels overlay task bars without requiring extra vertical space

### UI/UX
- Task name labels now overlay task bars instead of occupying separate space
- More compact timeline view (60px rows vs 85px)
- Labels remain visible and readable with semi-transparent background
- Cleaner visual appearance without extra whitespace
- Maintains all functionality from v2.8.4 (smart positioning, right-edge handling)

## [2.8.4] - 2026-03-20

### Fixed
- **Gantt Chart Horizontal Scrollbar**: Fixed missing horizontal scrollbar in gantt chart
  - Changed `.gantt-table` grid-template-columns from `250px 1fr` to `250px auto`
  - Timeline now expands to full content width
  - Horizontal scrollbar appears when timeline exceeds viewport width
  - Applied to all responsive breakpoints (1200px, 800px)

### Added
- **Task Name Labels on Timeline**: Added task name labels above task bars in gantt chart
  - Labels show format: `WBS_No: Task_Name` (e.g., "WBS1.2.3: Login Implementation")
  - Smart positioning: left-aligned by default, right-aligned when near right edge (>80%)
  - Semi-transparent white background with subtle border for readability
  - Labels can extend beyond task bar width
  - Non-interactive (pointer-events: none) to avoid blocking clicks

### Changed
- **Timeline Row Height**: Increased from 60px to 85px to accommodate task labels
  - Added 25px top padding for label space
  - Task list rows also increased to 85px for alignment
  - Maintains perfect vertical alignment between task list and timeline

- **CSS Modifications**:
  - `.gantt-table` (lines 3394-3398): Changed grid-template-columns to `250px auto`
  - `.task-row` (line 3426): Changed height from `60px` to `85px`
  - `.timeline-row` (lines 3483-3489): Changed height to `85px`, added `padding-top: 25px`
  - Added `.task-name-label` (lines 3541-3561): New label styling with positioning logic
  - Applied grid changes to media queries (lines 3576, 3588)

### Technical
- Modified gantt HTML generation (lines 3772-3786):
  - Added right-edge detection: `(startOffset + duration) > 80`
  - Added conditional alignment class: `align-right` for labels near right edge
  - Labels positioned at `top: -22px` above task bars
  - Labels render before task bars in DOM for proper z-index layering

### UI/UX
- Horizontal scrollbar now visible when timeline content is wide
- Task names always visible above bars, not just on hover
- Labels intelligently positioned to avoid right-edge cutoff
- Improved readability with semi-transparent background
- Increased row height provides better visual breathing room
- Seamless scrolling experience across wide timelines

## [2.8.3] - 2026-03-20

### Fixed
- **Gantt Chart Header Alignment**: Fixed header height mismatch between task list and timeline
  - Unified `.task-list-header` to 50px with horizontal padding only
  - Unified `.timeline-header` to 50px fixed height
  - Unified `.timeline-cell` to 50px with flexbox centering
  - Headers now perfectly align vertically

### Added
- **Comprehensive Tooltips**: Enhanced hover information for gantt chart elements
  - Added detailed tooltips to task rows showing: WBS, Phase, Type, Task Name, Duration, Period, Owners, Status, Predecessors
  - Enhanced task bar tooltips with same comprehensive information
  - Tooltips now provide complete task context on hover

- **Click Integration**: Added click-to-view functionality between gantt and main window
  - Click on task row in gantt window to open task detail in main window
  - Click on task bar in gantt window to open task detail in main window
  - Main window automatically brought to front when task is opened
  - Added `openTaskInMainWindow(wbsNo)` function to handle window communication

### Changed
- **Task List Header CSS** (lines 3405-3418):
  - Changed `padding: 1rem` to `padding: 0 1rem` (horizontal only)
  - Added `height: 50px` and `min-height: 50px`
  - Added `display: flex` and `align-items: center` for vertical centering

- **Timeline Header CSS** (lines 3455-3464):
  - Added `height: 50px` and `min-height: 50px`

- **Timeline Cell CSS** (lines 3470-3481):
  - Changed `padding: 0.5rem` to `padding: 0 0.5rem` (horizontal only)
  - Added `height: 50px`
  - Added `display: flex`, `align-items: center`, `justify-content: center`

### Technical
- Modified task row generation (lines 3670-3688):
  - Added multi-line tooltip with complete task information
  - Added `onclick` handler calling `openTaskInMainWindow()`
  - Added `cursor: pointer` style

- Modified task bar generation (lines 3737-3756):
  - Replaced simple tooltip with comprehensive multi-line tooltip
  - Added `onclick` handler calling `openTaskInMainWindow()`
  - Added `cursor: pointer` style

- Added `openTaskInMainWindow()` function (lines 3849-3856):
  - Checks if main window is still open
  - Calls `window.opener.openTaskDetail(wbsNo)`
  - Brings main window to front with `window.opener.focus()`
  - Shows error if main window is closed

### UI/UX
- Headers now perfectly aligned across task list and timeline
- Hover over any task row or bar to see complete task details
- Click anywhere on task row or bar to jump to task detail in main window
- Seamless navigation between gantt chart and task management
- Improved user workflow for reviewing and editing tasks

## [2.8.2] - 2026-03-20

### Fixed
- **Gantt Chart Display Issues**: Fixed multiple visual problems in gantt chart window
  - Fixed row height mismatch between task list (left) and timeline (right)
  - Fixed task bar text visibility issue (white text on white background)
  - Fixed task bar text overflow when positioned at right edge

### Changed
- **Task Row Height**: Unified task row height to 60px for both task list and timeline
  - Added `height: 60px` to `.task-row`
  - Added `overflow: hidden` to prevent content overflow
  - Added flexbox centering for better vertical alignment
  - Task names now use ellipsis (`...`) when too long

- **Task Bar Width**: Guaranteed minimum task bar width
  - Changed `duration` calculation to `Math.max(3, ...)` for 3% minimum width
  - Changed `startOffset` calculation to `Math.max(0, ...)` to prevent negative values
  - Prevents task bars from becoming invisible

- **Task Bar Text Display**: Adaptive text display based on bar width
  - **15% or wider**: Display full task name
  - **5-15% width**: Display WBS number only
  - **Less than 5%**: No text (tooltip only)
  - Improved tooltip format: `WBS: Task Name\nDate Range`

### Technical
- Modified `.task-row` CSS (lines 3416-3427):
  - Added `height: 60px`
  - Added `display: flex`, `flex-direction: column`, `justify-content: center`
  - Added `overflow: hidden`

- Modified `.task-wbs` and `.task-name` CSS (lines 3429-3445):
  - Added `overflow: hidden`
  - Added `text-overflow: ellipsis`
  - Added `white-space: nowrap`

- Modified task bar rendering logic (lines 3685-3726):
  - Added minimum width guarantee (3%)
  - Added adaptive text display based on bar width
  - Enhanced tooltip with WBS number and line breaks

### UI/UX
- Task list and timeline rows now perfectly aligned
- Task bars are always visible (minimum 3% width)
- Task bar text adapts to available space
- Long task names show ellipsis instead of wrapping
- Improved tooltip readability with structured format
- No text overflow at timeline right edge

## [2.8.1] - 2026-03-20

### Fixed
- **Gantt Chart Display Bug**: Fixed JavaScript code appearing in gantt chart window
  - Escaped `</script>` tag in gantt HTML template literal (`<\/script>`)
  - Prevents browser from misinterpreting closing script tag
  - Gantt chart window now displays correctly without code artifacts

### Technical
- Changed line 3803: `</script>` → `<\/script>`
- Template literal escape fix for `document.write()` compatibility

## [2.8.0] - 2026-03-20

### Added
- **Gantt Chart Visualization**: Separate window display for project timeline
  - New "ガントチャート" button in header navigation
  - Opens in independent browser window (not tab) for side-by-side view
  - Timeline display with task bars showing start/finish dates
  - Status-based color coding (completed: green, in-progress: blue, not-started: gray, on-hold: orange, cancelled: red)
  - View mode switching: Day/Week/Month units
  - Responsive design with automatic layout adjustments
  - Real-time data synchronization from main window (5-second auto-refresh)
  - Manual refresh button for on-demand updates
  - Today indicator line
  - Task information tooltips on hover

### Features
- **Gantt Chart Display**:
  - Left column: Task list (WBS number, phase, task name)
  - Right column: Timeline with horizontal task bars
  - Color-coded task bars by status
  - Scrollable timeline for long projects
  - Sticky headers for easy navigation

- **Responsive Timeline**:
  - Large screens (1200px+): Daily view (day-by-day columns)
  - Medium screens (800-1199px): Weekly view (week-by-week columns)
  - Small screens (800px-): Monthly view (month-by-month columns)
  - Automatic grid adjustment based on screen width
  - Horizontal and vertical scrolling support

- **Data Synchronization**:
  - Automatic sync every 5 seconds via `window.opener`
  - Manual refresh button in gantt window
  - Real-time updates when tasks are modified in main window
  - Handles main window closure gracefully

### Technical
- Added `openGanttChart()` function in main window
- Added `generateGanttHTML()` function to create standalone gantt HTML
- Uses `window.open()` with custom dimensions (1400x800, resizable)
- Gantt window contains fully embedded HTML/CSS/JavaScript (no external dependencies)
- CSS Grid layout for task list and timeline columns
- Flexbox for timeline cells and responsive controls
- JavaScript-based timeline generation (day/week/month modes)
- Task filtering: Only displays tasks with both start_date and finish_date
- Date range auto-calculation with 3-day padding
- Position calculation using percentage-based offsets

### UI/UX
- Clean, professional design matching main application
- Navy color scheme consistent with v2.7.1
- Legend showing status colors
- Hover effects on task bars for enhanced visibility
- Empty state handling (message when no tasks have dates)
- Responsive button layout in controls
- Window title shows project name

### Constraints
- **No External Libraries**: Pure HTML/CSS/JavaScript only
- **Single File Architecture**: All gantt code embedded in main HTML
- **Edge Standard APIs**: Uses only `window.open()`, basic DOM manipulation
- **Security Compliant**: No CDN, no external resources, offline-capable

## [2.7.1] - 2026-03-20

### Changed
- **UI Improvement**: Replaced text input dialogs with modal selection dialogs
  - Export selection now uses clickable buttons instead of `prompt()` text input
  - Import selection now uses clickable buttons instead of `prompt()` text input
  - Improved user experience with visual selection interface
  - Modal dialogs properly centered on screen using flexbox layout
- **Button Color Redesign**: Changed modal selection buttons to calming navy/dark color scheme
  - Export/Import buttons: Navy (#2c3e50) with hover effect (#34495e)
  - AI template buttons: Dark navy gradient (#1a252f to #2c3e50)
  - Improved visual hierarchy and professional appearance
- **New Task Creation Button**: Redesigned task creation button
  - Moved from filter bar to "タスク一覧" header (right side)
  - Changed from text button to icon-only button
  - Uses SVG icon (square with plus symbol)
  - Tooltip shows "新規タスク作成" on hover

### Technical
- Added `export-selection-dialog` modal with 4 option buttons
- Added `import-selection-dialog` modal with 2 option buttons
- Modified `showExport()` to display modal dialog with `display: flex` (for proper centering)
- Modified `showImportDialog()` to display modal dialog with `display: flex` (for proper centering)
- Added `closeExportDialog()` function
- Added `closeImportDialog()` function
- Added click-outside-to-close event listeners for both dialogs
- Removed old text-based "新規タスク作成" button from filter bar
- Added icon-based "新規タスク作成" button to card header

### UI/UX
- Export/Import options now displayed as large, descriptive buttons
- AI template options visually distinguished with darker gradient background
- Each option includes icon, title, and description
- Consistent modal design with other dialogs in the application
- Modal dialogs properly centered using existing flexbox layout
- New task creation button more compact and integrated into header

## [2.7.0] - 2026-03-20

### Added
- **AI Template Export**: AI-powered task generation templates
  - Integrated into Export button (options 3 & 4)
  - Two template formats available:
    - JSON template with comprehensive prompt
    - CSV template with comprehensive prompt
  - Detailed prompts for ChatGPT, Claude, Gemini, and other AI services
  - Templates include strict formatting rules and validation checklists

### Features
- **Comprehensive Prompt Templates**:
  - Project setup instructions with required fields
  - WBS number hierarchy rules (WBS1.0.0 → WBS1.1.0 → WBS1.1.1)
  - Phase selection guidelines (WBS0, PH1, PH2, TEST, RELEASE)
  - Date format specifications (YYYY/M/D for JSON, YYYY-MM-DD for CSV)
  - Status and priority distribution recommendations
  - Field-by-field detailed explanations
  - Sample task data for reference
  - Pre-export validation checklist
  - Step-by-step usage instructions

- **Template Content**:
  - JSON version: ~10KB Markdown file with embedded JSON structure
  - CSV version: ~8KB Markdown file with embedded CSV format
  - File naming: `task-manager-ai-prompt-{format}-{date}.md`
  - Single-file design (prompt + template in one document)

### Changed
- Export dialog expanded from 2 to 4 options
- Export functionality now supports both data export and template download
- No additional UI buttons (integrated into existing Export button)

### Technical
- Added `downloadAITemplate(format)` function
- Added `getJSONPromptTemplate()` with embedded template
- Added `getCSVPromptTemplate()` with embedded template
- Modified `showExport()` to handle 4 options
- Template content embedded as JavaScript string literals
- File size impact: ~15KB added to HTML file

### Use Case
1. Click "Export" button
2. Select option 3 (JSON template) or 4 (CSV template)
3. Download Markdown file with AI prompt
4. Copy prompt to ChatGPT/Claude/Gemini
5. Replace [Project Name] with actual project
6. AI generates structured task data
7. Import generated data via "Import" button

## [2.6.1] - 2026-03-20

### Changed
- **Save Button Redesign**: Updated save button icon to clarify save/overwrite functionality
  - Changed from download icon to floppy disk icon
  - Indicates initial save dialog and subsequent overwrite behavior
  - Title changed from "ファイルに保存" to "保存" for brevity
- **Button Layout Improvement**: Reorganized header navigation buttons
  - Moved Save button to rightmost position (was 3rd)
  - Moved Import button to 3rd position (was rightmost)
  - New order: Dashboard → Export → Import → Save

### Fixed
- **New Task Button Visibility**: Fixed ➕ emoji color in "New Task Creation" button
  - Changed emoji color to white for better visibility
  - Previously appeared in red/default color on blue background (difficult to read)

## [2.6.0] - 2026-03-20

### Added
- **Task Information Edit Mode**: Manual editing for task details, verification method, and pass criteria
  - Section-based edit mode toggle with explicit "Edit" button
  - Edit/Cancel/Save workflow for safe data modification
  - Real-time validation (task name required)
  - Automatic change tracking with system comments

### Changed
- **Enhanced Task Details Modal**: Improved task information display
  - Added WBS number to view mode
  - Added task name display in view mode
  - Clear separation between view mode and edit mode
  - Better visual hierarchy with edit button in top-right corner

### Features
- **Editable Fields**:
  - Task Name (required)
  - Task Type
  - Primary Owner
  - Support Owner
  - Duration (Business Days)
  - Predecessors
  - Verification Method (multi-line)
  - Pass Criteria (multi-line)

- **Read-Only Fields** (shown in view mode):
  - WBS Number (structural identifier)
  - Phase (structural grouping)
  - Period (editable in separate "Period Edit" section)

- **Safety Features**:
  - Confirmation required before saving
  - Cancel button to discard changes
  - Edit mode resets when closing modal
  - System comments log all changes

### Technical
- Added `isEditingTaskInfo` global state variable
- Modified `renderTaskDetails()` to support dual modes
- Added `toggleTaskInfoEdit()` for mode switching
- Added `saveTaskInfo()` with validation and change tracking
- Added `cancelTaskInfoEdit()` for safe cancellation
- Modified `closeModal()` to reset edit state

## [2.5.0] - 2026-03-20

### Added
- **Unified Import Dialog**: Single import button with JSON/CSV format selection
  - Users can now choose between JSON (project file) or CSV (task list) import
  - Improved user experience with clear format descriptions

### Changed
- **Import Button Redesign**: Updated import button icon using docs/ui/import.svg
  - New cleaner import icon design (arrow into document)
  - Changed button title from "CSVインポート" to "インポート"
- **Removed "Open from File" Button**: Integrated into new unified import button
  - Consolidated two buttons (Open from File + CSV Import) into one
  - Reduced header menu clutter

### Technical
- Added `showImportDialog()` function for format selection
- Import button now calls `showImportDialog()` instead of direct function calls
- Maintained backward compatibility with existing import functions

## [2.4.1] - 2026-03-20

### Fixed
- **CSV Import Bug**: Fixed `hasUnsavedChanges is not defined` error
  - Changed `hasUnsavedChanges` to `isDirty` (correct variable name)
  - Changed `markUnsaved()` to `markDirty()` (correct function name)
  - CSV import now properly checks for unsaved changes before importing

## [2.4.0] - 2026-03-20

### Added
- **CSV Import Functionality**: Import WBS tasks from CSV files
  - Complete data replacement mode (existing data is fully replaced)
  - CSV parser with RFC 4180 compliance (handles quoted fields, escaped quotes)
  - Data validation with required field checking (WBS_No, Phase, Task_Name)
  - Error handling with partial success support (skip invalid rows, continue processing)
  - Detailed error reporting (shows first 5 errors with row numbers)
  - Confirmation dialogs for unsaved changes and data replacement
- New CSV import button in header menu with table+upload icon
- Support for all CSV columns from sample format (14 columns total)

### Changed
- Version updated from v2.3.0 to v2.4.0
- File handle cleared after CSV import (imported data treated as new file)
- Import triggers automatic save to LocalStorage and UI refresh

### Technical
- Added `importFromCSV()` function (~80 lines)
- Added `parseCSV(text)` function with row-by-row validation
- Added `parseCSVLine(line)` function with proper quote handling
- CSV import integrated into existing file menu workflow
- Maintains single-file architecture with no external dependencies

## [2.3.0] - 2026-03-20

### Added
- **Project Generalization**: Transformed from Dify-specific tool to generic task management application
- MIT License for open source distribution
- Comprehensive English documentation (README.md)
- Sample CSV file with realistic project example (examples/sample-wbs.csv)
- GitHub repository structure with docs/, examples/, and .github/ directories
- .gitignore configuration for project files
- Version badges in README

### Changed
- **Breaking**: Renamed file from `dify-wbs-visualizer.html` to `task-manager.html`
- **Breaking**: Changed LocalStorage key from `dify_wbs_v1_project_data` to `task_manager_v2_project_data`
- Updated application title from "Dify WBS Visualizer" to "Task Manager"
- Changed HTML language attribute from Japanese (`ja`) to English (`en`)
- Replaced 72-task Dify-specific initial data with empty project template
- Updated export filenames from `dify-wbs-*` to `task-manager-*`
- Default project name changed from Japanese to "New Project"
- Added SEO meta tags (description, keywords, author)

### Fixed
- Operation menu (⋮) overflow when opened near bottom of viewport
- Modal header no longer scrolls away (now sticky positioned)
- Reduced modal header padding for better space utilization

### Technical
- Maintained single-file architecture with no external dependencies
- Ensured backward compatibility through LocalStorage versioning
- File size: ~200-300KB, ~3,800 lines

## [2.2.0] - 2026-03-19

### Added
- Task position movement feature (drag-and-drop within same phase)
- New task creation at any position (insert mode)
- Up/down arrow icons (↕) for drag handles
- Plus (+) buttons for inserting new tasks
- Visual feedback during drag operations

### Changed
- Task rows now support drag-and-drop reordering
- Task numbers (WBS_No) automatically recalculated after position changes
- Improved task list interactivity

### Technical
- Implemented drag-and-drop API (dragstart, dragover, drop events)
- Added task insertion logic with automatic index management
- Enhanced CSS for drag-and-drop visual feedback

## [2.1.0] - 2026-03-18

### Added
- Comment system for individual tasks
- Timestamped note-taking functionality
- Comment history display in task detail modal
- File System Access API integration for save/load
- Automatic save to previously selected file location
- "Save As" functionality for new save locations

### Changed
- Enhanced task detail modal with comment section
- Improved data persistence with file-based storage
- LocalStorage now serves as backup storage

### Fixed
- File handle persistence across sessions
- Save dialog suggested filename format

## [2.0.0] - 2026-03-17

### Added
- Dashboard with project progress visualization
- Phase-based progress bars with color coding
- Task status management (Not Started, In Progress, Completed, On Hold, Cancelled)
- Task detail modal for editing
- Owner assignment (primary and support)
- Date tracking (start/finish dates)
- Duration calculation in business days
- Predecessor task tracking
- Verification method and pass criteria fields
- Export to CSV (Excel-compatible with BOM UTF-8)
- Export to JSON
- LocalStorage automatic backup

### Changed
- Major UI overhaul with modern design
- Tabbed interface for different views
- Color-coded status badges
- Responsive layout improvements

### Technical
- Vanilla JavaScript implementation (no frameworks)
- CSS Grid and Flexbox layout
- ES6+ features (arrow functions, template literals, modules)
- Single HTML file architecture

## [1.0.0] - 2026-03-16

### Added
- Initial release as "Dify WBS Visualizer"
- Basic task list display
- CSV import from Dify WBS v1.0.0
- Simple task viewing
- JSON data structure
- Basic styling with CSS

### Technical
- Single HTML file with embedded data
- Dify-specific 72-task dataset
- Japanese language interface
- Read-only visualization

---

## Migration Notes

### v2.3.0 Breaking Changes

If you are upgrading from a previous version (v2.2.0 or earlier):

1. **File Rename**: The application file has been renamed from `dify-wbs-visualizer.html` to `task-manager.html`
2. **LocalStorage Key Change**: Your saved data in LocalStorage will NOT automatically migrate. To preserve your data:
   - Before upgrading: Use "Export → JSON" to save your project data
   - After upgrading: Use "Open from File" to load your saved JSON file
3. **Initial Data**: New installations start with an empty project instead of 72 pre-loaded Dify tasks

### Backward Compatibility

- JSON export files from v2.x are fully compatible with v2.3.0
- CSV export files can be re-imported (CSV import feature coming soon)
- File System Access API save files are fully compatible

---

## Planned Features

### v2.4.0 (Next Release)
- CSV import functionality with complete data replacement mode
- Import validation and error handling
- Unsaved data warning before import
- Import confirmation dialog

### v2.5.0 (Future)
- Flowchart visualization (SVG-based, no Mermaid.js)
- Task-to-flowchart node linking
- Advanced search and filter capabilities
- Multi-criteria sorting

### v3.0.0 (Future)
- Dark mode support
- Keyboard shortcuts
- Undo/redo functionality
- Task templates
- Gantt chart view
- Resource allocation tracking

---

## Support

For bug reports, feature requests, or questions:
- **Issues**: [GitHub Issues](https://github.com/ks-source/task-manager/issues)
- **Discussions**: [GitHub Discussions](https://github.com/ks-source/task-manager/discussions)

---

[2.8.2]: https://github.com/ks-source/task-manager/compare/v2.8.1...v2.8.2
[2.8.1]: https://github.com/ks-source/task-manager/compare/v2.8.0...v2.8.1
[2.8.0]: https://github.com/ks-source/task-manager/compare/v2.7.1...v2.8.0
[2.7.1]: https://github.com/ks-source/task-manager/compare/v2.7.0...v2.7.1
[2.7.0]: https://github.com/ks-source/task-manager/compare/v2.6.1...v2.7.0
[2.6.1]: https://github.com/ks-source/task-manager/compare/v2.6.0...v2.6.1
[2.6.0]: https://github.com/ks-source/task-manager/compare/v2.5.0...v2.6.0
[2.5.0]: https://github.com/ks-source/task-manager/compare/v2.4.1...v2.5.0
[2.4.1]: https://github.com/ks-source/task-manager/compare/v2.4.0...v2.4.1
[2.4.0]: https://github.com/ks-source/task-manager/compare/v2.3.0...v2.4.0
[2.3.0]: https://github.com/ks-source/task-manager/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/ks-source/task-manager/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/ks-source/task-manager/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/ks-source/task-manager/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/ks-source/task-manager/releases/tag/v1.0.0
