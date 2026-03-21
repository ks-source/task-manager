# フローチャート連携仕様 - UI/UX設計

---
**feature**: クロスHTML連携
**document**: フローチャート連携
**version**: 1.0
**status**: design
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

task-manager.html と flowchart-editor.html の間でタスクデータとフローチャート属性を双方向同期する際のUI/UX設計。

---

## ユーザーフロー

### フロー1: タスクをフローチャートに反映

```
1. task-manager.html を開く
2. タスクを編集（名前、期間、ステータス等）
3. 自動保存 → LocalStorage更新
   ↓
4. flowchart-editor.html を開く（別タブ/ウィンドウ）
5. 自動的にタスクデータ読み込み
6. フローチャート上にタスク一覧表示
```

### フロー2: ノード関連付け

```
1. flowchart-editor.html でタスク一覧を表示
2. タスクをドラッグ
3. フローチャートノードにドロップ
4. 関連付け確認ダイアログ
5. LocalStorage に保存
   ↓
6. task-manager.html（開いている場合）が自動更新
7. タスク行にフローチャートアイコン表示
```

---

## flowchart-editor.html の UI拡張

### 1. タスクパネル（サイドバー）

```
┌─────────────────────────────────────────┐
│ タスク一覧                     [同期状態] │
├─────────────────────────────────────────┤
│ 🔄 最終同期: 2026/03/21 15:30           │
├─────────────────────────────────────────┤
│ □ WBS1.1.0 UI設計                       │
│   📅 2026/3/2 ~ 3/8  🟢 進行中          │
│                                         │
│ □ WBS1.2.0 実装                         │
│   📅 2026/3/9 ~ 3/15  ⚪ 未着手         │
│                                         │
│ ✓ WBS1.3.0 テスト                       │
│   📅 2026/3/16 ~ 3/20  🔵 関連付け済   │
│   → node-process-3                      │
├─────────────────────────────────────────┤
│ [手動同期] [すべて表示/非表示]           │
└─────────────────────────────────────────┘
```

### 2. ドラッグ&ドロップインタラクション

```
タスクパネルからノードへドラッグ:

[タスク WBS1.1.0]  ─┐
                    │ ドラッグ
                    ↓
              ┌───────────┐
              │  Process  │ ← ハイライト
              │   Node    │
              └───────────┘

ドロップ時:
┌─────────────────────────────┐
│ タスク関連付け確認            │
├─────────────────────────────┤
│ タスク: WBS1.1.0 UI設計      │
│ ノード: Process Node (ID: node-process-1) │
│                             │
│ このタスクをこのノードに      │
│ 関連付けますか？             │
│                             │
│     [キャンセル]  [関連付け] │
└─────────────────────────────┘
```

### 3. ノード表示の拡張

```
関連付け前:
┌───────────┐
│  Process  │
│           │
└───────────┘

関連付け後:
┌───────────┐
│  Process  │
│ WBS1.1.0  │ ← タスクWBS表示
│ UI設計    │ ← タスク名表示
│ 🟢 進行中  │ ← ステータス
└───────────┘
```

---

## task-manager.html の UI拡張

### 1. タスク一覧にフローチャート状態表示

```
WBS No   | Phase | タスク名 | 期間          | ステータス | フローチャート
---------|-------|---------|--------------|-----------|-------------
WBS1.1.0 | UI    | UI設計   | 3/2 ~ 3/8    | 進行中     | 🔵 node-process-1
WBS1.2.0 | 実装  | 実装     | 3/9 ~ 3/15   | 未着手     | -
WBS1.3.0 | テスト | テスト   | 3/16 ~ 3/20  | 完了      | 🔵 node-decision-2
```

### 2. タスク詳細モーダルにフローチャートタブ追加

```
┌────────────────────────────────────────┐
│ タスク詳細: WBS1.1.0                    │
├────────────────────────────────────────┤
│ [基本情報] [コメント] [フローチャート]  │
├────────────────────────────────────────┤
│ フローチャート情報:                     │
│                                        │
│ 関連ノード: node-process-1             │
│ ノードタイプ: Process (処理)            │
│ 位置: (150, 200)                       │
│ 最終更新: 2026/03/21 15:30             │
│                                        │
│ [フローチャートで表示]                  │
└────────────────────────────────────────┘
```

---

## 同期状態インジケーター

### flowchart-editor.html

