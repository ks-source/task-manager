# LocalStorage設計 - クロスHTML連携

---
**feature**: クロスHTML連携
**document**: LocalStorage設計
**version**: 1.0
**status**: design
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

LocalStorage + storage イベントを使用した、独立したHTMLファイル間のリアルタイムデータ同期設計。

---

## storage イベントの仕組み

### 基本動作

```javascript
// ウィンドウA（task-manager.html）
localStorage.setItem('some-key', 'some-value');
  ↓
// ブラウザが storage イベントを発火
  ↓
// ウィンドウB（flowchart-editor.html）
window.addEventListener('storage', (e) => {
  console.log('Key changed:', e.key);
  console.log('Old value:', e.oldValue);
  console.log('New value:', e.newValue);
});
```

### 重要な制約

1. **同一ウィンドウでは発火しない**
   - データを書き込んだウィンドウ自身では storage イベントは発火しない
   - 別のウィンドウ・タブでのみ発火

2. **同一オリジン制約**
   - file:/// 同士なら動作
   - http://localhost と file:/// は異なるオリジン（動作しない）

3. **キー単位での通知**
   - どのキーが変更されたかを `e.key` で識別できる

---

## LocalStorageキー設計

### 1. task-manager-data（メインデータ）

**目的**: タスクデータ全体を保存

**構造**:
```javascript
{
  "tasks": [
    {
      "wbs_no": "WBS1.1.0",
      "task_name": "UI設計",
      "start_date": "2026/3/2",
      "finish_date": "2026/3/8",
      "status": "進行中",
      "priority": "高",
      "duration_bd": 7,
      // ... 既存のタスク属性

      // フローチャート属性（連携後に追加）
      "flowchart": {
        "nodeId": "node-123",
        "nodeType": "process",
        "position": { "x": 100, "y": 200 },
        "connectedTo": ["node-456"],
        "lastModified": 1234567890
      }
    }
  ],
  "meta": {
    "project_name": "プロジェクト名",
    "version": "1.0"
  },
  "timestamp": 1234567890  // 最終更新タイムスタンプ
}
```

**書き込みタイミング**:
- タスク編集時
- ガントチャートドラッグ終了時
- ステータス/優先度変更時

---

### 2. flowchart-attributes（フローチャート専用属性）

**目的**: フローチャート固有の情報を軽量に保存

**構造**:
```javascript
{
  "WBS1.1.0": {
    "nodeId": "node-123",
    "nodeType": "process",  // process | decision | start | end | data
    "position": { "x": 100, "y": 200 },
    "shape": "rectangle",
    "color": "#3498db",
    "connectedTo": ["node-456", "node-789"],
    "metadata": {
      "layer": 1,
      "group": "Phase1"
    },
    "timestamp": 1234567890
  },
  "WBS1.2.0": {
    "nodeId": "node-456",
    "nodeType": "decision",
    "position": { "x": 300, "y": 400 },
    "shape": "diamond",
    "color": "#e74c3c",
    "connectedTo": ["node-789"],
    "timestamp": 1234567891
  }
}
```

**書き込みタイミング**:
- ノード関連付け時
- ノード位置変更時
- エッジ接続変更時

---

### 3. *-update-trigger（更新通知専用）

**目的**: storage イベント確実に発火させる

**理由**:
- `task-manager-data` の一部だけ変更した場合、`setItem()`で上書きするとイベントは発火する
- しかし、明示的なトリガーを別キーで発火させることで、確実に検知できる

**task-manager-update-trigger**:
```javascript
localStorage.setItem('task-manager-update-trigger', Date.now().toString());
```

**flowchart-update-trigger**:
```javascript
localStorage.setItem('flowchart-update-trigger', Date.now().toString());
```

**値**: タイムスタンプ文字列（変更を検知しやすくするため）

---

## 実装詳細

### task-manager.html 側

#### 1. 既存の saveToLocalStorage() を拡張

```javascript
// 既存の実装
function saveToLocalStorage() {
  const data = {
    tasks: projectData.tasks,
    meta: projectData.meta,
    timestamp: Date.now()
  };

  try {
    localStorage.setItem('task-manager-data', JSON.stringify(data));

    // 🆕 追加: 更新トリガーを発火
    localStorage.setItem('task-manager-update-trigger', Date.now().toString());

    console.log('[TaskManager] Data saved to LocalStorage');
  } catch (error) {
    if (error.name === 'QuotaExceededError') {
      alert('⚠ LocalStorage容量が不足しています。古いデータを削除してください。');
    } else {
      console.error('[TaskManager] Failed to save:', error);
    }
  }
}
```

