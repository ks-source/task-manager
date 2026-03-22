# Phase 2-A: タスク一覧同期（A→B）の実装

---
**feature**: データ連携機能（Phase 2-A）
**issue**: A側の全タスクをB側の左サイドパネルに自動表示
**version**: 1.0
**status**: 仕様確定、実装待ち
**created**: 2026-03-22
**updated**: 2026-03-22
**priority**: 🔴 Critical
**author**: Claude Code Session
**前提条件**: Phase 1修正（mermaid_ids双方向同期）完了
---

## 📋 概要

### ユーザーの期待するフロー

```
【ステップ1】A側でタスク作成
A: task-manager.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ task_name: "要件定義"    │
│ mermaid_ids: ""         │ ← 空でOK（B側で割当前）
└─────────────────────────┘
        ↓ LocalStorage保存

【ステップ2】B側で全タスク表示 ★Phase 2-A
B: flowchart-editor.html
┌─────────────────────────┐
│ 左サイドパネル           │
│ ┌─────────────────────┐ │
│ │ □ WBS1.1.0          │ │ ← A側の全タスクが表示
│ │   要件定義           │ │    （mermaid_ids空でも表示）
│ │                     │ │
│ │ □ WBS1.2.0          │ │
│ │   基本設計           │ │
│ └─────────────────────┘ │
└─────────────────────────┘
        ↓ ユーザー操作

【ステップ3】B側でフローチャート要素割当
- タスク編集モーダルでnode_1を追加
- mermaidIds: "node_1"に更新
        ↓ Ctrl+S保存

【ステップ4】A側へ反映 ★Phase 1修正（完了済み）
A: task-manager.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ mermaid_ids: "node_1"   │ ← B側の割当が反映される✅
└─────────────────────────┘
```

### Phase 2-Aの実装範囲

**実装する機能**:
- ✅ A側のタスクリストをLocalStorageから読み込み
- ✅ mockTasksをA側データで置き換え
- ✅ mermaid_ids空のタスクも表示
- ✅ タスク追加・削除時の自動更新（storage event）

**実装しない機能**（Phase 2-B以降）:
- ❌ 双方向ハイライト（ノード↔タスク）
- ❌ タスク一覧パネルの洗練されたUI
- ❌ フィルタリング・検索機能

---

## 🔍 現状分析

### 現在の問題

**flowchart-editor.html の mockTasks**（lines 2836-2913）:

```javascript
let mockTasks = [
  {
    wbs: "WBS1.1.0",
    taskName: "要件ヒアリング",
    status: "Completed",
    mermaidIds: "F_01",
    flowchartLinks: {
      nodes: [
        { id: "F_01", label: "要件定義", highlightType: "primary" }
      ],
      edges: []
    }
  },
  // ... ハードコードされた5つのタスク
];
```

**問題点**:
1. ❌ **ハードコード**: 5つのデモタスクが固定
2. ❌ **手動編集が必要**: A側でタスク作成してもB側に表示されない
3. ❌ **データ不整合**: mockTasksとA側のタスクが別物
4. ❌ **スケーラビリティ**: タスク数が増えるたびに手動編集が必要

### 既存の実装（活用できる部分）

**✅ Phase 1で実装済み**:

1. **loadTaskManagerData()** (lines 3487-3502)
   - LocalStorageから task-manager-data を読み込む
   - A側のタスクリストを取得可能

2. **publishFlowchartAttributes()** (lines 3508-3564)
   - B→A送信（Phase 1修正で mermaid_ids も送信）

3. **renderTaskList()** (既存)
   - mockTasks をレンダリング

**Phase 2-Aで必要な拡張**:
- mockTasks の初期化処理を変更
- storage event リスナーを追加
- A側データから mockTasks 形式への変換

---

## 🔧 技術仕様

### 修正1: mockTasks初期化処理の変更

**ファイル**: `flowchart-editor.html`
**場所**: lines 2836-2913（現在のmockTasks定義）
**修正内容**: loadTaskManagerData()を使用して動的初期化

#### 修正前のコード

