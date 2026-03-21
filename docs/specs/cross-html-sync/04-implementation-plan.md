# 実装計画 - クロスHTML連携

---
**feature**: クロスHTML連携
**document**: 実装計画
**version**: 1.0
**status**: ready
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

LocalStorage + storage イベントを使用した独立HTML間のリアルタイム同期機能の実装計画。

**重要**: 本計画は以下の仕様書に基づいています:
- [README.md](./README.md) - 全体概要、データ永続化戦略
- [01-localstorage-design.md](./01-localstorage-design.md) - LocalStorage設計、容量管理
- [02-data-structure.md](./02-data-structure.md) - データ構造定義
- [03-flowchart-integration.md](./03-flowchart-integration.md) - UI/UX設計

---

## 実装の全体構造

### 三層アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│ レイヤー1: UI/UXレイヤー                         │
│ - タスク編集UI                                  │
│ - フローチャート編集UI                           │
│ - 同期状態表示                                  │
└─────────────┬───────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────┐
│ レイヤー2: 同期レイヤー（LocalStorage）          │
│ - リアルタイム通信バッファ                       │
│ - storage イベントハンドリング                   │
│ - 容量管理・エラーハンドリング                   │
└─────────────┬───────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────┐
│ レイヤー3: 永続化レイヤー（File System）         │
│ - 本体ファイル自動保存（3秒デバウンス）          │
│ - File System Access API                       │
│ - マスターデータ管理                            │
└─────────────────────────────────────────────────┘
```

---

## 実装フェーズ

### Phase 0: 前提条件の確認（1h）

#### 既存機能の確認

**task-manager.html 側**:
- ✅ LocalStorage保存機能 (`saveToLocalStorage()`) の存在確認
- ✅ タスクデータ構造 (`projectData`) の確認
- ✅ File System Access API実装状況の確認

**flowchart-editor.html 側**:
- ✅ 現在のSVG描画機能の確認
- ✅ ノード構造・ID体系の確認
- ✅ 既存のデータ保存機能の確認

#### 実施内容

```bash
# 1. task-manager.htmlの既存機能確認
grep -n "saveToLocalStorage" task-manager.html
grep -n "localStorage.setItem" task-manager.html
grep -n "showSaveFilePicker\|showOpenFilePicker" task-manager.html

# 2. flowchart-editor.htmlの確認
grep -n "svg" docs/ui/flowchart-editor.html
grep -n "node-" docs/ui/flowchart-editor.html
```

**成功基準**:
- 既存のLocalStorage実装を理解
- File System Access API実装状況を把握
- 既存機能との統合ポイントを特定

---

### Phase 1: 基本連携機能（6-8h）

#### 1-1. task-manager.html 側の実装（3-4h）

##### 実装項目1: LocalStorage保存の強化

**目的**: flowchart属性を含むデータを保存

**実装箇所**: `saveToLocalStorage()` 関数

```javascript
// 既存実装の拡張
function saveToLocalStorage() {
  const data = {
    tasks: projectData.tasks,  // flowchart属性を含む
    meta: projectData.meta,
    timestamp: Date.now()
  };

  const jsonStr = JSON.stringify(data);
  const sizeKB = new Blob([jsonStr]).size / 1024;
  const sizeMB = sizeKB / 1024;

  // 🆕 容量警告（3MB超過時）
  if (sizeKB > 3072) {
    console.warn(`[LocalStorage] データサイズ: ${sizeKB.toFixed(0)}KB`);

    const proceed = confirm(
      `タスクデータが大きくなっています (${sizeMB.toFixed(1)}MB)。\n` +
      `保存を続けますか？\n\n` +
      `対処法:\n` +
      `1. 完了タスクをアーカイブ\n` +
      `2. 不要なコメントを削除\n` +
      `3. JSONファイルへエクスポート`
    );

    if (!proceed) return false;
  }

  try {
    localStorage.setItem('task-manager-data', jsonStr);
    // 🆕 更新トリガーを発火
    localStorage.setItem('task-manager-update-trigger', Date.now().toString());

    console.log(`[TaskManager] データ保存成功 (${sizeKB.toFixed(0)}KB)`);
    return true;

  } catch (e) {
    if (e.name === 'QuotaExceededError') {
      alert(
        `❌ LocalStorage容量不足でデータを保存できませんでした。\n\n` +
        `現在のデータサイズ: ${sizeMB.toFixed(1)}MB\n` +
        `ブラウザ上限: 5MB\n\n` +
        `対処法:\n` +
        `1. 完了タスクをアーカイブ（推奨）\n` +
        `2. 不要なコメント・履歴を削除\n` +
        `3. JSONファイルへエクスポートして別管理\n` +
        `4. ブラウザの開発者ツールでLocalStorageをクリア`
      );
      return false;
    }
    throw e;
  }
}
```

**テスト**:
```javascript
// コンソールで実行
saveToLocalStorage();
console.log(localStorage.getItem('task-manager-data'));
console.log(localStorage.getItem('task-manager-update-trigger'));
```

##### 実装項目2: storage イベントハンドラ追加

**目的**: flowchart側からの更新を検知

**実装箇所**: グローバルスコープ（DOMContentLoaded内）

```javascript
// 🆕 追加: フローチャート側からの更新を検知
window.addEventListener('storage', handleFlowchartUpdate);