#### 2. storage イベントハンドラを追加

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
    projectData.tasks.forEach(task => {
      if (attrs[task.wbs_no]) {
        task.flowchart = attrs[task.wbs_no];
        console.log(`[TaskManager] Updated flowchart info for ${task.wbs_no}`);
      }
    });

    // UI更新
    renderTaskTable();

    // 変更を永続化（flowchart属性を含めて保存）
    saveToLocalStorage();

  } catch (error) {
    console.error('[TaskManager] Failed to parse flowchart attributes:', error);
  }
}
```

#### 3. 初期化時にフローチャート属性を読み込み

```javascript
// 🆕 追加: ページ読み込み時にフローチャート属性を統合
function loadFromLocalStorage() {
  // 既存の処理
  const dataStr = localStorage.getItem('task-manager-data');
  if (dataStr) {
    projectData = JSON.parse(dataStr);
  }

  // 🆕 フローチャート属性を読み込み
  const attrsStr = localStorage.getItem('flowchart-attributes');
  if (attrsStr) {
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

  renderTaskTable();
}
```

---

### flowchart-editor.html 側

#### 1. タスクデータ読み込み機能

```javascript
// 🆕 追加: タスクデータを LocalStorage から読み込み
let taskData = null;

function loadTaskData() {
  try {
    const dataStr = localStorage.getItem('task-manager-data');
    if (!dataStr) {
      console.warn('[Flowchart] No task data found in LocalStorage');
      return null;
    }

    taskData = JSON.parse(dataStr);
    console.log('[Flowchart] Loaded', taskData.tasks.length, 'tasks');

    return taskData;
  } catch (error) {
    console.error('[Flowchart] Failed to load task data:', error);
    return null;
  }
}

// ページ読み込み時に実行
window.addEventListener('DOMContentLoaded', () => {
  loadTaskData();
  renderFlowchart();
});
```

#### 2. storage イベントハンドラ

```javascript
// 🆕 追加: タスクマネージャー側の更新を検知
window.addEventListener('storage', handleTaskManagerUpdate);

function handleTaskManagerUpdate(e) {
  // タスクマネージャー更新トリガーのみを処理
  if (e.key !== 'task-manager-update-trigger') return;

  console.log('[Flowchart] Task data updated, reloading...');

  const newData = loadTaskData();
  if (newData) {
    taskData = newData;

    // フローチャートを再描画
    renderFlowchart();

    // 関連付けられたノードを更新
    updateTaskNodes();
  }
}

function updateTaskNodes() {
  // タスクに関連付けられたノードを更新
  taskData.tasks.forEach(task => {
    if (task.flowchart && task.flowchart.nodeId) {
      const node = document.getElementById(task.flowchart.nodeId);
      if (node) {
        // ノードのラベルを更新
        const label = node.querySelector('.node-label');
        if (label) {
          label.textContent = task.task_name;
        }

        // ステータスに応じて色を変更
        updateNodeColor(node, task.status);
      }
    }
  });
}
```

#### 3. フローチャート属性保存機能

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

  } catch (error) {
    if (error.name === 'QuotaExceededError') {
      alert('⚠ LocalStorage容量が不足しています。');
    } else {
      console.error('[Flowchart] Failed to save attribute:', error);
    }
  }
}

// ドラッグ&ドロップでタスクをノードに関連付ける例
function onTaskDroppedOnNode(taskWbsNo, nodeElement) {
  const nodeId = nodeElement.id;
  const nodeType = nodeElement.dataset.type || 'process';
  const position = {
    x: parseFloat(nodeElement.getAttribute('cx') || nodeElement.getAttribute('x')),
    y: parseFloat(nodeElement.getAttribute('cy') || nodeElement.getAttribute('y'))
  };

  saveFlowchartAttribute(taskWbsNo, nodeId, nodeType, position);

  // UIフィードバック
  showNotification(`タスク ${taskWbsNo} をノード ${nodeId} に関連付けました`);
}
```

---

## エラーハンドリング

### 1. LocalStorage容量超過

```javascript
function saveWithErrorHandling(key, value) {
  try {
    localStorage.setItem(key, value);
    return true;
  } catch (error) {
    if (error.name === 'QuotaExceededError') {
      // 容量超過時の処理
      console.error('[LocalStorage] Quota exceeded');

      // ユーザーに通知
      const result = confirm(
        'LocalStorageの容量が不足しています。\n' +
        '古いデータを削除しますか？'
      );

      if (result) {
        cleanupOldData();
        // 再試行
        try {
          localStorage.setItem(key, value);
          return true;
        } catch (retryError) {
          alert('データを保存できませんでした。ブラウザのストレージをクリアしてください。');
          return false;
        }
      }
    } else {
      console.error('[LocalStorage] Unexpected error:', error);
      return false;
    }
  }
}

function cleanupOldData() {
  // 古い更新トリガーを削除
  const keysToRemove = [];
  for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    if (key.endsWith('-update-trigger')) {
      keysToRemove.push(key);
    }
  }

  keysToRemove.forEach(key => localStorage.removeItem(key));
  console.log('[LocalStorage] Cleaned up', keysToRemove.length, 'old triggers');
}
```

### 2. JSONパースエラー

```javascript
function safeJSONParse(jsonString, defaultValue = null) {
  try {
    return JSON.parse(jsonString);
  } catch (error) {
    console.error('[JSON] Parse error:', error);
    console.error('[JSON] Invalid data:', jsonString);
    return defaultValue;
  }
}

// 使用例
function loadTaskData() {
  const dataStr = localStorage.getItem('task-manager-data');
  if (!dataStr) return null;

  const data = safeJSONParse(dataStr);
  if (!data) {
    alert('⚠ タスクデータが破損しています。LocalStorageをクリアしてください。');
    return null;
  }

  return data;
}
```

### 3. タイムスタンプ競合解決

```javascript
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
        // リモートの方が新しい
        merged[wbsNo] = remote;
        console.log(`[Merge] Using remote data for ${wbsNo}`);
      } else {
        // ローカルの方が新しい（変更なし）
        console.log(`[Merge] Keeping local data for ${wbsNo}`);
      }
    }
  }

  return merged;
}
```

---

## LocalStorage容量管理

### 容量上限の理解

**重要**: LocalStorageの容量制限は「ドメイン全体で共有される総容量」です。

| ブラウザ | 容量上限 | 備考 |
|---------|---------|------|
| Chrome/Edge | **5MB** | 最も厳しい制限 |
| Firefox | **10MB** | やや余裕あり |
| Safari | **5MB** | Chrome同等 |

**誤解されやすいポイント**:
- ❌ 間違い: 「1つのキーに5-10MB保存できる」
- ✅ 正解: 「すべてのキー合計で5-10MB」

```javascript
// file:// プロトコルの場合、すべての file:// が同一ドメイン扱い
localStorage.setItem('task-manager-data', '3MBのJSON');      // OK
localStorage.setItem('flowchart-attributes', '2MBのJSON');   // OK (合計5MB)
localStorage.setItem('other-data', '6MBのJSON');             // ❌ QuotaExceededError
```

### タスクデータのサイズ見積もり

現在のタスク構造から推定すると、1タスクあたり約250-300バイトです。

```javascript
// 1タスクあたりの平均サイズ（JSON文字列化後）
{
  "wbs_no": "WBS1.1.0",           // ~15 bytes
  "task_name": "UI設計",          // ~30 bytes (平均)
  "start_date": "2026/3/2",       // ~12 bytes
  "finish_date": "2026/3/8",      // ~12 bytes
  "status": "進行中",             // ~10 bytes
  "priority": "高",               // ~5 bytes
  "duration_bd": 7,               // ~5 bytes
  "comments": [...],              // ~50 bytes (平均)
  "flowchart": {...}              // ~150 bytes (付与時)
}
// 合計: 約 250-300 bytes/タスク
```

**タスク数別の容量見積もり**:

| タスク数 | 推定サイズ | LocalStorage使用率 (5MB基準) |
|---------|-----------|------------------------------|
| 100個 | ~30KB | 0.6% (余裕) |
| 500個 | ~150KB | 3% (余裕) |
| 1,000個 | ~300KB | 6% (余裕) |
| 5,000個 | ~1.5MB | 30% (注意) |
| 10,000個 | ~3MB | 60% (警告) |
| 15,000個 | ~4.5MB | 90% (危険) |
| 17,000個以上 | ~5MB以上 | **容量超過エラー** |

**実用的な制約**:
- タスク1,000個程度までは問題なく動作（推定300KB）
- 大量コメント・履歴データを含む場合は注意が必要
- Base64エンコードされた添付ファイルは使用不可（容量超過リスク大）

### 容量監視機能

保存時に自動的に容量をチェックし、警告を表示します。

```javascript
function saveToLocalStorage() {
  const data = {
    tasks: projectData.tasks,
    meta: projectData.meta,
    timestamp: Date.now()
  };

  const jsonStr = JSON.stringify(data);
  const sizeKB = new Blob([jsonStr]).size / 1024;
  const sizeMB = sizeKB / 1024;

  // 容量警告レベル
  if (sizeKB > 3072) { // 3MB超過
    console.warn(`[LocalStorage] データサイズ: ${sizeKB.toFixed(0)}KB (警告: 3MB超)`);

    const proceed = confirm(
      `タスクデータが大きくなっています (${sizeMB.toFixed(1)}MB)。\n` +
      `保存を続けますか？\n\n` +
      `対処法:\n` +
      `1. 完了タスクをアーカイブ\n` +
      `2. 不要なコメントを削除\n` +
      `3. JSONファイルへエクスポート`
    );

    if (!proceed) {
      return false;
    }
  }

  try {
    localStorage.setItem('task-manager-data', jsonStr);
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

### 容量削減戦略

#### 戦略1: 完了タスクのアーカイブ

```javascript
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

  alert(
    `✓ ${completedTasks.length}個の完了タスクをアーカイブしました。\n` +
    `ファイル: archived-tasks-${Date.now()}.json`
  );
}
```

#### 戦略2: フローチャート属性の分離管理

```javascript
// タスクデータからフローチャート属性を分離
function separateFlowchartAttributes() {
  const attrs = {};
  let separatedCount = 0;

  projectData.tasks.forEach(task => {
    if (task.flowchart) {
      attrs[task.wbs_no] = task.flowchart;
      delete task.flowchart; // メインデータから削除
      separatedCount++;
    }
  });

  if (separatedCount > 0) {
    // フローチャート属性を別キーで保存
    localStorage.setItem('flowchart-attributes', JSON.stringify(attrs));

    // メインデータを保存（軽量化済み）
    saveToLocalStorage();

    console.log(`[Optimize] ${separatedCount}個のフローチャート属性を分離`);
  }
}
```

#### 戦略3: 古いコメント・履歴の削除

```javascript
function cleanupOldComments(keepRecentDays = 30) {
  const cutoffTime = Date.now() - (keepRecentDays * 24 * 60 * 60 * 1000);
  let removedCount = 0;

  projectData.tasks.forEach(task => {
    if (task.comments && Array.isArray(task.comments)) {
      const originalLength = task.comments.length;

      // 最近N日のコメントのみ保持
      task.comments = task.comments.filter(comment => {
        const timestamp = new Date(comment.timestamp).getTime();
        return timestamp > cutoffTime;
      });

      removedCount += originalLength - task.comments.length;
    }
  });

  if (removedCount > 0) {
    saveToLocalStorage();
    alert(`✓ ${removedCount}個の古いコメントを削除しました。`);
  } else {
    alert('削除可能な古いコメントはありません。');
  }
}
```

### 容量状態の可視化

```javascript
// LocalStorage使用状況を表示
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

  // サイズ順にソート
  details.sort((a, b) => b.size - a.size);

  const totalKB = totalSize / 1024;
  const totalMB = totalKB / 1024;
  const usagePercent = (totalMB / 5 * 100).toFixed(1); // 5MB基準

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
```

### 容量超過時のユーザーガイド

容量超過エラーが発生した場合、ユーザーに以下の対処法を提示します:

```javascript
function showQuotaExceededHelp() {
  const status = showStorageStatus();

  const message = `
❌ LocalStorage容量が不足しています

現在の使用状況:
- 使用量: ${status.totalMB.toFixed(2)}MB / 5MB
- 使用率: ${status.usagePercent}%

対処法（推奨順）:

1️⃣ 完了タスクをアーカイブ
   → [アーカイブ実行]ボタンをクリック

2️⃣ 古いコメント・履歴を削除
   → [クリーンアップ]ボタンをクリック

3️⃣ JSONファイルへエクスポート
   → データをファイル保存し、LocalStorageをクリア

4️⃣ ブラウザストレージを手動クリア
   → F12 → Application → LocalStorage → 右クリック → Clear
  `;

  alert(message);
}
```

---

## パフォーマンス最適化

### 1. デバウンス処理

```javascript
// 連続的な保存を防ぐ
let saveTimeout = null;

