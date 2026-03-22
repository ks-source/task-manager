# Phase 1修正: mermaid_ids双方向同期の実装

---
**feature**: データ連携機能（Phase 1修正）
**issue**: mermaid_ids（リンク情報）のB→A送信機能が欠落
**version**: 1.1
**status**: 仕様確定、実装待ち
**created**: 2026-03-22
**updated**: 2026-03-22
**priority**: 🔴 Critical
**author**: Claude Code Session
---

## 📋 問題の発見

### 背景

Phase 1実装（v2.15.0、2026-03-22完了）において、以下が実装されました：

| 実装内容 | 状態 | 場所 |
|---------|------|------|
| B側でノード/エッジへの属性付与 | ✅ 完了 | flowchart-editor.html lines 3453-3468 |
| 属性情報のB→A送信 | ✅ 完了 | flowchart-editor.html lines 3508-3564 |
| A側での属性情報統合 | ✅ 完了 | task-manager.html lines 1722-1759 |
| 永続化（JSON/LocalStorage） | ✅ 完了 | task-manager.html lines 2659-2680 |

### 発見された重大な欠落

しかし、**タスク⇔フローチャート要素のリンク情報（mermaid_ids）**そのもののB→A送信機能が実装されていませんでした。

#### 実装されているB側の機能（問題なし）

**タスク編集モーダル**（flowchart-editor.html）:
- ✅ ノード追加機能（lines 6293-6329）
- ✅ ノード削除機能（lines 6374-6384）
- ✅ エッジ追加機能（lines 6331-6372）
- ✅ エッジ削除機能（lines 6386-6395）
- ✅ mermaidIds更新（line 6140）

```javascript
// location: flowchart-editor.html lines 6133-6144
function saveTaskEditModal() {
  if (!currentEditingTask) return;

  currentEditingTask.taskName = document.getElementById('edit-task-name').value;
  currentEditingTask.status = document.getElementById('edit-task-status').value;

  // ★重要: flowchartLinks.nodesからmermaidIdsを生成
  currentEditingTask.mermaidIds =
    currentEditingTask.flowchartLinks.nodes.map(n => n.id).join(',');

  renderTaskList();
  closeTaskEditModal();
}
```

#### 欠落している送信機能（問題あり）

**publishFlowchartAttributes()の問題**:

```javascript
// location: flowchart-editor.html lines 3508-3564
function publishFlowchartAttributes() {
  const taskManagerData = loadTaskManagerData();
  const flowchartAttrs = {};

  taskManagerData.tasks.forEach(task => {
    // ★問題: A側のmermaid_idsを読み取るだけ
    const nodeIds = task.mermaid_ids.split(',').map(id => id.trim());

    // ★問題: B側で編集したmermaidIdsを送信していない！
    flowchartAttrs[task.wbs_no] = {
      nodeIds: nodeIds,  // ← A側の古い値をそのまま返している
      memos: { ... },
      customLabels: { ... }
    };
  });
}
```

---

## 🔄 現状のデータフロー（問題あり）

### シナリオ: B側でタスクにノードを追加

```
【ステップ1】A側の初期状態
A: task-manager.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ task_name: "要件定義"    │
│ mermaid_ids: "node_1"   │ ← 初期値
└─────────────────────────┘
        ↓ LocalStorage送信

【ステップ2】B側で受信・表示
B: flowchart-editor.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ taskName: "要件定義"     │
│ mermaidIds: "node_1"    │ ← A側から受信
│ flowchartLinks.nodes:   │
│   ["node_1"]            │
└─────────────────────────┘
        ↓ ユーザー操作

【ステップ3】B側でnode_2を追加
タスク編集モーダル:
- ノードセレクタでnode_2を選択
- 「追加」ボタンクリック
- saveTaskEditModal()実行

B: flowchart-editor.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ mermaidIds: "node_1,node_2" │ ← 更新された！
│ flowchartLinks.nodes:   │
│   ["node_1", "node_2"]  │
└─────────────────────────┘
        ↓ Ctrl+S保存

【ステップ4】publishFlowchartAttributes()実行
function publishFlowchartAttributes() {
  taskManagerData.tasks.forEach(task => {
    // ★問題: A側のmermaid_idsを参照
    const nodeIds = task.mermaid_ids.split(',');
    // task.mermaid_ids = "node_1" (古い値)

    flowchartAttrs[task.wbs_no] = {
      nodeIds: ["node_1"],  // ← B側の編集が反映されない
      memos: { ... }
    };
  });
}
        ↓ LocalStorage送信

【ステップ5】A側で受信（問題発生）
A: task-manager.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ mermaid_ids: "node_1"   │ ← 更新されない！
│ flowchart: {            │
│   nodeIds: ["node_1"]   │ ← 古い値
│ }                       │
└─────────────────────────┘

❌ 結果: B側でnode_2を追加したのに、A側に反映されない
```

