# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[2.3.0]: https://github.com/ks-source/task-manager/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/ks-source/task-manager/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/ks-source/task-manager/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/ks-source/task-manager/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/ks-source/task-manager/releases/tag/v1.0.0