function debouncedSave() {
  if (saveTimeout) {
    clearTimeout(saveTimeout);
  }

  saveTimeout = setTimeout(() => {
    saveToLocalStorage();
    saveTimeout = null;
  }, 500); // 500ms待機
}

// 使用例
function onTaskEdit() {
  // 即座に保存せず、デバウンス
  debouncedSave();
}
```

### 2. 差分更新

```javascript
// タスク全体ではなく、変更されたタスクのみを更新
function updateSingleTask(wbsNo, updates) {
  try {
    const dataStr = localStorage.getItem('task-manager-data');
    const data = JSON.parse(dataStr);

    const task = data.tasks.find(t => t.wbs_no === wbsNo);
    if (task) {
      Object.assign(task, updates);
      data.timestamp = Date.now();

      localStorage.setItem('task-manager-data', JSON.stringify(data));
      localStorage.setItem('task-manager-update-trigger', Date.now().toString());
    }
  } catch (error) {
    console.error('[Update] Failed to update single task:', error);
  }
}
```

---

## テスト項目

### 単体テスト

| 機能 | テストケース |
|------|------------|
| `saveToLocalStorage()` | 正常保存、容量超過、JSON変換エラー |
| `loadTaskData()` | データ存在、データなし、破損データ |
| `saveFlowchartAttribute()` | 新規追加、上書き、容量超過 |
| `mergeFlowchartAttributes()` | タイムスタンプ比較、キー追加 |

### 統合テスト

| シナリオ | 確認項目 |
|---------|---------|
| タスク編集 → フローチャート更新 | storage イベント発火、SVG更新 |
| ノード関連付け → タスク更新 | flowchart属性追加、UI反映 |
| 両方同時編集 | 競合解決、最新データ採用 |
| ブラウザリロード | データ永続化、属性復元 |

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要
- [02-data-structure.md](./02-data-structure.md) - データ構造定義
- [03-flowchart-integration.md](./03-flowchart-integration.md) - フローチャート連携
- [../gantt-interactive-editing/05-file-protocol-support.md](../gantt-interactive-editing/05-file-protocol-support.md) - postMessage vs LocalStorage

---

**最終更新**: 2026-03-21
**ステータス**: 設計完了、実装準備完了

---

## 補足: LocalStorage容量に関するFAQ

### Q1: タスク100個（3MB）を一度に読み込めますか？

**A**: はい、可能です。

- task-manager-data: 3MB
- flowchart-attributes: ~100KB
- 合計: 3.1MB < 5MB → ✅ OK

### Q2: タスク1000個（30MB）を一度に読み込めますか？

**A**: いいえ、容量超過エラーになります。

- task-manager-data: 30MB
- 合計: 30MB > 5MB → ❌ QuotaExceededError

ただし、**実際のタスク1000個は約300KB程度**なので問題なく動作します。30MBになるのは、1タスクあたり30KBの大量コメント・画像データを含む場合のみです。

### Q3: 容量制限は「1つのキー」ごとですか？

**A**: いいえ、「ドメイン全体」です。

- ❌ 間違い: 各キーに5MB保存できる
- ✅ 正解: すべてのキー合計で5MB

file:// プロトコルの場合、すべての file:// URLが同一ドメイン扱いとなります。

### Q4: 容量超過を防ぐ方法は？

**A**: 以下の対策があります:

1. **完了タスクのアーカイブ** - JSONファイルとしてダウンロード
2. **フローチャート属性の分離** - 別キーで軽量管理
3. **古いコメント削除** - 30日以前のコメントを削除
4. **容量監視** - 3MB超過時に自動警告

詳細は「LocalStorage容量管理」セクションを参照してください。
