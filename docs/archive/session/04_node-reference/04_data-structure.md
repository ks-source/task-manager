# データ構造定義 - v2.0メタデータスキーマ完全仕様

---
**作成日**: 2026-03-22
**バージョン**: 2.0
**対象**: flowchart-editor.html
**目的**: v2.0メタデータスキーマの完全な技術仕様定義（Phase 4統合版）

---

## 概要

本ドキュメントは、flowchart-editor.html の v2.0 メタデータスキーマの完全な技術仕様を定義する。

v2.0 では、以下3つの新機能が追加される：
1. **ステータス色分け機能**（Phase 4） - 要素の視覚的な状態管理
2. **参照関係管理機能** - ノード間の関連性を明示的に記録
3. **タスク連携スナップショット** - task-manager.htmlのタスク情報の保存

---

## v2.0 スキーマの全体像

### 完全なJSON構造

```json
{
  "version": "2.0",
  "schemaVersion": "2.0",
  "exportedAt": "2026-03-22T12:00:00Z",
  "svgFile": "workflow.svg",

  "elements": {
    "flowchart-A": {
      "type": "node",
      "displayText": "開始",
      "memo": "プロジェクトキックオフ会議で決定",
      "customLabel": "Phase 1開始",
      "originalLabel": "開始"
    }
  },

  "originalLabels": {
    "flowchart-A": "開始",
    "flowchart-B": "設計"
  },

  "manualNodes": {
    "memo_node_1": {
      "id": "memo_node_1",
      "x": 500,
      "y": 300,
      "width": 200,
      "height": 100,
      "label": "会議メモ",
      "content": "次回レビュー時に確認",
      "createdAt": "2026-03-22T10:00:00Z"
    }
  },

  "manualEdges": {
    "manual-edge-1710000000000": {
      "id": "manual-edge-1710000000000",
      "startX": 100,
      "startY": 50,
      "startNodeId": "flowchart-A",
      "endX": 300,
      "endY": 200,
      "endNodeId": "flowchart-B",
      "label": "承認後に実施",
      "style": {
        "color": "#666",
        "width": 2,
        "dashArray": "5,5"
      },
      "createdAt": "2026-03-21T10:30:00Z"
    }
  },

  "elementStatuses": {
    "flowchart-A": "completed",
    "flowchart-B": "in-progress",
    "subgraph_1": "on-hold",
    "memo_node_1": "postponed"
  },

  "references": {
    "flowchart-A": ["flowchart-B", "memo_node_1"],
    "flowchart-B": ["flowchart-C"],
    "memo_node_1": ["flowchart-A"]
  },

  "taskFlowchartLinks": {
    "nodes": {
      "WBS1.1.0": {
        "taskId": "task-001",
        "wbsNo": "WBS1.1.0",
        "taskName": "要件定義",
        "mermaidIds": ["flowchart-A", "flowchart-B"],
        "_snapshotAt": "2026-03-22T12:00:00Z"
      }
    },
    "_snapshotAt": "2026-03-22T12:00:00Z"
  }
}
```

---

## フィールド一覧表

### トップレベルフィールド

| フィールド | 型 | 必須 | v1.0 | v2.0 | 説明 |
|-----------|---|------|------|------|------|
| `version` | string | ✅ | ✅ | ✅ | メタデータバージョン（"1.0" or "2.0"） |
| `schemaVersion` | string | ✅ | ❌ | ✅ | スキーマバージョン（v2.0で明示的に追加） |
| `exportedAt` | string | ✅ | ✅ | ✅ | エクスポート日時（ISO 8601形式） |
| `svgFile` | string | ✅ | ✅ | ✅ | 元のSVGファイル名 |
| `elements` | object | ✅ | ✅ | ✅ | ノード・エッジのメモ・ラベル情報 |
| `originalLabels` | object | ✅ | ✅ | ✅ | 元のラベル情報 |
| `manualNodes` | object | ✅ | ✅ | ✅ | メモノード情報 |
| `manualEdges` | object | ✅ | ✅ | ✅ | 手動エッジ情報 |
| **`elementStatuses`** | **object** | ✅ | **❌** | **✅** | **ステータス情報（Phase 4）** |
| **`references`** | **object** | ✅ | **❌** | **✅** | **参照関係情報** |
| **`taskFlowchartLinks`** | **object** | ✅ | **❌** | **✅** | **タスク連携スナップショット** |