---

## ✅ 期待されるデータフロー（修正後）

### 同じシナリオ: B側でタスクにノードを追加

```
【ステップ1～3】同じ
B側でnode_2を追加
mermaidIds: "node_1,node_2"
        ↓ Ctrl+S保存

【ステップ4】publishFlowchartAttributes()実行（修正版）
function publishFlowchartAttributes() {
  taskManagerData.tasks.forEach(task => {
    // ★修正: B側のmockTasksを参照
    const localTask = mockTasks.find(t => t.wbs === task.wbs_no);

    let nodeIds = [];
    if (localTask && localTask.mermaidIds) {
      nodeIds = localTask.mermaidIds.split(','); // ← B側の最新値
    } else if (task.mermaid_ids) {
      nodeIds = task.mermaid_ids.split(',');
    }

    flowchartAttrs[task.wbs_no] = {
      nodeIds: nodeIds,
      mermaidIds: nodeIds.join(', '),  // ★追加: A側フォーマット
      memos: { ... }
    };
  });
}
        ↓ LocalStorage送信

【ステップ5】A側で受信（修正版）
function handleFlowchartUpdate(e) {
  const attrs = JSON.parse(localStorage.getItem('flowchart-attributes'));

  projectData.tasks.forEach(task => {
    if (attrs[task.wbs_no]) {
      task.flowchart = attrs[task.wbs_no];

      // ★追加: mermaidIdsを反映
      if (attrs[task.wbs_no].mermaidIds) {
        task.mermaid_ids = attrs[task.wbs_no].mermaidIds;
      }
    }
  });

  saveToLocalStorage();  // ← mermaid_idsも保存される
}

A: task-manager.html
┌─────────────────────────┐
│ Task: WBS1.1.0          │
│ mermaid_ids: "node_1, node_2" │ ← 更新された！✅
│ flowchart: {            │
│   nodeIds: ["node_1", "node_2"] │
│   mermaidIds: "node_1, node_2"  │
│ }                       │
└─────────────────────────┘

✅ 結果: B側の編集がA側に正しく反映される
```

---

## 🔧 技術仕様

### 修正1: publishFlowchartAttributes()の拡張

**ファイル**: `flowchart-editor.html`
**場所**: lines 3508-3564
**修正内容**: B側のmockTasksを参照し、編集されたmermaidIdsを優先的に送信

#### 修正前のコード

```javascript
function publishFlowchartAttributes() {
  const taskManagerData = loadTaskManagerData();
  if (!taskManagerData || !taskManagerData.tasks) {
    console.log('[FlowchartEditor] Task Manager not connected - skipping publish');
    return;
  }

  const flowchartAttrs = {};

  // 各タスクのmermaid_idsに紐づくフローチャート情報を収集
  taskManagerData.tasks.forEach(task => {
    if (!task.mermaid_ids) return;

    // ★問題: A側のmermaid_idsを使用
    const nodeIds = task.mermaid_ids.split(',').map(id => id.trim()).filter(id => id);
    if (nodeIds.length === 0) return;

    const memos = {};
    const customLabels = {};
    const relatedManualEdges = [];

    // メモ・カスタムラベル収集処理...

    flowchartAttrs[task.wbs_no] = {
      nodeIds: nodeIds,  // ← A側の値
      memos: memos,
      customLabels: customLabels,
      relatedManualEdges: relatedManualEdges
    };
  });

  localStorage.setItem('flowchart-attributes', JSON.stringify(flowchartAttrs));
  localStorage.setItem('flowchart-update-trigger', Date.now().toString());
}
```

#### 修正後のコード