function handleFlowchartUpdate(e) {
  // フローチャート更新トリガーのみを処理
  if (e.key !== 'flowchart-update-trigger') return;

  console.log('[TaskManager] Flowchart updated, reloading attributes...');

  try {
    // フローチャート属性を取得
    const attrsStr = localStorage.getItem('flowchart-attributes');
    if (!attrsStr) return;

    const attrs = JSON.parse(attrsStr);

    // タスクにフローチャート属性を統合
    let updatedCount = 0;
    projectData.tasks.forEach(task => {
      if (attrs[task.wbs_no]) {
        task.flowchart = attrs[task.wbs_no];
        updatedCount++;
        console.log(`[TaskManager] Updated flowchart info for ${task.wbs_no}`);
      }
    });

    if (updatedCount > 0) {
      // UI更新
      renderTaskTable();

      // 変更を永続化（flowchart属性を含めて保存）
      saveToLocalStorage();

      // 🆕 本体ファイルへも自動保存
      autoSaveToFile();

      showNotification(`✓ ${updatedCount}個のタスクにフローチャート情報を統合しました`);
    }

  } catch (error) {
    console.error('[TaskManager] Failed to parse flowchart attributes:', error);
  }
}
```

**テスト**:
```javascript
// 別タブで実行してstorageイベントをシミュレート
localStorage.setItem('flowchart-update-trigger', Date.now().toString());
```

##### 実装項目3: 初期化時のフローチャート属性読み込み

**目的**: ページ読み込み時に既存のflowchart属性を統合

**実装箇所**: `loadFromLocalStorage()` または初期化関数

```javascript
// 🆕 追加: ページ読み込み時にフローチャート属性を統合
function loadFlowchartAttributes() {
  const attrsStr = localStorage.getItem('flowchart-attributes');
  if (!attrsStr) return;

  try {
    const attrs = JSON.parse(attrsStr);

    // タスクに統合
    projectData.tasks.forEach(task => {
      if (attrs[task.wbs_no]) {
        task.flowchart = attrs[task.wbs_no];
      }
    });

    console.log('[TaskManager] Flowchart attributes loaded');
  } catch (error) {
    console.error('[TaskManager] Failed to load flowchart attributes:', error);
  }
}

// DOMContentLoaded で実行
window.addEventListener('DOMContentLoaded', () => {
  // 既存の初期化処理
  loadFromLocalStorage();

  // 🆕 フローチャート属性の読み込み
  loadFlowchartAttributes();

  renderTaskTable();
});
```

##### 実装項目4: 本体ファイル自動保存（File System Access API）

**目的**: LocalStorageとは別に永続保存

**実装箇所**: グローバルスコープ

```javascript
// 🆕 追加: 本体ファイル自動保存
let autoSaveTimeout = null;
let currentFileHandle = null; // File System Access APIのハンドル

function autoSaveToFile() {
  // デバウンス: 3秒間編集がなければ保存
  if (autoSaveTimeout) clearTimeout(autoSaveTimeout);

  autoSaveTimeout = setTimeout(async () => {
    await saveToMasterFile();
  }, 3000);
}