---

## グローバル変数

### JavaScript内部データ構造

```javascript
// ========== 既存の変数（v1.0から継続） ==========
let selectedElement = null;               // 選択中のSVG要素
let elementMemos = {};                    // { elementId: memoText }
let elementLabels = {};                   // { elementId: customLabel }
let originalLabels = {};                  // { elementId: originalLabel }
let manualNodes = {};                     // メモノードのマップ
let manualEdges = {};                     // 手動エッジのマップ
let autoSaveTimer = null;                 // 自動保存タイマー
let currentFileHandle = null;             // File System Access API ハンドル
let currentSvgFileName = null;            // 読み込み中のSVGファイル名

// ========== v2.0 新規追加の変数 ==========
let elementStatuses = {};                 // { elementId: statusKey } - Phase 4
let elementReferences = {};               // { elementId: [refId1, refId2, ...] } - 参照関係管理
```

---

## 1. elementStatuses（ステータス色分け機能 - Phase 4）

### データ構造

```javascript
elementStatuses = {
  'flowchart-A': 'completed',       // 正規ノード（Mermaidノード）
  'flowchart-B': 'in-progress',     // 正規ノード
  'subgraph_1': 'on-hold',          // サブグラフ
  'memo_node_1': 'postponed'        // メモノード
};

// ステータス未設定の要素はキーが存在しない（undefinedまたはnull）
```

### ステータス定義（task-manager.htmlと同一）

```javascript
/**
 * ステータス色定義
 * task-manager.htmlのステータス仕様と完全同一
 */
const STATUS_COLORS = Object.freeze({
  'not-started': {
    bg: '#e2e3e5',       // 背景色（薄グレー）
    text: '#383d41',     // テキスト色（ダークグレー）
    border: '#6c757d',   // ボーダー色
    label: '未着手'
  },
  'in-progress': {
    bg: '#fff3cd',       // 背景色（薄イエロー）
    text: '#856404',     // テキスト色（ダークイエロー）
    border: '#ffc107',   // ボーダー色
    label: '進行中'
  },
  'completed': {
    bg: '#d4edda',       // 背景色（薄グリーン）
    text: '#155724',     // テキスト色（ダークグリーン）
    border: '#28a745',   // ボーダー色
    label: '完了'
  },
  'on-hold': {
    bg: '#ffe5d0',       // 背景色（薄オレンジ）
    text: '#8a4a0e',     // テキスト色（ダークオレンジ）
    border: '#fd7e14',   // ボーダー色
    label: '保留'
  },
  'postponed': {
    bg: '#f8d7da',       // 背景色（薄ピンク）
    text: '#721c24',     // テキスト色（ダークレッド）
    border: '#dc3545',   // ボーダー色
    label: '延期'
  },
  'cancelled': {
    bg: '#f5f5f5',       // 背景色（薄グレー）
    text: '#666',        // テキスト色（グレー）
    border: '#999',      // ボーダー色
    label: '中止'
  },
  'unknown': {
    bg: '#e9ecef',       // 背景色（薄グレー）
    text: '#495057',     // テキスト色（グレー）
    border: '#adb5bd',   // ボーダー色
    label: '不明'
  }
});
```

### フィールド仕様

| フィールド | 型 | 説明 | 制約 |
|-----------|---|------|------|
| `elementStatuses` | object | 要素IDとステータスキーのマップ | トップレベルフィールド |
| キー | string | 要素ID（正規ノード・サブグラフ・メモノード） | 必須 |
| 値 | string | ステータスキー | `STATUS_COLORS`のキーのいずれか |

**値の制約**:
- `not-started`, `in-progress`, `completed`, `on-hold`, `postponed`, `cancelled`, `unknown`のいずれか
- ステータス未設定の要素はキーが存在しない