```javascript
// 現状: ハードコードされたデモデータ
let mockTasks = [
  {
    wbs: "WBS1.1.0",
    taskName: "要件ヒアリング",
    status: "Completed",
    mermaidIds: "F_01",
    flowchartLinks: { ... }
  },
  // ... 4個のタスク
];
```

#### 修正後のコード

```javascript
// ★Phase 2-A: タスクマネージャーからデータを読み込む
let mockTasks = [];  // 初期化時は空

/**
 * タスクマネージャーのデータをmockTasks形式に変換
 */
function initializeTasksFromTaskManager() {
  const taskManagerData = loadTaskManagerData();

  if (!taskManagerData || !taskManagerData.tasks) {
    console.log('[FlowchartEditor] Task Manager not connected - using empty task list');
    mockTasks = [];
    return;
  }

  // A側のタスクをmockTasks形式に変換
  mockTasks = taskManagerData.tasks.map(task => {
    const mermaidIds = task.mermaid_ids || "";

    return {
      wbs: task.wbs_no,
      taskName: task.task_name || task.taskName || "（名前なし）",
      status: task.status || "Not Started",
      mermaidIds: mermaidIds,
      flowchartLinks: parseMermaidIds(mermaidIds)
    };
  });

  console.log(`[FlowchartEditor] Initialized ${mockTasks.length} tasks from Task Manager`);
}

/**
 * mermaid_ids文字列をflowchartLinks形式に変換
 */
function parseMermaidIds(mermaidIds) {
  if (!mermaidIds || mermaidIds.trim() === "") {
    return { nodes: [], edges: [] };
  }

  const ids = mermaidIds.split(',').map(id => id.trim()).filter(id => id);
  const nodes = [];
  const edges = [];

  ids.forEach(id => {
    // ノードIDかエッジIDかを判定
    if (id.startsWith('E_')) {
      // エッジの場合（簡易実装：詳細情報は取得しない）
      edges.push({
        id: id,
        from: "",  // 実際のエッジ情報はSVGから取得が必要
        to: "",
        label: "",
        highlightType: "primary",
        highlightMode: "full"
      });
    } else {
      // ノードの場合
      nodes.push({
        id: id,
        label: "",  // ラベルはSVGから取得が必要
        highlightType: "primary"
      });
    }
  });

  return { nodes, edges };
}
```

### 修正2: SVG読み込み時の初期化

**ファイル**: `flowchart-editor.html`
**場所**: SVG読み込み完了時（line 5021付近）
**修正内容**: initializeTasksFromTaskManager()を呼び出し

#### 修正前のコード

```javascript
// ★ task-manager.htmlとの接続確認（データ連携Phase 1）
loadTaskManagerData();
```

#### 修正後のコード

```javascript
// ★ Phase 2-A: タスクマネージャーからタスク一覧を読み込み
initializeTasksFromTaskManager();
renderTaskList();  // タスク一覧を表示
```

### 修正3: storage eventリスナーの追加

**ファイル**: `flowchart-editor.html`
**場所**: 初期化処理（DOMContentLoaded等）
**修正内容**: A側のタスク更新を監視

#### 新規追加コード

```javascript
/**
 * タスクマネージャーの更新を監視
 * A側でタスク追加・削除・編集時にB側を自動更新
 */
window.addEventListener('storage', function(e) {
  // task-manager-update-trigger の変更を監視
  if (e.key === 'task-manager-update-trigger') {
    console.log('[FlowchartEditor] Task Manager data updated, reloading tasks...');

    // タスクリストを再読み込み
    initializeTasksFromTaskManager();
    renderTaskList();

    console.log('[FlowchartEditor] ✓ Task list updated:', mockTasks.length, 'tasks');
  }
});
```

### 修正4: renderTaskList()の拡張（オプション）

**ファイル**: `flowchart-editor.html`
**場所**: renderTaskList()関数
**修正内容**: 空のmermaid_idsでも適切に表示

#### 修正箇所の確認

現在のrenderTaskList()がmermaid_ids空のタスクを正しく表示できるか確認が必要。
もし表示されない場合は、以下の修正を追加:

```javascript
function renderTaskList() {
  const taskListContainer = document.getElementById('task-list');
  if (!taskListContainer) return;

  taskListContainer.innerHTML = '';

  mockTasks.forEach(task => {
    const taskItem = document.createElement('div');
    taskItem.className = 'task-item';

    // mermaid_ids空でも表示
    const hasLinks = task.mermaidIds && task.mermaidIds.trim() !== "";
    const linksIndicator = hasLinks ? '📌' : '⚪';  // リンクありなしを視覚化

    taskItem.innerHTML = `
      <div class="task-header">
        <span class="task-wbs">${linksIndicator} ${task.wbs}</span>
        <span class="task-status ${task.status.toLowerCase()}">${task.status}</span>
      </div>
      <div class="task-name">${task.taskName}</div>
      ${hasLinks ? `<div class="task-links">${task.mermaidIds}</div>` : ''}
    `;

    taskListContainer.appendChild(taskItem);
  });
}
```

---

## 📊 データフロー

### 現状（Phase 1修正完了時点）

```
A: task-manager.html                     B: flowchart-editor.html
┌────────────────────┐                   ┌────────────────────┐
│ タスク作成・編集    │                   │ mockTasks          │
│  ↓                 │                   │ (ハードコード)      │
│ saveToLocalStorage │─ localStorage ──→│ loadTaskManager    │
│                    │  (task-manager-  │ Data()             │
│                    │   data)          │  ↓                 │
│                    │                   │ 接続確認のみ        │
│                    │                   │ （表示はしない）    │
└────────────────────┘                   └────────────────────┘

問題: A側のタスクがB側に表示されない
```

### Phase 2-A実装後

```
A: task-manager.html                     B: flowchart-editor.html
┌────────────────────┐                   ┌────────────────────┐
│ タスク作成・編集    │                   │ SVG読み込み完了     │
│  ↓                 │                   │  ↓                 │
│ saveToLocalStorage │─ localStorage ──→│ initializeTasks    │
│  ↓                 │  (task-manager-  │ FromTaskManager()  │
│ localStorage.set   │   data)          │  ↓                 │
│ Item('task-       │                   │ mockTasks =        │
│ manager-update-   │                   │ convert(tasks)     │
│ trigger')         │                   │  ↓                 │
│                    │                   │ renderTaskList()   │
│                    │                   │  ↓                 │
│                    │  storage event ──→│ window.onStorage   │
│ タスク追加時        │                   │  ↓                 │
│  ↓                 │                   │ 自動再読み込み      │
│ update-trigger発火 │                   │ renderTaskList()   │
└────────────────────┘                   └────────────────────┘

解決: A側のタスクがB側に自動表示される
```

---

## 📊 データスキーマ

### A側のデータ構造（LocalStorage: task-manager-data）

```typescript
interface TaskManagerData {
  tasks: Array<{
    wbs_no: string;           // 例: "WBS1.1.0"
    task_name: string;        // 例: "要件定義"
    taskName?: string;        // 互換性のため
    mermaid_ids: string;      // 例: "node_1, node_2" または ""
    status: string;           // 例: "進行中", "完了", "未着手"
    start_date?: string;
    end_date?: string;
    flowchart?: {             // Phase 1で追加される属性
      nodeIds: string[];
      mermaidIds: string;
      memos: { [nodeId: string]: string };
      customLabels: { [nodeId: string]: string };
      relatedManualEdges: string[];
    };
  }>;
  meta: {
    projectName: string;
    created: string;
    lastModified: string;
  };
  timestamp: number;
}
```

### B側のデータ構造（mockTasks）

```typescript
interface MockTask {
  wbs: string;              // A側の wbs_no
  taskName: string;         // A側の task_name
  status: string;           // A側の status
  mermaidIds: string;       // A側の mermaid_ids（空でもOK）
  flowchartLinks: {
    nodes: Array<{
      id: string;
      label: string;
      highlightType: "primary" | "secondary";
    }>;
    edges: Array<{
      id: string;
      from: string;
      to: string;
      label: string;
      highlightType: "primary" | "secondary";
      highlightMode: "full" | "label-only";
    }>;
  };
}
```

### 変換ロジックの詳細

```javascript
function convertTaskToMockTask(task) {
  return {
    wbs: task.wbs_no,
    taskName: task.task_name || task.taskName || "（名前なし）",
    status: task.status || "Not Started",
    mermaidIds: task.mermaid_ids || "",
    flowchartLinks: parseMermaidIds(task.mermaid_ids || "")
  };
}
```