```javascript
function publishFlowchartAttributes() {
  const taskManagerData = loadTaskManagerData();
  if (!taskManagerData || !taskManagerData.tasks) {
    console.log('[FlowchartEditor] Task Manager not connected - skipping publish');
    return;
  }

  const flowchartAttrs = {};

  // 各タスクのmermaid_idsに紐づくフローチャート情報を収集
  taskManagerData.tasks.forEach(task => {
    // ★修正: B側で編集されたタスクを探す
    const localTask = mockTasks.find(t => t.wbs === task.wbs_no);

    // B側で編集されている場合は、B側の値を優先
    let nodeIds = [];
    if (localTask && localTask.mermaidIds) {
      nodeIds = localTask.mermaidIds.split(',').map(id => id.trim()).filter(id => id);
    } else if (task.mermaid_ids) {
      nodeIds = task.mermaid_ids.split(',').map(id => id.trim()).filter(id => id);
    }

    if (nodeIds.length === 0) return;

    const memos = {};
    const customLabels = {};
    const relatedManualEdges = [];

    // 各ノードIDのメモ・カスタムラベルを収集
    nodeIds.forEach(nodeId => {
      if (elementMemos[nodeId]) {
        memos[nodeId] = elementMemos[nodeId];
      }
      if (elementCustomLabels[nodeId]) {
        customLabels[nodeId] = elementCustomLabels[nodeId];
      }
    });

    // 関連する手動エッジを収集
    Object.keys(manualEdges).forEach(edgeId => {
      const edge = manualEdges[edgeId];
      if (nodeIds.includes(edge.from) || nodeIds.includes(edge.to)) {
        relatedManualEdges.push(edgeId);
      }
    });

    // タスクのWBS番号をキーとしてflowchart属性を保存
    flowchartAttrs[task.wbs_no] = {
      nodeIds: nodeIds,
      mermaidIds: nodeIds.join(', '),  // ★追加: A側のフォーマットで送信
      memos: memos,
      customLabels: customLabels,
      relatedManualEdges: relatedManualEdges
    };
  });

  // LocalStorageへ送信
  try {
    localStorage.setItem('flowchart-attributes', JSON.stringify(flowchartAttrs));
    localStorage.setItem('flowchart-update-trigger', Date.now().toString());
    console.log('[FlowchartEditor] Published flowchart attributes for', Object.keys(flowchartAttrs).length, 'tasks');
  } catch (err) {
    console.error('[FlowchartEditor] Failed to publish flowchart attributes:', err);
  }
}
```

### 修正2: handleFlowchartUpdate()の拡張

**ファイル**: `task-manager.html`
**場所**: lines 1722-1759
**修正内容**: 受信したmermaidIdsをtask.mermaid_idsへ反映

#### 修正前のコード

```javascript
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
        task.flowchart = attrs[task.wbs_no];  // ← flowchart属性のみ更新
        updatedCount++;
        console.log(`[TaskManager] Updated flowchart info for ${task.wbs_no}`);
      }
    });

    if (updatedCount > 0) {
      // UI更新
      renderTaskTable();

      // 変更を永続化（flowchart属性を含めて保存）
      markDirty();
      saveToLocalStorage();

      console.log(`[TaskManager] ✓ ${updatedCount}個のタスクにフローチャート情報を統合しました`);
    }

  } catch (error) {
    console.error('[TaskManager] Failed to parse flowchart attributes:', error);
  }
}
```

#### 修正後のコード

```javascript
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

        // ★追加: B側で編集されたmermaidIdsを反映
        if (attrs[task.wbs_no].mermaidIds) {
          const oldValue = task.mermaid_ids || '';
          const newValue = attrs[task.wbs_no].mermaidIds;

          if (oldValue !== newValue) {
            task.mermaid_ids = newValue;
            console.log(`[TaskManager] Updated mermaid_ids for ${task.wbs_no}: "${oldValue}" → "${newValue}"`);
          }
        }

        updatedCount++;
        console.log(`[TaskManager] Updated flowchart info for ${task.wbs_no}`);
      }
    });

    if (updatedCount > 0) {
      // UI更新
      renderTaskTable();

      // 変更を永続化（flowchart属性とmermaid_idsを含めて保存）
      markDirty();
      saveToLocalStorage();

      console.log(`[TaskManager] ✓ ${updatedCount}個のタスクにフローチャート情報を統合しました`);
    }

  } catch (error) {
    console.error('[TaskManager] Failed to parse flowchart attributes:', error);
  }
}
```

---

## 📊 データスキーマ

### 送信データ構造（LocalStorage: flowchart-attributes）

```typescript
interface FlowchartAttributes {
  [wbsNo: string]: {
    nodeIds: string[];           // 紐づくノードIDのリスト
    mermaidIds: string;          // ★追加: A側のフォーマット（カンマ+スペース区切り）
    memos: {                     // ノードIDごとのメモ
      [nodeId: string]: string;
    };
    customLabels: {              // ノードIDごとのカスタムラベル
      [nodeId: string]: string;
    };
    relatedManualEdges: string[]; // 関連する手動エッジのIDリスト
  };
}
```

### 送信データ例

```json
{
  "WBS1.1.0": {
    "nodeIds": ["node_1", "node_2"],
    "mermaidIds": "node_1, node_2",
    "memos": {
      "node_1": "要件定義の詳細説明"
    },
    "customLabels": {
      "node_1": "要件（重要）"
    },
    "relatedManualEdges": ["edge_123"]
  }
}
```