### タスクステータスとの独立性

**重要**: フローチャートステータスとタスクステータスは**完全に独立**している。

```
┌─────────────────────────┐         ┌─────────────────────────┐
│ flowchart-editor.html   │         │ task-manager.html       │
│                         │         │                         │
│ elementStatuses {       │         │ projectData.tasks [{    │
│   'F_01': 'completed'   │         │   wbs_no: "WBS1.1.0",   │
│ }                       │         │   status: "進行中"      │
│                         │ 🚫 連携なし │ }]                      │
│ 用途: 視覚的な状態管理    │         │ 用途: 実タスク進捗管理   │
│ 保存: SVGファイル         │         │ 保存: JSONファイル       │
└─────────────────────────┘         └─────────────────────────┘

理由:
1. フローチャートとタスクは1:Nの関係（1つのノードに複数タスク）
2. フローチャートステータスは「工程全体」の状態を表す
3. タスクステータスは「個別作業」の状態を表す
4. データ同期による複雑性を回避
```

---

## 2. references（参照関係管理機能）

### データ構造

```javascript
elementReferences = {
  'flowchart-A': ['flowchart-B', 'memo_node_1'],  // 正規ノード → 正規ノード + メモノード
  'flowchart-B': ['flowchart-C'],                 // 正規ノード → 正規ノード
  'memo_node_1': ['flowchart-A'],                 // メモノード → 正規ノード
  'subgraph_1': ['flowchart-D']                   // サブグラフ → 正規ノード
};

// 参照なしの要素はキーが存在しない
```

### フィールド仕様

| フィールド | 型 | 説明 | 制約 |
|-----------|---|------|------|
| `references` | object | 要素IDと参照先ID配列のマップ | トップレベルフィールド |
| キー | string | 参照元の要素ID | 必須 |
| 値 | array of string | 参照先の要素ID一覧 | 空配列不可（空の場合はキー削除） |

**値の制約**:
- 配列要素は有効な要素ID（正規ノード・サブグラフ・メモノード）
- 重複不可
- 自己参照不可（`'F_01': ['F_01']` は禁止）
- MVPではエッジIDは参照先から除外（Phase 3で開放検討）

### 参照の意味論

**設計原則**（外部レビューより）:

本機能における `references` は**図要素間の汎用的な関連先ID一覧**として扱う。

- **補足対象**: 「このメモノードはこの正規ノードの補足である」
- **関連対象**: 「この要素は別の要素と関連がある」
- **依存対象**: 「この要素は別の要素を前提としている」

将来的に relation type（related/prerequisite/supplement）の導入可能性を前提とするが、MVPでは単一の `references` 配列として実装。

### IDの安定性問題

Mermaid由来のIDは実装都合の識別子であり、必ずしも業務上安定した識別子ではない。

**対策**:
- SVG読込時に参照先IDが見つからない場合、ラベルベースの再解決候補をユーザーに提示
- 自動で勝手に修復せず、ユーザーに選択肢を見せる（Phase 2以降）

---

## 3. taskFlowchartLinks（タスク連携スナップショット）

### データ構造

```javascript
taskFlowchartLinks = {
  "nodes": {
    "WBS1.1.0": {
      "taskId": "task-001",
      "wbsNo": "WBS1.1.0",
      "taskName": "要件定義",
      "mermaidIds": ["flowchart-A", "flowchart-B"],
      "_snapshotAt": "2026-03-22T12:00:00Z"
    },
    "WBS1.2.0": {
      "taskId": "task-002",
      "wbsNo": "WBS1.2.0",
      "taskName": "基本設計",
      "mermaidIds": ["flowchart-C"],
      "_snapshotAt": "2026-03-22T12:00:00Z"
    }
  },
  "_snapshotAt": "2026-03-22T12:00:00Z"
};
```

### フィールド仕様

| フィールド | 型 | 説明 | 制約 |
|-----------|---|------|------|
| `taskFlowchartLinks` | object | タスク連携情報のスナップショット | トップレベルフィールド |
| `nodes` | object | WBS番号をキーとしたタスク連携マップ | 必須 |
| `_snapshotAt` | string | スナップショット取得日時（ISO 8601） | 必須 |