async function saveToMasterFile() {
  if (!currentFileHandle) {
    console.warn('[AutoSave] ファイルハンドルがありません。手動保存してください。');
    return;
  }

  try {
    const writable = await currentFileHandle.createWritable();
    const data = {
      tasks: projectData.tasks,  // flowchart属性を含む
      meta: projectData.meta,
      timestamp: Date.now()
    };

    await writable.write(JSON.stringify(data, null, 2));
    await writable.close();

    const sizeKB = (JSON.stringify(data).length / 1024).toFixed(0);
    console.log(`✅ 本体ファイルへ自動保存完了 (${sizeKB}KB)`);

    // 🆕 最終保存時刻を表示（UI更新）
    updateLastSavedTime();

  } catch (error) {
    console.error('❌ 本体ファイル保存エラー:', error);
    alert('ファイル保存に失敗しました。手動で保存してください。');
  }
}

// 既存の保存処理を拡張
function onTaskEdit() {
  markDirty();
  saveToLocalStorage();  // リアルタイム同期用（即時）
  autoSaveToFile();      // 永続保存用（3秒デバウンス）
}
```

**統合ポイント**:
```javascript
// 既存の以下の関数に autoSaveToFile() を追加
- changeStatus()
- changePriority()
- updateDate()
- updateTaskDates() (ガントチャート用)
- その他のタスク編集関数
```

**テスト**:
```javascript
// 1. ファイルを開く
const [fileHandle] = await window.showOpenFilePicker();
currentFileHandle = fileHandle;

// 2. タスクを編集
changeStatus('WBS1.1.0', '進行中');

// 3. 3秒待機後、ファイルが自動保存されることを確認
```

#### 1-2. flowchart-editor.html 側の実装（3-4h）

##### 実装項目1: タスクデータ読み込み機能

**目的**: task-manager側のタスクデータをLocalStorageから読み込み

**実装箇所**: グローバルスコープ

```javascript
// 🆕 追加: タスクデータを LocalStorage から読み込み
let taskData = null;

function loadTaskData() {
  try {
    const dataStr = localStorage.getItem('task-manager-data');
    if (!dataStr) {
      console.warn('[Flowchart] No task data found in LocalStorage');
      updateSyncStatus('no-data');
      return null;
    }

    taskData = JSON.parse(dataStr);
    console.log('[Flowchart] Loaded', taskData.tasks.length, 'tasks');

    updateSyncStatus('synced');
    return taskData;

  } catch (error) {
    console.error('[Flowchart] Failed to load task data:', error);
    updateSyncStatus('error');
    return null;
  }
}

// ページ読み込み時に実行
window.addEventListener('DOMContentLoaded', () => {
  loadTaskData();
  renderFlowchart();
  renderTaskPanel(); // 🆕 タスクパネルを表示
});
```

##### 実装項目2: storage イベントハンドラ

**目的**: task-manager側の更新を検知してフローチャートを更新

**実装箇所**: グローバルスコープ

```javascript
// 🆕 追加: タスクマネージャー側の更新を検知
window.addEventListener('storage', handleTaskManagerUpdate);

function handleTaskManagerUpdate(e) {
  // タスクマネージャー更新トリガーのみを処理
  if (e.key !== 'task-manager-update-trigger') return;

  console.log('[Flowchart] Task data updated, reloading...');
  updateSyncStatus('syncing');

  // 少し遅延させてアニメーションを見せる
  setTimeout(() => {
    const newData = loadTaskData();
    if (newData) {
      taskData = newData;

      // フローチャートを再描画
      renderFlowchart();

      // 関連付けられたノードを更新
      updateTaskNodes();

      updateSyncStatus('synced');
      showNotification('✓ タスクデータを同期しました');
    }
  }, 300);
}