---

## 📅 実装計画

### 工数見積もり

| タスク | 工数 | 詳細 |
|--------|------|------|
| initializeTasksFromTaskManager()実装 | 30分 | データ変換ロジック |
| parseMermaidIds()実装 | 20分 | 文字列パース |
| storage eventリスナー追加 | 15分 | イベントハンドラ |
| renderTaskList()拡張（必要な場合） | 20分 | 空mermaid_ids対応 |
| SVG読み込み時の初期化処理修正 | 10分 | 関数呼び出し追加 |
| 動作確認・デバッグ | 30-45分 | 統合テスト |
| **合計** | **2-2.5時間** | |

### 実装順序

1. **parseMermaidIds()実装**
   - 単体テストしやすい関数から
   - mermaid_ids文字列 → flowchartLinks変換

2. **initializeTasksFromTaskManager()実装**
   - loadTaskManagerData()を使用
   - データ変換ロジック

3. **SVG読み込み時の初期化処理修正**
   - initializeTasksFromTaskManager()を呼び出し
   - renderTaskList()を呼び出し

4. **storage eventリスナー追加**
   - task-manager-update-trigger を監視
   - 自動再読み込み

5. **統合テスト**
   - 検証シナリオ実施

---

## 🧪 検証シナリオ

### シナリオ1: A側でタスク作成 → B側に自動表示

**前提条件**:
- task-manager.htmlが開かれていない、またはタスクが0個

**検証手順**:

1. **task-manager.htmlでタスク作成**
   - 新規タスクを作成（例: WBS1.1.0 "要件定義"）
   - mermaid_idsは空のまま
   - 自動保存される（LocalStorageに保存）

2. **flowchart-editor.htmlを開く**
   - SVGファイルをアップロード
   - DevToolsコンソールで確認:
     ```
     [FlowchartEditor] Initialized 1 tasks from Task Manager
     ```

3. **左サイドパネルを確認**
   - ✅ **期待値**: WBS1.1.0 "要件定義" が表示される
   - ✅ **期待値**: mermaid_ids空なので "⚪ WBS1.1.0" と表示（リンクなしアイコン）

4. **LocalStorageを確認**（DevTools → Application → Local Storage）
   - `task-manager-data` キーが存在
   - tasks配列に1個のタスクが含まれる

### シナリオ2: A側でタスク追加 → B側に自動反映

**前提条件**:
- シナリオ1完了状態
- flowchart-editor.html が開いたまま

**検証手順**:

1. **task-manager.htmlでタスク追加**
   - 新規タスク作成（例: WBS1.2.0 "基本設計"）
   - 自動保存される

2. **flowchart-editor.htmlで確認**
   - コンソールログ:
     ```
     [FlowchartEditor] Task Manager data updated, reloading tasks...
     [FlowchartEditor] ✓ Task list updated: 2 tasks
     ```
   - ✅ **期待値**: 左サイドパネルに自動的にWBS1.2.0が追加表示される

### シナリオ3: B側でmermaid_ids割当 → A側へ反映

**前提条件**:
- シナリオ2完了状態
- タスク: WBS1.1.0, WBS1.2.0

**検証手順**:

1. **flowchart-editor.htmlでタスク編集**
   - WBS1.1.0の「編集」ボタンをクリック
   - ノードセレクタで "F_01" を選択
   - 「追加」ボタンクリック
   - 「保存」ボタンクリック

2. **左サイドパネルで視覚変化を確認**
   - ✅ **期待値**: "⚪ WBS1.1.0" → "📌 WBS1.1.0" に変化（リンクありアイコン）

3. **Ctrl+Sで保存**
   - コンソールログ:
     ```
     [FlowchartEditor] Published flowchart attributes for 2 tasks
     ```

4. **task-manager.htmlで確認**
   - コンソールログ:
     ```
     [TaskManager] Updated mermaid_ids for WBS1.1.0: "" → "F_01"
     ```
   - タスク詳細モーダルを開く
   - ✅ **期待値**: Mermaid ノードIDフィールドに "F_01" が表示される