#### nodes の各エントリフィールド

| フィールド | 型 | 説明 | 必須 |
|-----------|---|------|------|
| `taskId` | string | タスクの一意識別子 | ✅ |
| `wbsNo` | string | WBS番号 | ✅ |
| `taskName` | string | タスク名 | ✅ |
| `mermaidIds` | array of string | 連携するMermaidノードID一覧 | ✅ |
| `_snapshotAt` | string | この連携情報のスナップショット日時 | ✅ |

### 正本（Source of Truth）の定義

**設計原則**（外部レビューより）:

- **タスク本体の正本**: A側（task-manager.html の LocalStorage）
- **SVG内の `taskFlowchartLinks`**: スナップショット（バックアップ）
- **競合時**: A側のタスクデータを優先

**運用方針**:
1. SVG単独読込時のみスナップショットを復元に使用
2. task-manager.htmlが起動している場合は、A側のデータを優先
3. 同期は手動トリガー（自動同期は行わない）

---

## buildMemoData() の最終形（完全版）

### 完全なコード

```javascript
/**
 * メタデータを構築してJSON形式で返す（v2.0完全版）
 * @returns {Object} v2.0メタデータオブジェクト
 */
function buildMemoData() {
  const elements = {};

  // 既存のメモ・ラベル情報を収集
  const allElementIds = new Set([
    ...Object.keys(elementMemos),
    ...Object.keys(elementLabels),
    ...Object.keys(originalLabels)
  ]);

  for (const elementId of allElementIds) {
    const element = document.querySelector(`[data-id="${CSS.escape(elementId)}"]`);
    const type = element?.getAttribute('data-et') || 'unknown';
    const displayText = getElementDisplayText(element);
    const memo = elementMemos[elementId] || '';
    const customLabel = elementLabels[elementId] || '';

    elements[elementId] = {
      type: type,
      displayText: displayText,
      memo: memo,
      customLabel: customLabel,
      originalLabel: originalLabels[elementId] || ''
    };
  }

  return {
    // バージョン情報
    version: "2.0",
    schemaVersion: "2.0",
    exportedAt: new Date().toISOString(),
    svgFile: currentSvgFileName || 'unknown.svg',

    // 既存フィールド（v1.0から継続）
    elements: elements,
    originalLabels: originalLabels,
    manualNodes: manualNodes,
    manualEdges: manualEdges,

    // v2.0 新機能
    elementStatuses: elementStatuses,        // Phase 4: ステータス色分け
    references: elementReferences,           // 参照関係管理
    taskFlowchartLinks: buildTaskLinks()     // タスク連携スナップショット
  };
}

/**
 * タスク連携情報を構築
 * @returns {Object} taskFlowchartLinks オブジェクト
 */
function buildTaskLinks() {
  // task-manager.html から受信したデータを使用
  const taskLinks = window.taskFlowchartData || { nodes: {} };

  return {
    nodes: taskLinks.nodes || {},
    _snapshotAt: new Date().toISOString()
  };
}
```

---

## マイグレーション処理（v1.0 → v2.0 統合版）

### 完全なマイグレーション関数

```javascript
/**
 * メタデータのバージョンマイグレーション（v1.0 → v2.0）
 * @param {Object} data - メタデータ
 * @returns {Object} マイグレーション後のデータ
 */
function migrateMetadata(data) {
  // 既にv2.0の場合はそのまま返す
  if (data.version === "2.0" || data.schemaVersion === "2.0") {
    console.log('[Migration] 既にv2.0形式です');
    return data;
  }

  // v1.0の場合、構造をクローンしてマイグレーション
  let migrated = structuredClone(data);

  console.log('[Migration] v1.0 → v2.0 マイグレーション開始');

  // バージョン情報の更新
  migrated.version = "2.0";
  migrated.schemaVersion = "2.0";

  // ★ Phase 4: ステータス管理の初期化
  if (!migrated.elementStatuses) {
    migrated.elementStatuses = {};
    console.log('[Migration] elementStatuses を空オブジェクトで初期化');
  }

  // ★ 参照関係管理の初期化
  if (!migrated.references) {
    migrated.references = {};
    console.log('[Migration] references を空オブジェクトで初期化');
  }

  // ★ タスク連携の初期化
  if (!migrated.taskFlowchartLinks) {
    migrated.taskFlowchartLinks = {
      nodes: {},
      _snapshotAt: null
    };
    console.log('[Migration] taskFlowchartLinks を空で初期化');
  }

  console.log('[Migration] ✅ v2.0へのマイグレーション完了');

  return migrated;
}
```