function updateTaskNodes() {
  // タスクに関連付けられたノードを更新
  if (!taskData || !taskData.tasks) return;

  taskData.tasks.forEach(task => {
    if (task.flowchart && task.flowchart.nodeId) {
      const node = document.getElementById(task.flowchart.nodeId);
      if (node) {
        // ノードのラベルを更新
        updateNodeLabel(node, task);

        // ステータスに応じて色を変更
        updateNodeColor(node, task.status);
      }
    }
  });
}
```

##### 実装項目3: フローチャート属性保存機能

**目的**: ノードとタスクの関連付けをLocalStorageに保存

**実装箇所**: グローバルスコープ

```javascript
// 🆕 追加: ノードとタスクの関連付けを保存
function saveFlowchartAttribute(wbsNo, nodeId, nodeType, position) {
  try {
    // 既存の属性を取得
    const attrsStr = localStorage.getItem('flowchart-attributes') || '{}';
    const attrs = JSON.parse(attrsStr);

    // 新しい属性を追加/更新
    attrs[wbsNo] = {
      nodeId: nodeId,
      nodeType: nodeType,
      position: position,
      timestamp: Date.now()
    };

    // 保存
    localStorage.setItem('flowchart-attributes', JSON.stringify(attrs));

    // 更新トリガーを発火
    localStorage.setItem('flowchart-update-trigger', Date.now().toString());

    console.log(`[Flowchart] Saved attribute for ${wbsNo} → ${nodeId}`);
    showNotification(`✓ タスク ${wbsNo} を関連付けました`);

  } catch (error) {
    if (error.name === 'QuotaExceededError') {
      alert('⚠ LocalStorage容量が不足しています。');
    } else {
      console.error('[Flowchart] Failed to save attribute:', error);
    }
  }
}
```

##### 実装項目4: タスクパネル（サイドバー）UI

**目的**: タスク一覧を表示し、ドラッグ&ドロップでノードに関連付け

**実装箇所**: HTML + JavaScript

```html
<!-- 🆕 追加: タスクパネル -->
<div id="task-panel" class="task-panel">
  <div class="panel-header">
    <h3>タスク一覧</h3>
    <div id="sync-indicator" class="sync-indicator">
      🟢 同期済み
    </div>
  </div>
  <div id="task-list" class="task-list">
    <!-- JavaScriptで動的生成 -->
  </div>
  <div class="panel-footer">
    <button onclick="loadTaskData()">手動同期</button>
    <button onclick="toggleTaskPanel()">表示/非表示</button>
  </div>
</div>

<style>
.task-panel {
  position: fixed;
  left: 0;
  top: 0;
  width: 300px;
  height: 100%;
  background: white;
  border-right: 1px solid #ccc;
  overflow-y: auto;
  box-shadow: 2px 0 5px rgba(0,0,0,0.1);
}

.panel-header {
  padding: 15px;
  border-bottom: 1px solid #eee;
}

.sync-indicator {
  font-size: 12px;
  margin-top: 5px;
}

.task-list {
  padding: 10px;
}

.task-item {
  padding: 10px;
  margin: 5px 0;
  background: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: move;
}

.task-item:hover {
  background: #f0f0f0;
}

.task-item.dragging {
  opacity: 0.5;
}
</style>
```

```javascript
// 🆕 追加: タスクパネル描画
function renderTaskPanel() {
  if (!taskData || !taskData.tasks) {
    showEmptyState();
    return;
  }

  const taskList = document.getElementById('task-list');
  taskList.innerHTML = '';

  taskData.tasks.forEach(task => {
    const taskItem = document.createElement('div');
    taskItem.className = 'task-item';
    taskItem.draggable = true;
    taskItem.dataset.wbsNo = task.wbs_no;

    const statusIcon = getStatusIcon(task.status);
    const linkedInfo = task.flowchart
      ? `<div class="linked-info">🔵 ${task.flowchart.nodeId}</div>`
      : '';

    taskItem.innerHTML = `
      <div class="task-wbs">${task.wbs_no}</div>
      <div class="task-name">${task.task_name}</div>
      <div class="task-meta">
        📅 ${task.start_date} ~ ${task.finish_date}<br>
        ${statusIcon} ${task.status}
      </div>
      ${linkedInfo}
    `;

    // ドラッグイベント
    makeTaskDraggable(taskItem, task);

    taskList.appendChild(taskItem);
  });
}

function getStatusIcon(status) {
  const icons = {
    '未着手': '⚪',
    '進行中': '🟢',
    '完了': '🔵',
    '保留': '🟡',
    '中止': '🔴'
  };
  return icons[status] || '⚪';
}

function showEmptyState() {
  const panel = document.getElementById('task-panel');
  panel.innerHTML = `
    <div class="empty-state">
      <p>タスクデータが見つかりません</p>
      <p>task-manager.html を開いてタスクを作成してください</p>
      <button onclick="loadTaskData()">再読み込み</button>
    </div>
  `;
}
```

##### 実装項目5: ドラッグ&ドロップ実装

**目的**: タスクをノードにドラッグして関連付け

**実装箇所**: グローバルスコープ

```javascript
// 🆕 追加: タスク要素をドラッグ可能にする
function makeTaskDraggable(taskElement, task) {
  taskElement.addEventListener('dragstart', (e) => {
    e.dataTransfer.setData('text/plain', task.wbs_no);
    e.dataTransfer.effectAllowed = 'link';
    taskElement.classList.add('dragging');
  });

  taskElement.addEventListener('dragend', (e) => {
    taskElement.classList.remove('dragging');
  });
}