### シナリオ4: A側でタスク削除 → B側から消える

**前提条件**:
- シナリオ3完了状態

**検証手順**:

1. **task-manager.htmlでタスク削除**
   - WBS1.2.0を削除
   - 自動保存される

2. **flowchart-editor.htmlで確認**
   - ✅ **期待値**: 左サイドパネルからWBS1.2.0が自動的に消える
   - ✅ **期待値**: WBS1.1.0のみ表示される

---

## ⚠️ リスクと対策

### リスク1: データ構造の不一致

**問題**: A側のタスクデータ構造が想定と異なる（task_name vs taskName など）

**影響度**: 中

**対策**:
- convertTaskToMockTask()で両フィールドをサポート
- `task.task_name || task.taskName || "（名前なし）"`

### リスク2: storage eventの発火タイミング

**問題**: 同一ウィンドウではstorage eventが発火しない

**影響度**: 低

**理由**: A（task-manager.html）とB（flowchart-editor.html）は別タブ/ウィンドウで開くため問題なし

**対策**: ドキュメントに「別タブで開く」と明記

### リスク3: parseMermaidIds()の簡易実装

**問題**: エッジの詳細情報（from, to, label）が取得できない

**影響度**: 低

**理由**: flowchartLinks.edges はハイライト表示にのみ使用され、現時点では詳細情報不要

**対策**: Phase 2-B（双方向ハイライト）で必要になれば拡張

### リスク4: タスク数が多い場合のパフォーマンス

**問題**: 100個以上のタスクで描画が遅い可能性

**影響度**: 低

**理由**: 一般的なプロジェクトではタスク数は50個以下

**対策**:
- 仮想スクロール（Virtual Scroll）は現時点では不要
- Phase 2-B以降でパフォーマンス最適化を検討

### リスク5: 既存のmockTasks依存コード

**問題**: mockTasksを参照している他の関数が動作しなくなる可能性

**影響度**: 中

**対策**:
- mockTasksの構造を維持（互換性確保）
- 既存のタスク編集モーダル、renderTaskList()などは変更不要

---

## 🔗 関連ドキュメント

### 前提条件

- [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) - Phase 1修正（必須完了）
- [phase1-implementation-summary.md](./phase1-implementation-summary.md) - Phase 1基本実装

### 全体計画

- [future-roadmap.md](./future-roadmap.md) - Phase 0-3の全体ロードマップ
- [README.md](./README.md) - データ連携機能概要

### 次のステップ

- Phase 2-B: 双方向ハイライト（ノード↔タスク）
- Phase 2-C: タスク一覧パネルUI改善

---

## 📝 実装チェックリスト

実装時に以下をチェックしてください:

### コード実装

- [ ] `parseMermaidIds(mermaidIds)` 関数実装
- [ ] `initializeTasksFromTaskManager()` 関数実装
- [ ] SVG読み込み時に `initializeTasksFromTaskManager()` 呼び出し
- [ ] SVG読み込み時に `renderTaskList()` 呼び出し
- [ ] storage eventリスナー追加（task-manager-update-trigger）
- [ ] `renderTaskList()` で空mermaid_ids対応（必要な場合）

### 動作確認

- [ ] シナリオ1: A側でタスク作成 → B側に自動表示
- [ ] シナリオ2: A側でタスク追加 → B側に自動反映
- [ ] シナリオ3: B側でmermaid_ids割当 → A側へ反映
- [ ] シナリオ4: A側でタスク削除 → B側から消える

### コンソールログ

- [ ] "Initialized X tasks from Task Manager" が表示される
- [ ] "Task Manager data updated, reloading tasks..." が表示される
- [ ] "Task list updated: X tasks" が表示される
- [ ] エラーログがない

### LocalStorage確認

- [ ] `task-manager-data` が存在
- [ ] `task-manager-update-trigger` が更新される
- [ ] `flowchart-attributes` が送信される（Phase 1修正）

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成 - Phase 2-A実装仕様策定 |

---

**最終更新**: 2026-03-22
**ステータス**: 仕様確定、実装待ち
**次のアクション**: flowchart-editor.htmlの修正開始
**前提条件**: Phase 1修正（mermaid_ids双方向同期）が完了していること
