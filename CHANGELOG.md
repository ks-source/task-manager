# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[2.6.0]: https://github.com/ks-source/task-manager/compare/v2.5.0...v2.6.0
[2.5.0]: https://github.com/ks-source/task-manager/compare/v2.4.1...v2.5.0
[2.4.1]: https://github.com/ks-source/task-manager/compare/v2.4.0...v2.4.1
[2.4.0]: https://github.com/ks-source/task-manager/compare/v2.3.0...v2.4.0
[2.3.0]: https://github.com/ks-source/task-manager/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/ks-source/task-manager/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/ks-source/task-manager/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/ks-source/task-manager/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/ks-source/task-manager/releases/tag/v1.0.0