// 🆕 追加: ノードをドロップターゲットにする
function makeNodeDroppable(nodeElement) {
  nodeElement.addEventListener('dragover', (e) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'link';
    nodeElement.classList.add('drop-target');
  });

  nodeElement.addEventListener('dragleave', (e) => {
    nodeElement.classList.remove('drop-target');
  });

  nodeElement.addEventListener('drop', (e) => {
    e.preventDefault();
    nodeElement.classList.remove('drop-target');

    const wbsNo = e.dataTransfer.getData('text/plain');
    const task = taskData.tasks.find(t => t.wbs_no === wbsNo);

    if (task) {
      confirmTaskNodeLink(task, nodeElement);
    }
  });
}

// 🆕 追加: 関連付け確認
function confirmTaskNodeLink(task, nodeElement) {
  const nodeId = nodeElement.id;
  const nodeType = nodeElement.dataset.type || 'process';

  const confirmed = confirm(
    `タスク: ${task.wbs_no} ${task.task_name}\n` +
    `ノード: ${nodeId} (${nodeType})\n\n` +
    `このタスクをこのノードに関連付けますか？`
  );

  if (confirmed) {
    const position = getNodePosition(nodeElement);
    saveFlowchartAttribute(task.wbs_no, nodeId, nodeType, position);

    // ノード表示を更新
    updateNodeLabel(nodeElement, task);

    showNotification(`✓ タスク ${task.wbs_no} を関連付けました`);
  }
}

function getNodePosition(nodeElement) {
  const rect = nodeElement.getBoundingClientRect();
  return {
    x: rect.left + rect.width / 2,
    y: rect.top + rect.height / 2
  };
}

function updateNodeLabel(nodeElement, task) {
  const label = nodeElement.querySelector('.node-label');
  if (label) {
    label.innerHTML = `
      <tspan x="0" dy="0">${task.wbs_no}</tspan>
      <tspan x="0" dy="1.2em">${task.task_name}</tspan>
      <tspan x="0" dy="1.2em">${getStatusIcon(task.status)} ${task.status}</tspan>
    `;
  }

  updateNodeColor(nodeElement, task.status);
}

function updateNodeColor(nodeElement, status) {
  const colors = {
    '未着手': '#cccccc',
    '進行中': '#4caf50',
    '完了': '#2196f3',
    '保留': '#ff9800',
    '中止': '#f44336'
  };

  const color = colors[status] || '#cccccc';
  nodeElement.style.fill = color;
}
```

##### 実装項目6: 同期状態インジケーター

**目的**: 同期状態を視覚的に表示

**実装箇所**: グローバルスコープ

```javascript
// 🆕 追加: 同期状態を表示
function updateSyncStatus(status) {
  const indicator = document.getElementById('sync-indicator');
  if (!indicator) return;

  switch (status) {
    case 'synced':
      indicator.innerHTML = '🟢 同期済み';
      indicator.className = 'sync-indicator status-ok';
      break;

    case 'syncing':
      indicator.innerHTML = '🔄 同期中...';
      indicator.className = 'sync-indicator status-syncing';
      break;

    case 'error':
      indicator.innerHTML = '🔴 同期エラー';
      indicator.className = 'sync-indicator status-error';
      break;

    case 'no-data':
      indicator.innerHTML = '⚪ タスクデータなし';
      indicator.className = 'sync-indicator status-warning';
      break;
  }
}

// 🆕 追加: トースト通知
function showNotification(message, type = 'info') {
  const toast = document.createElement('div');
  toast.className = `toast toast-${type}`;
  toast.textContent = message;

  document.body.appendChild(toast);

  setTimeout(() => toast.classList.add('show'), 10);

  setTimeout(() => {
    toast.classList.remove('show');
    setTimeout(() => toast.remove(), 300);
  }, 3000);
}
```

```css
/* 🆕 追加: トーストスタイル */
.toast {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 12px 20px;
  background: #333;
  color: white;
  border-radius: 4px;
  opacity: 0;
  transform: translateY(20px);
  transition: all 0.3s ease;
  z-index: 10000;
}

.toast.show {
  opacity: 1;
  transform: translateY(0);
}

