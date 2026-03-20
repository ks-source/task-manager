# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