### A側のタスクデータ統合後

```json
{
  "wbs_no": "WBS1.1.0",
  "task_name": "要件定義",
  "mermaid_ids": "node_1, node_2",
  "status": "進行中",
  "flowchart": {
    "nodeIds": ["node_1", "node_2"],
    "mermaidIds": "node_1, node_2",
    "memos": {
      "node_1": "要件定義の詳細説明"
    },
    "customLabels": {
      "node_1": "要件（重要）"
    },
    "relatedManualEdges": ["edge_123"]
  }
}
```

---

## 📅 実装計画

### 工数見積もり

| タスク | 工数 |
|--------|------|
| flowchart-editor.html修正 | 30分 |
| task-manager.html修正 | 15分 |
| 動作確認・デバッグ | 15-30分 |
| **合計** | **1-1.5時間** |

### 実装順序

1. **flowchart-editor.html修正**
   - publishFlowchartAttributes()関数を拡張
   - コンソールログで送信内容を確認

2. **task-manager.html修正**
   - handleFlowchartUpdate()関数を拡張
   - コンソールログで受信・更新を確認

3. **統合テスト**
   - 検証シナリオ実施

---

## 🧪 検証シナリオ

### シナリオ1: ノード追加の反映確認

**前提条件**:
- task-manager.htmlでタスクWBS1.1.0作成
- mermaid_ids: "node_1"

**検証手順**:

1. **flowchart-editor.htmlでSVGを開く**
   - コンソールで「Task Manager connected: X tasks」確認
   - 左サイドパネルにWBS1.1.0が表示される

2. **タスク編集モーダルでnode_2を追加**
   - WBS1.1.0の「編集」ボタンをクリック
   - ノードセレクタでnode_2を選択
   - 「追加」ボタンクリック
   - 「保存」ボタンクリック

3. **Ctrl+Sで保存**
   - コンソールで「Published flowchart attributes for X tasks」確認
   - コンソールログに送信データを表示（mermaidIdsを含む）

4. **LocalStorageを確認**（DevTools）
   - `flowchart-attributes`キー
   - ✅ **期待値**:
     ```json
     {
       "WBS1.1.0": {
         "nodeIds": ["node_1", "node_2"],
         "mermaidIds": "node_1, node_2"
       }
     }
     ```

5. **task-manager.htmlで確認**
   - コンソールで「Updated mermaid_ids for WBS1.1.0」確認
   - タスク詳細モーダルを開く
   - Mermaid ノードIDフィールドを確認
   - ✅ **期待値**: "node_1, node_2"

6. **JSONエクスポートで永続性確認**
   - JSONエクスポート実行
   - ファイル内容を確認
   - ✅ **期待値**: `mermaid_ids: "node_1, node_2"`が含まれる

### シナリオ2: ノード削除の反映確認

**前提条件**:
- シナリオ1完了状態
- mermaid_ids: "node_1, node_2"

**検証手順**:

1. **flowchart-editor.htmlでnode_2を削除**
   - タスク編集モーダルを開く
   - node_2の「削除」ボタンをクリック
   - 「保存」ボタンクリック

2. **Ctrl+Sで保存**
   - コンソールログ確認

3. **task-manager.htmlで確認**
   - ✅ **期待値**: mermaid_ids: "node_1"に戻る

---

## ⚠️ リスクと対策

### リスク1: フォーマットの不一致

**問題**: A側は "node_1, node_2"（カンマ+スペース）、B側は "node_1,node_2"（カンマのみ）

**対策**:
- publishFlowchartAttributes()で`.join(', ')`を使用
- handleFlowchartUpdate()ではそのまま受け取る

### リスク2: LocalStorage容量制限

**問題**: mermaidIdsフィールド追加でデータ量が増加

**影響**: 低（文字列1つ追加のみ）

**対策**: 既存のLocalStorage監視機能で警告表示

### リスク3: 既存データとの互換性

**問題**: mermaidIdsフィールドが存在しない古いデータ

**対策**:
- `if (attrs[task.wbs_no].mermaidIds)`で存在チェック
- 存在しない場合はスキップ（既存動作を維持）

---

## 🔗 関連ドキュメント

- [future-roadmap.md](./future-roadmap.md) - Phase 0～3全体計画
- [phase1-implementation-summary.md](./phase1-implementation-summary.md) - Phase 1実装詳細
- [data-integration/README.md](./README.md) - データ連携機能概要

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成 - Phase 1修正仕様策定 |

---

**最終更新**: 2026-03-22
**ステータス**: 仕様確定、実装待ち
**次のアクション**: flowchart-editor.htmlのpublishFlowchartAttributes()を修正