### バージョン互換性マトリクス

| バージョン | elementStatuses | references | taskFlowchartLinks | 後方互換性 | 前方互換性 |
|-----------|----------------|------------|-------------------|-----------|-----------|
| v1.0 | なし | なし | なし | - | ✅ v2.0で読み込み可（空で初期化） |
| v2.0 | あり | あり | あり | ❌ v1.0で読み込み不可 | - |

**注意事項**:
- v2.0ファイルをv1.0実装で開くと新規フィールドが無視される
- v1.0ファイルをv2.0実装で開くと自動マイグレーションされる
- マイグレーション後は再保存でv2.0形式になる

---

## 復元処理の完全なフロー

### SVG読み込み時の復元処理

```javascript
/**
 * SVGファイルを読み込んでメタデータを復元
 * @param {File} file - SVGファイル
 */
async function loadSVGFile(file) {
  const text = await file.text();
  const parser = new DOMParser();
  const svgDoc = parser.parseFromString(text, 'image/svg+xml');

  // SVGをDOMに挿入
  const svgContainer = document.getElementById('svg-container');
  svgContainer.innerHTML = '';
  svgContainer.appendChild(svgDoc.documentElement);

  // メタデータを抽出
  const memoData = extractMemoDataFromSVG(svgDoc);

  if (memoData) {
    // v1.0 → v2.0 マイグレーション
    const migratedData = migrateMetadata(memoData);

    // 1. 既存機能の復元（v1.0から継続）
    restoreLabelsFromMemoData(migratedData);
    restoreMemosFromMemoData(migratedData);
    restoreManualElementsFromMemoData(migratedData);

    // 2. v2.0機能の復元（順序任意 - 各関数は独立）
    restoreStatusesFromMetadata(migratedData);      // Phase 4: ステータス
    restoreReferencesFromMemoData(migratedData);    // 参照関係
    restoreTaskLinksFromMemoData(migratedData);     // タスク連携

    // 3. 統合検証
    validateRestoredData();

    console.log('[Load] ✅ SVG読み込み完了（v2.0）');
  } else {
    console.log('[Load] メタデータなし（新規SVG）');
    initializeGlobalState();
  }
}

/**
 * ステータス情報を復元（Phase 4）
 * @param {Object} metadata - メタデータ
 */
function restoreStatusesFromMetadata(metadata) {
  if (!metadata || !metadata.elementStatuses) {
    console.log('[Status] ステータス情報なし（v1.0以前のファイル）');
    elementStatuses = {};
    return;
  }

  elementStatuses = metadata.elementStatuses;

  // 各要素にステータススタイルを適用
  for (const [elementId, status] of Object.entries(elementStatuses)) {
    if (!STATUS_COLORS[status]) {
      console.warn(`[Status] 不正なステータス値: ${status} (要素ID: ${elementId})`);
      continue;
    }

    applyStatusStyle(elementId, status);
  }

  console.log(`[Status] ✅ 復元完了: ${Object.keys(elementStatuses).length}個の要素`);
}

/**
 * 参照関係情報を復元
 * @param {Object} memoData - メタデータ
 */
function restoreReferencesFromMemoData(memoData) {
  if (!memoData || !memoData.references) {
    console.log('[References] 参照関係情報なし');
    elementReferences = {};
    return;
  }

  elementReferences = memoData.references;

  // 無効な参照を検証
  for (const [elementId, refs] of Object.entries(elementReferences)) {
    const invalidRefs = refs.filter(refId => {
      const refElement = document.querySelector(`[data-id="${CSS.escape(refId)}"]`);
      return !refElement && !manualNodes[refId];
    });

    if (invalidRefs.length > 0) {
      console.warn(`[References] 無効な参照を検出: ${elementId} → [${invalidRefs.join(', ')}]`);
    }
  }

  console.log(`[References] ✅ 復元完了: ${Object.keys(elementReferences).length}個の要素`);
}

/**
 * タスク連携情報を復元
 * @param {Object} memoData - メタデータ
 */
function restoreTaskLinksFromMemoData(memoData) {
  if (!memoData || !memoData.taskFlowchartLinks) {
    console.log('[TaskLinks] タスク連携情報なし');
    return;
  }

  // スナップショットをグローバル変数に保存（必要に応じて）
  window.taskFlowchartSnapshot = memoData.taskFlowchartLinks;

  console.log(`[TaskLinks] ✅ 復元完了: ${Object.keys(memoData.taskFlowchartLinks.nodes).length}個のタスク`);
}

/**
 * グローバル状態の初期化
 */
function initializeGlobalState() {
  // 既存
  elementLabels = {};
  elementMemos = {};
  manualNodes = {};
  manualEdges = {};

  // v2.0追加
  elementStatuses = {};
  elementReferences = {};
}

/**
 * 復元データの検証
 */
function validateRestoredData() {
  // 参照関係の循環検出（簡易版）
  // Phase 2以降で実装予定

  console.log('[Validation] データ検証完了');
}
```

