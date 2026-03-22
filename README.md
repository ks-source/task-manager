# Task Manager - WBS Visualizer & Flowchart Editor

A lightweight, browser-based Work Breakdown Structure (WBS) visualization and project management tool with integrated flowchart editor. No installation required, no external dependencies, works completely offline.

![Version](https://img.shields.io/badge/version-2.4.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Browser](https://img.shields.io/badge/browser-Edge%2090%2B-blue)

## 📁 Quick Start Files

```
/dev/task-manager/
  ├── task-manager.html           ← タスク管理メイン画面
  └── flowchart-editor.html       ← フローチャート編集画面（新機能）
```

**新機能**: タスクとフローチャートの自動同期機能を追加しました。詳細は下記「フローチャート連携」セクションをご覧ください。

## 🌟 Features

### Core Functionality
- **📊 Interactive Dashboard**: Real-time project progress visualization with phase-based progress bars
- **📝 Task Management**: Create, edit, and track tasks with detailed metadata
- **🗂️ Phase Organization**: Support for multiple project phases (e.g., WBS0, PH1, PH2, TEST, RELEASE)
- **✅ Status Tracking**: Comprehensive status management (Not Started, In Progress, Completed, On Hold, Cancelled)
- **💾 Persistent Storage**: Automatic LocalStorage backup with manual file save/load
- **📤 Export Options**: Export to JSON or CSV (Excel-compatible with BOM UTF-8)

### Advanced Features
- **🎯 Task Positioning**: Drag-and-drop task reordering within the same phase
- **➕ Quick Task Creation**: Insert new tasks at any position
- **📅 Date Management**: Track start/finish dates with duration calculation
- **👥 Owner Assignment**: Primary and support owner fields
- **🔗 Task Dependencies**: Predecessor tracking for workflow management
- **💬 Comments**: Add timestamped notes to individual tasks
- **🎨 Visual Indicators**: Color-coded status badges and progress indicators
- **🔍 Responsive UI**: Adapts to different screen sizes

### 🆕 Flowchart Editor Features (v2.4.0)
- **🔄 Real-time Sync**: Automatic synchronization between task manager and flowchart editor
- **🎨 Drag & Drop**: Link tasks to flowchart nodes by dragging from task panel
- **📊 Visual Workflow**: Visualize project flow with SVG-based flowchart
- **🔵 Status Colors**: Nodes automatically colored based on task status
- **📋 Task Panel**: Side panel showing all tasks with sync status
- **💾 Dual Storage**: LocalStorage for sync + File System for permanent storage

## 🚀 Quick Start

### Option 1: Direct Usage (Recommended)
1. Download `task-manager.html` and `flowchart-editor.html` to your computer
2. Double-click `task-manager.html` to open it in Microsoft Edge
3. Start creating tasks or import a CSV file
4. (Optional) Open `flowchart-editor.html` in a separate tab for flowchart visualization

### Option 2: GitHub Pages
Visit the hosted version at: [https://ks-source.github.io/task-manager](https://ks-source.github.io/task-manager)

---

## 🔄 Flowchart Editor Integration (NEW)

### How It Works

The flowchart editor automatically synchronizes with your task manager using **LocalStorage + File System Access API**.

```
┌─────────────────────────────────────────────────┐
│ 1. task-manager.html でタスク編集               │
│    ↓                                            │
│ 2. LocalStorage へ即座に保存（リアルタイム同期）│
│    ↓                                            │
│ 3. 本体ファイルへ自動保存（3秒デバウンス）       │
│    ↓                                            │
│ 4. flowchart-editor.html が検知（storage event）│
│    ↓                                            │
│ 5. フローチャートが自動更新                      │
└─────────────────────────────────────────────────┘
```

### Step-by-Step Guide

#### Step 1: Open Task Manager
```
file:///C:/dev/task-manager/task-manager.html
```
1. Create or edit tasks
2. Tasks are automatically saved to LocalStorage and file

#### Step 2: Open Flowchart Editor (別タブ)
```
file:///C:/dev/task-manager/flowchart-editor.html
```
1. Task panel automatically loads your tasks
2. Drag tasks from panel to flowchart nodes
3. Nodes are linked to tasks and sync automatically

#### Step 3: Real-time Sync
- Edit task status in task-manager → Flowchart colors update
- Link tasks to nodes in flowchart → Task manager shows flowchart info
- Both files stay in sync automatically

### Data Storage Architecture

**Two-layer protection**:
1. **LocalStorage** (5MB): Real-time sync buffer
2. **File System** (unlimited): Permanent master data

**Important**: Even if LocalStorage reaches capacity, your data in files is safe.

### Troubleshooting

**Q: Files don't sync?**
- Ensure both files are opened with `file://` protocol
- Check both are in the same directory

**Q: LocalStorage quota exceeded?**
- Archive completed tasks (button in task-manager)
- Data in files remains safe

**Q: Lost data?**
- Open task-manager.html → "Open from File"
- Select your .json file → All data restored

For detailed specifications, see [docs/specs/cross-html-sync/README.md](./docs/specs/cross-html-sync/README.md)

## 📖 Usage Guide

### Creating a New Project

1. **Manual Task Creation**:
   - Click the **"+"** button next to any task row to insert a new task
   - Fill in task details in the modal dialog
   - Click "Save" to add the task

2. **CSV Import** (Coming in v2.3.0):
   - Prepare a CSV file following the format specification below
   - Click "File" → "Open from File"
   - Select your CSV file

### Managing Tasks

#### Status Changes
- Click any task row to open the detail modal
- Use the status dropdown to update task progress
- Changes are automatically saved to LocalStorage

#### Task Details
Each task includes:
- **WBS Number**: Unique identifier (e.g., WBS1.2.3)
- **Phase**: Project phase grouping
- **Task Type**: Category (e.g., Design, Development, Testing)
- **Task Name**: Descriptive title
- **Duration**: Work days required
- **Start/Finish Dates**: Schedule tracking
- **Owners**: Primary and support assignees
- **Predecessors**: Dependent task references
- **Verification**: QA method and pass criteria
- **Status**: Current state
- **Comments**: Notes and history

#### Reordering Tasks
- Click and hold the **"↕"** icon on any task row
- Drag the task to a new position within the same phase
- Release to drop

### File Operations

#### Save Project
- **Menu → Save**: Save to previously selected location (if any)
- **Menu → Save As**: Choose new save location
- Saved as JSON with date suffix (e.g., `task-manager-2026-03-20.json`)

#### Open Project
- **Menu → Open from File**: Load a previously saved JSON project file
- All task data, comments, and metadata will be restored

#### Export Data
- **Menu → Export → JSON**: Export project data in JSON format
- **Menu → Export → CSV**: Export task list in Excel-compatible CSV format

## 📋 CSV Format Specification

### CSV Structure

The CSV file must include the following columns (order must match):

```csv
WBS_No,Phase,Task_Type,Task_Name,Mermaid_IDs,Primary_Owner,Support_Owner,Duration_BD,Start_Date,Finish_Date,Predecessors,Verification_Method,Pass_Criteria,Status
```

### Column Descriptions

| Column | Required | Format | Example | Description |
|--------|----------|--------|---------|-------------|
| `WBS_No` | ✅ | String | `WBS1.2.3` | Unique task identifier |
| `Phase` | ✅ | String | `PH1`, `TEST` | Project phase code |
| `Task_Type` | ❌ | String | `Design`, `Dev` | Task category |
| `Task_Name` | ✅ | String | `Create login UI` | Task description |
| `Mermaid_IDs` | ❌ | String | `F_01,F_02` | Flowchart node references |
| `Primary_Owner` | ❌ | String | `John` | Primary assignee |
| `Support_Owner` | ❌ | String | `Jane` | Support assignee |
| `Duration_BD` | ❌ | Integer | `5` | Work days (business days) |
| `Start_Date` | ❌ | YYYY-MM-DD | `2026-03-23` | Planned start date |
| `Finish_Date` | ❌ | YYYY-MM-DD | `2026-03-28` | Planned finish date |
| `Predecessors` | ❌ | String | `WBS1.2.1,WBS1.2.2` | Dependent tasks |
| `Verification_Method` | ❌ | String | `Unit test` | QA method |
| `Pass_Criteria` | ❌ | String | `100% coverage` | Acceptance criteria |
| `Status` | ❌ | String | `Not Started` | Current status |

### Valid Status Values

- `Not Started` (default)
- `In Progress`
- `Completed`
- `On Hold`
- `Cancelled`

### Sample CSV

See `examples/sample-wbs.csv` for a complete working example.

```csv
WBS_No,Phase,Task_Type,Task_Name,Mermaid_IDs,Primary_Owner,Support_Owner,Duration_BD,Start_Date,Finish_Date,Predecessors,Verification_Method,Pass_Criteria,Status
WBS1.0.0,WBS0,Planning,Project Kickoff,,PM,Team Lead,1,2026-03-23,2026-03-23,,Meeting minutes,All stakeholders present,Completed
WBS1.1.0,PH1,Design,UI Mockups,F_01,Designer,Developer,3,2026-03-24,2026-03-26,WBS1.0.0,Design review,Approved by PM,In Progress
WBS1.2.0,PH1,Development,Frontend Implementation,F_01,Developer,,5,2026-03-27,2026-04-02,WBS1.1.0,Code review,Tests passing,Not Started
```

## 🖥️ Browser Compatibility

### Supported Browsers
- ✅ **Microsoft Edge 90+** (Primary target, fully supported)
- ⚠️ **Google Chrome 90+** (Most features work, File System Access API requires user permissions)
- ⚠️ **Brave 1.30+** (Most features work, File System Access API requires user permissions)

### Unsupported Browsers
- ❌ Firefox (File System Access API not supported as of 2026)
- ❌ Safari (File System Access API not supported)
- ❌ Internet Explorer (All versions)

### Required Browser Features
- HTML5
- CSS3 (Grid, Flexbox)
- ES6+ JavaScript
- LocalStorage API
- File System Access API (for save/load functionality)

## 🛠️ Technical Details

### Architecture
- **Single HTML File**: All code, styles, and initial data embedded
- **No External Dependencies**: No CDN, no npm packages, no build tools
- **Vanilla JavaScript**: Pure ES6+ without frameworks
- **Edge Standard APIs**: Only browser-native functionality

### Data Storage
- **Primary**: File System Access API (user-controlled save location)
- **Backup**: LocalStorage (automatic, 5-10MB limit)
- **Format**: JSON (pretty-printed for readability)

### File Size
- ~3,800 lines of code
- ~200-300KB uncompressed
- Opens instantly, no loading time

## 📁 Project Structure

```
task-manager/
├── task-manager.html          # Main task management application
├── flowchart-editor.html      # Flowchart editor (NEW in v2.4.0)
├── index.html                 # Entry point
├── README.md                  # This file
├── LICENSE                    # MIT License
├── CHANGELOG.md              # Version history
├── .gitignore                # Git ignore rules
├── docs/                     # Documentation
│   ├── specs/                # Technical specifications
│   │   ├── cross-html-sync/           # LocalStorage sync specs
│   │   ├── flowchart-manual-editing/  # Flowchart editing specs
│   │   └── gantt-interactive-editing/ # Gantt chart specs
│   ├── screenshots/          # UI screenshots
│   └── ui/                   # UI mockups and prototypes
│       └── flowchart-mockup-interactive.html  # Archive (development history)
├── examples/                 # Sample files
│   ├── sample-wbs.csv       # Example CSV import
│   └── templates/           # Project templates
└── .github/                 # GitHub configuration
    ├── workflows/           # CI/CD pipelines
    └── ISSUE_TEMPLATE/      # Issue templates
```

## 🤝 Contributing

Contributions are welcome! Please follow these guidelines:

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/amazing-feature`)
3. **Commit** your changes (`git commit -m 'Add some amazing feature'`)
4. **Push** to the branch (`git push origin feature/amazing-feature`)
5. **Open** a Pull Request

### Development Guidelines
- Maintain single-file architecture (no external dependencies)
- Use only Edge standard APIs
- Test on Microsoft Edge 90+
- Update version number in HTML title and meta data
- Add changelog entry for all changes

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🐛 Bug Reports & Feature Requests

Please use [GitHub Issues](https://github.com/ks-source/task-manager/issues) to report bugs or request features.

### When reporting bugs, please include:
- Browser version
- Operating system
- Steps to reproduce
- Expected vs actual behavior
- Screenshots (if applicable)

## 📞 Support

- **Documentation**: [Wiki](https://github.com/ks-source/task-manager/wiki)
- **Issues**: [GitHub Issues](https://github.com/ks-source/task-manager/issues)
- **Discussions**: [GitHub Discussions](https://github.com/ks-source/task-manager/discussions)

## 🗺️ Roadmap

### v2.3.0 (Released - 2026-03-20)
- ✅ Generalized from Dify-specific tool to generic task management application
- ✅ Single-file HTML architecture with no external dependencies
- ✅ Complete documentation (README, CHANGELOG, LICENSE)
- ✅ Sample CSV file for import testing
- ✅ GitHub repository setup with Pages support

### v2.4.0 (Released - 2026-03-21)
- ✅ Flowchart editor (SVG-based, no external libraries)
- ✅ Task-to-flowchart linking via drag & drop
- ✅ Real-time sync between task manager and flowchart
- ✅ LocalStorage + File System Access API integration
- ✅ Complete technical specifications (cross-html-sync)
- ✅ Dual-layer data protection architecture

### v2.5.0 (In Progress)
- [ ] LocalStorage sync implementation (Phase 1)
- [ ] Storage event handlers
- [ ] Automatic file save (3-second debounce)
- [ ] Task panel UI in flowchart editor
- [ ] Drag & drop node linking

### v2.6.0 (Planned)
- [ ] LocalStorage capacity management
- [ ] Completed task archiving
- [ ] Error handling & recovery
- [ ] CSV import functionality
- [ ] Search and filter capabilities

### v2.7.0 (Future)
- [ ] Dark mode support
- [ ] Keyboard shortcuts
- [ ] Undo/redo functionality
- [ ] Task templates

## ⭐ Acknowledgments

- Inspired by traditional WBS project management methodologies
- Built for modern web browsers without external dependencies
- Designed for offline-first, privacy-focused task management

---

**Made with ❤️ by [ks-source](https://github.com/ks-source)**

*Version 2.4.0 | Last Updated: 2026-03-21*