```javascript
// 同期状態を表示
function updateSyncStatus(status) {
  const indicator = document.getElementById('sync-indicator');

  switch (status) {
    case 'synced':
      indicator.innerHTML = '🟢 同期済み';
      indicator.className = 'status-ok';
      break;

    case 'syncing':
      indicator.innerHTML = '🔄 同期中...';
      indicator.className = 'status-syncing';
      break;

    case 'error':
      indicator.innerHTML = '🔴 同期エラー';
      indicator.className = 'status-error';
      break;

    case 'no-data':
      indicator.innerHTML = '⚪ タスクデータなし';
      indicator.className = 'status-warning';
      break;
  }
}

// storage イベント発火時
window.addEventListener('storage', (e) => {
  if (e.key === 'task-manager-update-trigger') {
    updateSyncStatus('syncing');

    setTimeout(() => {
      loadTaskData();
      renderFlowchart();
      updateSyncStatus('synced');
    }, 300);
  }
});
```

---

## 実装例: ドラッグ&ドロップ

### flowchart-editor.html

```javascript
// タスク要素をドラッグ可能にする
function makeTaskDraggable(taskElement, task) {
  taskElement.draggable = true;

  taskElement.addEventListener('dragstart', (e) => {
    e.dataTransfer.setData('text/plain', task.wbs_no);
    e.dataTransfer.effectAllowed = 'link';

    // ドラッグ中のスタイル
    taskElement.classList.add('dragging');
  });

  taskElement.addEventListener('dragend', (e) => {
    taskElement.classList.remove('dragging');
  });
}

// ノードをドロップターゲットにする
function makeNodeDroppable(nodeElement) {
  nodeElement.addEventListener('dragover', (e) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'link';

    // ハイライト
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
      // 確認ダイアログ
      confirmTaskNodeLink(task, nodeElement);
    }
  });
}

// 関連付け確認
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

    // フィードバック
    showNotification(`✓ タスク ${task.wbs_no} を関連付けました`);
  }
}

// ノードラベルを更新
function updateNodeLabel(nodeElement, task) {
  const label = nodeElement.querySelector('.node-label');
  if (label) {
    label.innerHTML = `
      <tspan x="0" dy="0">${task.wbs_no}</tspan>
      <tspan x="0" dy="1.2em">${task.task_name}</tspan>
      <tspan x="0" dy="1.2em">${getStatusIcon(task.status)} ${task.status}</tspan>
    `;
  }

  // ステータスに応じて色を変更
  updateNodeColor(nodeElement, task.status);
}
```

---

## 通知・フィードバック

### トースト通知

```javascript
function showNotification(message, type = 'info') {
  const toast = document.createElement('div');
  toast.className = `toast toast-${type}`;
  toast.textContent = message;

  document.body.appendChild(toast);

  // アニメーション
  setTimeout(() => toast.classList.add('show'), 10);

  // 3秒後に消す
  setTimeout(() => {
    toast.classList.remove('show');
    setTimeout(() => toast.remove(), 300);
  }, 3000);
}

// 使用例
showNotification('✓ タスクデータを同期しました', 'success');
showNotification('⚠ LocalStorage容量が不足しています', 'warning');
showNotification('✗ 同期に失敗しました', 'error');
```

---

## エラーケースとハンドリング

### ケース1: タスクデータが存在しない

```javascript
function loadTaskData() {
  const dataStr = localStorage.getItem('task-manager-data');

  if (!dataStr) {
    // タスクデータなし
    updateSyncStatus('no-data');
    showEmptyState();
    return null;
  }

  // ...
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

### ケース2: 破損したデータ

```javascript
function loadTaskData() {
  const dataStr = localStorage.getItem('task-manager-data');
  if (!dataStr) return null;

  try {
    const data = JSON.parse(dataStr);

    // データ検証
    if (!data.tasks || !Array.isArray(data.tasks)) {
      throw new Error('Invalid data structure');
    }

    return data;

  } catch (error) {
    console.error('[Flowchart] Data corrupted:', error);
    updateSyncStatus('error');

    const recover = confirm(
      'タスクデータが破損しています。\n' +
      'LocalStorageをクリアしますか？'
    );

    if (recover) {
      localStorage.removeItem('task-manager-data');
      localStorage.removeItem('flowchart-attributes');
      location.reload();
    }

    return null;
  }
}
```

---

## 関連ドキュメント

- [README.md](./README.md)
- [01-localstorage-design.md](./01-localstorage-design.md)
- [02-data-structure.md](./02-data-structure.md)
- [../flowchart-manual-editing/README.md](../flowchart-manual-editing/README.md)

---

**最終更新**: 2026-03-21
**ステータス**: 設計完了