---

## バリデーション関数

### データ整合性検証

```javascript
/**
 * elementStatuses の検証
 * @param {Object} statuses - ステータスオブジェクト
 * @returns {boolean} 検証結果
 */
function validateElementStatuses(statuses) {
  if (typeof statuses !== 'object' || statuses === null) {
    console.error('[Validation] elementStatuses はオブジェクトである必要があります');
    return false;
  }

  for (const [elementId, status] of Object.entries(statuses)) {
    // ステータスキーの検証
    if (!STATUS_COLORS[status]) {
      console.error(`[Validation] 不正なステータスキー: ${status} (要素ID: ${elementId})`);
      return false;
    }

    // 要素IDの検証（要素が存在するか）
    const element = document.querySelector(`[data-id="${CSS.escape(elementId)}"]`);
    if (!element && !manualNodes[elementId]) {
      console.warn(`[Validation] 要素が見つかりません: ${elementId}`);
    }
  }

  return true;
}

/**
 * references の検証
 * @param {Object} references - 参照関係オブジェクト
 * @returns {boolean} 検証結果
 */
function validateReferences(references) {
  if (typeof references !== 'object' || references === null) {
    console.error('[Validation] references はオブジェクトである必要があります');
    return false;
  }

  for (const [elementId, refs] of Object.entries(references)) {
    // 配列であることを確認
    if (!Array.isArray(refs)) {
      console.error(`[Validation] 参照先は配列である必要があります: ${elementId}`);
      return false;
    }

    // 空配列は無効
    if (refs.length === 0) {
      console.warn(`[Validation] 空の参照配列: ${elementId}（キーを削除すべき）`);
    }

    // 自己参照の検出
    if (refs.includes(elementId)) {
      console.error(`[Validation] 自己参照が検出されました: ${elementId}`);
      return false;
    }

    // 重複の検出
    const uniqueRefs = [...new Set(refs)];
    if (uniqueRefs.length !== refs.length) {
      console.warn(`[Validation] 重複した参照先: ${elementId}`);
    }
  }

  return true;
}

/**
 * taskFlowchartLinks の検証
 * @param {Object} taskLinks - タスク連携オブジェクト
 * @returns {boolean} 検証結果
 */
function validateTaskFlowchartLinks(taskLinks) {
  if (typeof taskLinks !== 'object' || taskLinks === null) {
    console.error('[Validation] taskFlowchartLinks はオブジェクトである必要があります');
    return false;
  }

  // nodes フィールドの存在確認
  if (!taskLinks.nodes || typeof taskLinks.nodes !== 'object') {
    console.error('[Validation] taskFlowchartLinks.nodes が存在しません');
    return false;
  }

  // _snapshotAt の存在確認
  if (!taskLinks._snapshotAt) {
    console.warn('[Validation] taskFlowchartLinks._snapshotAt が存在しません');
  }

  return true;
}
```