.toast.toast-success { background: #4caf50; }
.toast.toast-warning { background: #ff9800; }
.toast.toast-error { background: #f44336; }
```

---

### Phase 2: エラーハンドリング（2-3h）

#### 2-1. LocalStorage容量管理（1h）

##### 実装項目1: 容量超過時のアーカイブ機能

**実装箇所**: task-manager.html

```javascript
// 🆕 追加: 完了タスクのアーカイブ
function archiveCompletedTasks() {
  const activeTasks = projectData.tasks.filter(t => t.status !== '完了');
  const completedTasks = projectData.tasks.filter(t => t.status === '完了');

  if (completedTasks.length === 0) {
    alert('アーカイブ可能な完了タスクがありません。');
    return;
  }

  // 完了タスクをJSONファイルとしてダウンロード
  const archiveData = {
    tasks: completedTasks,
    meta: {
      project_name: projectData.meta.project_name,
      archived_at: new Date().toISOString(),
      task_count: completedTasks.length
    }
  };

  const blob = new Blob([JSON.stringify(archiveData, null, 2)], {
    type: 'application/json'
  });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `archived-tasks-${Date.now()}.json`;
  a.click();
  URL.revokeObjectURL(url);

  // LocalStorageには進行中タスクのみ保持
  projectData.tasks = activeTasks;
  saveToLocalStorage();
  autoSaveToFile();

  alert(
    `✓ ${completedTasks.length}個の完了タスクをアーカイブしました。\n` +
    `ファイル: archived-tasks-${Date.now()}.json`
  );

  renderTaskTable();
}

// UIボタン追加
// <button onclick="archiveCompletedTasks()">完了タスクをアーカイブ</button>
```

##### 実装項目2: 容量状態の可視化

**実装箇所**: task-manager.html

```javascript
// 🆕 追加: LocalStorage使用状況を表示
function showStorageStatus() {
  let totalSize = 0;
  const details = [];

  for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    const value = localStorage.getItem(key);
    const size = new Blob([value]).size;
    totalSize += size;

    details.push({
      key: key,
      size: size,
      sizeKB: (size / 1024).toFixed(1)
    });
  }

  details.sort((a, b) => b.size - a.size);

  const totalKB = totalSize / 1024;
  const totalMB = totalKB / 1024;
  const usagePercent = (totalMB / 5 * 100).toFixed(1);

  console.log('=== LocalStorage 使用状況 ===');
  console.log(`合計: ${totalKB.toFixed(0)}KB (${totalMB.toFixed(2)}MB)`);
  console.log(`使用率: ${usagePercent}% / 5MB`);
  console.log('');
  console.log('キー別内訳:');
  details.forEach(d => {
    console.log(`  ${d.key}: ${d.sizeKB}KB`);
  });

  return {
    totalKB: totalKB,
    totalMB: totalMB,
    usagePercent: parseFloat(usagePercent),
    details: details
  };
}

// UIボタン追加
// <button onclick="showStorageStatus()">容量確認</button>
```

#### 2-2. エラーハンドリング強化（1-2h）

##### 実装項目1: JSONパースエラー処理

**実装箇所**: 両HTML

```javascript
// 🆕 追加: 安全なJSONパース
function safeJSONParse(jsonString, defaultValue = null) {
  try {
    return JSON.parse(jsonString);
  } catch (error) {
    console.error('[JSON] Parse error:', error);
    console.error('[JSON] Invalid data:', jsonString?.substring(0, 100) + '...');
    return defaultValue;
  }
}

// 使用例（既存のJSON.parse()を置き換え）
function loadTaskData() {
  const dataStr = localStorage.getItem('task-manager-data');
  if (!dataStr) return null;

  const data = safeJSONParse(dataStr);
  if (!data || !data.tasks) {
    alert('⚠ タスクデータが破損しています。LocalStorageをクリアしてください。');
    return null;
  }

  return data;
}
```

##### 実装項目2: タイムスタンプ競合解決

**実装箇所**: 両HTML

```javascript
// 🆕 追加: フローチャート属性のマージ
function mergeFlowchartAttributes(localAttrs, remoteAttrs) {
  const merged = { ...localAttrs };

  for (const wbsNo in remoteAttrs) {
    const remote = remoteAttrs[wbsNo];
    const local = localAttrs[wbsNo];

    if (!local) {
      // ローカルになければ追加
      merged[wbsNo] = remote;
    } else {
      // タイムスタンプで比較
      if (remote.timestamp > local.timestamp) {
        merged[wbsNo] = remote;
        console.log(`[Merge] Using remote data for ${wbsNo}`);
      } else {
        console.log(`[Merge] Keeping local data for ${wbsNo}`);
      }
    }
  }

  return merged;
}
```

---

### Phase 3: UX改善（3-4h）

#### 3-1. 同期状態の可視化（1-2h）

##### 実装項目1: 最終保存時刻の表示

**実装箇所**: task-manager.html

```javascript
// 🆕 追加: 最終保存時刻を表示
function updateLastSavedTime() {
  const now = new Date();
  const timeStr = now.toLocaleTimeString('ja-JP');

  const indicator = document.getElementById('last-saved-time');
  if (indicator) {
    indicator.textContent = `最終保存: ${timeStr}`;
  }
}
```

```html
<!-- UIに追加 -->
<div id="last-saved-time" class="last-saved-time">
  最終保存: --:--:--
</div>
```

##### 実装項目2: 未保存変更の警告

**実装箇所**: task-manager.html

```javascript
// 🆕 追加: 未保存変更の警告
let hasUnsavedChanges = false;

function markDirty() {
  hasUnsavedChanges = true;
  updateDirtyIndicator();
}

function markClean() {
  hasUnsavedChanges = false;
  updateDirtyIndicator();
}

function updateDirtyIndicator() {
  const indicator = document.getElementById('dirty-indicator');
  if (indicator) {
    indicator.style.display = hasUnsavedChanges ? 'inline' : 'none';
  }
}

// ページ離脱時の警告
window.addEventListener('beforeunload', (e) => {
  if (hasUnsavedChanges) {
    e.preventDefault();
    e.returnValue = '未保存の変更があります。ページを離れますか？';
    return e.returnValue;
  }
});

// 保存成功時にクリア
async function saveToMasterFile() {
  // ... 既存の処理

  markClean(); // 🆕 保存成功後にクリア
}
```

```html
<!-- UIに追加 -->
<span id="dirty-indicator" class="dirty-indicator" style="display: none;">
  ● 未保存
</span>
```

#### 3-2. 手動リフレッシュ機能（30分）

##### 実装項目1: 手動同期ボタン

**実装箇所**: flowchart-editor.html

```javascript
// 🆕 追加: 手動同期
function manualSync() {
  updateSyncStatus('syncing');

  setTimeout(() => {
    const data = loadTaskData();
    if (data) {
      taskData = data;
      renderFlowchart();
      renderTaskPanel();
      updateTaskNodes();
      updateSyncStatus('synced');
      showNotification('✓ 手動同期完了', 'success');
    } else {
      updateSyncStatus('error');
      showNotification('✗ 同期に失敗しました', 'error');
    }
  }, 500);
}
```

```html
<!-- UIに追加 -->
<button onclick="manualSync()">🔄 手動同期</button>
```

#### 3-3. 接続状態インジケーター（1-2h）

##### 実装項目1: リアルタイム接続状態の表示

**実装箇所**: flowchart-editor.html

```javascript
// 🆕 追加: 接続状態の監視
function startConnectionMonitor() {
  setInterval(() => {
    const trigger = localStorage.getItem('task-manager-update-trigger');

    if (trigger) {
      const lastUpdate = parseInt(trigger);
      const now = Date.now();
      const diffMinutes = (now - lastUpdate) / 1000 / 60;

      if (diffMinutes > 5) {
        updateSyncStatus('warning');
      } else {
        updateSyncStatus('synced');
      }
    } else {
      updateSyncStatus('no-data');
    }
  }, 10000); // 10秒ごとにチェック
}

// ページ読み込み時に開始
window.addEventListener('DOMContentLoaded', () => {
  loadTaskData();
  renderFlowchart();
  renderTaskPanel();
  startConnectionMonitor(); // 🆕 監視開始
});
```

---

## テスト計画

### 単体テスト

| 機能 | テストケース | 期待結果 |
|------|------------|---------|
| `saveToLocalStorage()` | 正常保存 | データ保存成功 |
| `saveToLocalStorage()` | 容量超過 | QuotaExceededErrorキャッチ |
| `loadTaskData()` | データ存在 | タスクデータ読み込み成功 |
| `loadTaskData()` | データなし | null返却、エラーなし |
| `loadTaskData()` | 破損データ | エラーメッセージ表示 |
| `saveFlowchartAttribute()` | 新規追加 | 属性保存成功 |
| `saveFlowchartAttribute()` | 上書き | タイムスタンプ更新 |
| `handleFlowchartUpdate()` | storage イベント | タスク更新成功 |

### 統合テスト

| シナリオ | 確認項目 |
|---------|---------|
| タスク編集 → フローチャート更新 | storage イベント発火、SVG更新 |
| ノード関連付け → タスク更新 | flowchart属性追加、UI反映 |
| 両方同時編集 | 競合解決、最新データ採用 |
| ブラウザリロード | データ永続化、属性復元 |
| LocalStorage容量超過 | エラー表示、アーカイブ提案 |
| 本体ファイル自動保存 | 3秒後に保存、エラーハンドリング |

### E2Eテストシナリオ

#### シナリオ1: 初回使用フロー

```
1. task-manager.htmlを開く
2. 新規プロジェクトを作成
3. タスクを3つ追加
4. 本体ファイルへ保存
5. flowchart-editor.htmlを別タブで開く
6. タスクパネルに3つのタスクが表示されることを確認
7. タスクをノードにドラッグ&ドロップ
8. 関連付け確認ダイアログでOK
9. task-manager.htmlに戻る
10. タスクにフローチャート情報が追加されていることを確認
```

#### シナリオ2: LocalStorage容量超過フロー

```
1. 大量タスク（5000個）を含むプロジェクトを開く
2. タスクを編集
3. LocalStorage容量警告が表示されることを確認
4. アーカイブボタンをクリック
5. 完了タスクがJSONファイルとしてダウンロードされる
6. LocalStorageの空き容量が増えることを確認
7. 本体ファイルには全データが保存されていることを確認
```

#### シナリオ3: 本体ファイル復旧フロー

```
1. LocalStorageをクリア（開発者ツール）
2. task-manager.htmlをリロード
3. 本体ファイルから開く
4. すべてのタスクが復元されることを確認
5. フローチャート属性も復元されることを確認
6. LocalStorageへ再展開されることを確認
```

---

## 実装優先順位

### 🔴 必須（Phase 1）

1. **task-manager.html**: storage イベントハンドラ追加
2. **task-manager.html**: 本体ファイル自動保存
3. **flowchart**: タスクデータ読み込み
4. **flowchart**: storage イベントハンドラ
5. **flowchart**: フローチャート属性保存

### 🟡 高優先度（Phase 2）

6. **両方**: LocalStorage容量管理
7. **両方**: エラーハンドリング
8. **task-manager.html**: アーカイブ機能

### 🟢 中優先度（Phase 3）

9. **flowchart**: タスクパネルUI
10. **flowchart**: ドラッグ&ドロップ
11. **両方**: 同期状態表示

---

## 成功基準

### Phase 1完了の定義

- ✅ task-manager でタスク編集すると flowchart が自動更新される
- ✅ flowchart でノード関連付けすると task-manager に反映される
- ✅ 本体ファイルへの自動保存が動作する
- ✅ file://プロトコルで動作する

### Phase 2完了の定義

- ✅ LocalStorage容量超過時にエラー表示される
- ✅ 不正なJSONデータでもクラッシュしない
- ✅ アーカイブ機能が動作する

### Phase 3完了の定義

- ✅ 同期状態が視覚的に分かる
- ✅ 手動リフレッシュで強制同期できる
- ✅ ユーザーフィードバックが適切

---

## 工数見積もり

| フェーズ | 工数 | 備考 |
|---------|------|------|
| Phase 0: 前提条件確認 | 1h | 既存実装の理解 |
| Phase 1: 基本連携 | 6-8h | コア機能の実装 |
| Phase 2: エラーハンドリング | 2-3h | 堅牢性の向上 |
| Phase 3: UX改善 | 3-4h | ユーザー体験の向上 |
| **合計** | **12-16h** | **2-3日間** |

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要、データ永続化戦略
- [01-localstorage-design.md](./01-localstorage-design.md) - LocalStorage設計、容量管理
- [02-data-structure.md](./02-data-structure.md) - データ構造定義
- [03-flowchart-integration.md](./03-flowchart-integration.md) - UI/UX設計

---

**最終更新**: 2026-03-21
**ステータス**: 実装準備完了
**次のアクション**: Phase 0実行 - 既存機能の確認