---

## セキュリティ対策

### 1. CSS.escape() の使用

**脆弱性**: CSSセレクタインジェクション

```javascript
// ❌ 危険: セレクタが壊れる可能性
const id = 'node[data-dangerous="true"]';
document.querySelector(`[data-id="${id}"]`);

// ✅ 安全な実装
document.querySelector(`[data-id="${CSS.escape(id)}"]`);
```

**実装箇所**:
- 全ての `document.querySelector()` で使用
- `applyStatusStyle()`、`restoreReferencesFromMemoData()` 等

### 2. Prototype Pollution 対策

**脆弱性**: 外部SVG内のJSONをそのままオブジェクトにマージすると危険

```javascript
/**
 * メタデータのホワイトリスト検証
 * @param {Object} metadata - メタデータ
 * @returns {Object} サニタイズ済みメタデータ
 */
function sanitizeMetadata(metadata) {
  const ALLOWED_METADATA_KEYS = [
    'version', 'schemaVersion', 'exportedAt', 'svgFile',
    'elements', 'originalLabels', 'manualNodes', 'manualEdges',
    'elementStatuses', 'references', 'taskFlowchartLinks'
  ];

  const sanitized = Object.create(null);  // プロトタイプなしオブジェクト

  for (const key of ALLOWED_METADATA_KEYS) {
    if (key in metadata) {
      sanitized[key] = metadata[key];
    }
  }

  return sanitized;
}

/**
 * SVGからメタデータを抽出（サニタイズ付き）
 */
function extractMemoDataFromSVG(svgDoc) {
  const comment = findMemoComment(svgDoc);
  if (!comment) return null;

  try {
    const data = JSON.parse(comment.textContent.trim());
    return sanitizeMetadata(data);  // ★ サニタイズ
  } catch (error) {
    console.error('[Extract] メタデータのパースに失敗:', error);
    return null;
  }
}
```

### 3. SVG Sanitization

**脆弱性**: 読み込むSVGに `<script>` や `on*` 属性が含まれている場合

```javascript
/**
 * SVGコンテンツのサニタイズ
 * @param {string} svgString - SVG文字列
 * @returns {string} サニタイズ済みSVG
 */
function sanitizeSVG(svgString) {
  // <script> タグを削除
  svgString = svgString.replace(/<script[^>]*>[\s\S]*?<\/script>/gi, '');

  // on* イベント属性を削除
  svgString = svgString.replace(/\son\w+="[^"]*"/g, '');
  svgString = svgString.replace(/\son\w+='[^']*'/g, '');

  // foreignObject を削除（任意のHTMLを含む可能性）
  svgString = svgString.replace(/<foreignObject[^>]*>[\s\S]*?<\/foreignObject>/gi, '');

  return svgString;
}
```

### 4. XSS対策

**脆弱性**: innerHTML の使用

```javascript
// ❌ 危険: XSSの可能性
element.innerHTML = userInput;

// ✅ 安全な実装: textContent + addEventListener
element.textContent = userInput;
element.addEventListener('click', handleClick);
```

---

## 更新履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-22 | 1.0 | 初版作成 - 参照関係管理機能の基本仕様 |
| 2026-03-22 | 2.0 | v2.0完全版 - Phase 4（ステータス色分け機能）を統合 |

---

**関連ドキュメント**:
- [02_proposed-design.md](./02_proposed-design.md) - 提案設計
- [05_implementation-plan.md](./05_implementation-plan.md) - 実装計画
- [06_technical-concerns.md](./06_technical-concerns.md) - 技術的懸念事項
- [07_integrated-roadmap.md](./07_integrated-roadmap.md) - 統合実装ロードマップ
