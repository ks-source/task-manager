# データ構造定義

---
**feature**: フローチャート手動編集
**document**: データ構造定義
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

手動編集機能で扱う全てのデータ構造を定義する。

## JavaScript内部データ

### グローバル変数

```javascript
// ========== 既存の変数 ==========
let selectedElement = null;               // 選択中のSVG要素
let elementMemos = {};                    // { elementId: memoText }
let elementCustomLabels = {};             // { elementId: customLabel }
let originalLabels = {};                  // { elementId: originalLabel }
let autoSaveTimer = null;                 // 自動保存タイマー
let currentFileHandle = null;             // File System Access API ハンドル
let currentSvgFileName = null;            // 読み込み中のSVGファイル名

// ========== 新規追加の変数 ==========
let edgeAddMode = false;                  // エッジ追加モードON/OFF
let edgeStartPoint = null;                // 開始点情報（後述）
let edgePreviewLine = null;               // プレビュー線のSVG要素参照
let manualEdges = {};                     // 手動エッジのマップ
let manualEdgeCounter = 0;                // エッジID生成用カウンター（未使用）
```

### edgeStartPoint の構造

```javascript
// エッジ追加の開始点情報
edgeStartPoint = {
  x: 150,                    // SVG座標系のX座標
  y: 80,                     // SVG座標系のY座標
  nodeId: "flowchart-A"      // 吸着したノードID（吸着していない場合はnull）
};

// 例: 自由位置の場合
edgeStartPoint = {
  x: 200,
  y: 100,
  nodeId: null               // ノードに接続していない
};
```

### manualEdges の構造

```javascript
// 手動エッジのマップ
manualEdges = {
  "manual-edge-1710000000000": {
    id: "manual-edge-1710000000000",
    startX: 100,                      // 開始点X座標
    startY: 50,                       // 開始点Y座標
    startNodeId: "flowchart-A",       // 開始点ノードID（nullの場合あり）
    endX: 300,                        // 終了点X座標
    endY: 200,                        // 終了点Y座標
    endNodeId: "flowchart-B",         // 終了点ノードID（nullの場合あり）
    label: "承認後に実施",             // エッジラベル（空文字の場合あり）
    style: {                          // スタイル情報
      color: "#666",                  // 線の色
      width: 2,                       // 線の太さ
      dashArray: "5,5"                // 破線パターン
    },
    createdAt: "2026-03-21T10:30:00Z" // 作成日時（ISO 8601形式）
  },

  "manual-edge-1710000001234": {
    id: "manual-edge-1710000001234",
    startX: 150,
    startY: 80,
    startNodeId: null,                // ノードに接続していない
    endX: 400,
    endY: 250,
    endNodeId: null,                  // ノードに接続していない
    label: "",                        // ラベルなし
    style: {
      color: "#666",
      width: 2,
      dashArray: "5,5"
    },
    createdAt: "2026-03-21T10:32:00Z"
  }
};
```

## JSON保存形式

### ファイル構造（統合版）

```json
{
  "flowchartId": "project-workflow",
  "svgFile": "workflow.svg",
  "savedAt": "2026-03-21T11:00:00Z",

  "elements": {
    "flowchart-A": {
      "type": "node",
      "displayText": "開始",
      "memo": "プロジェクトキックオフ会議で決定",
      "customLabel": "Phase 1開始",
      "originalLabel": "開始"
    },
    "flowchart-B": {
      "type": "node",
      "displayText": "設計",
      "memo": "要件定義書を基に設計",
      "customLabel": "",
      "originalLabel": "設計"
    }
  },

  "originalLabels": {
    "flowchart-A": "開始",
    "flowchart-B": "設計",
    "flowchart-C": "実装"
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
  }
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 | 必須 |
|-----------|---|------|------|
| `flowchartId` | string | フローチャートの識別子 | ✅ |
| `svgFile` | string | 元のSVGファイル名 | ✅ |
| `savedAt` | string | 保存日時（ISO 8601） | ✅ |
| `elements` | object | ノード・エッジのメモ・ラベル情報 | ✅ |
| `originalLabels` | object | 元のラベル情報 | ✅ |
| `manualEdges` | object | 手動エッジ情報 | ✅ |

#### manualEdges の各エッジフィールド

| フィールド | 型 | 説明 | 必須 |
|-----------|---|------|------|
| `id` | string | エッジのユニークID | ✅ |
| `startX` | number | 開始点X座標（SVG座標系） | ✅ |
| `startY` | number | 開始点Y座標（SVG座標系） | ✅ |
| `startNodeId` | string\|null | 開始点ノードID（接続なしの場合null） | ✅ |
| `endX` | number | 終了点X座標（SVG座標系） | ✅ |
| `endY` | number | 終了点Y座標（SVG座標系） | ✅ |
| `endNodeId` | string\|null | 終了点ノードID（接続なしの場合null） | ✅ |
| `label` | string | エッジラベル（空文字可） | ✅ |
| `style` | object | スタイル情報 | ✅ |
| `style.color` | string | 線の色（CSSカラー） | ✅ |
| `style.width` | number | 線の太さ（px） | ✅ |
| `style.dashArray` | string | 破線パターン（SVG形式） | ✅ |
| `createdAt` | string | 作成日時（ISO 8601） | ✅ |

## 接続グラフ出力形式（Phase 3）

### JSON出力形式

```json
{
  "flowchartId": "project-workflow",
  "exportedAt": "2026-03-21T11:30:00Z",
  "svgFile": "workflow.svg",

  "nodes": [
    {
      "id": "flowchart-A",
      "type": "mermaid",
      "label": "開始",
      "displayText": "Phase 1開始",
      "hasCustomLabel": true,
      "hasMemo": true
    },
    {
      "id": "flowchart-B",
      "type": "mermaid",
      "label": "設計",
      "displayText": "設計",
      "hasCustomLabel": false,
      "hasMemo": true
    }
  ],

  "connections": [
    {
      "id": "manual-edge-1710000000000",
      "from": "flowchart-A",
      "to": "flowchart-B",
      "label": "承認後に実施",
      "type": "manual",
      "createdAt": "2026-03-21T10:30:00Z"
    },
    {
      "id": "mermaid-original-edge-1",
      "from": "flowchart-B",
      "to": "flowchart-C",
      "label": "",
      "type": "mermaid-original",
      "createdAt": null
    }
  ],

  "freeEdges": [
    {
      "id": "manual-edge-1710000001234",
      "startX": 150,
      "startY": 80,
      "startNodeId": null,
      "endX": 400,
      "endY": 250,
      "endNodeId": null,
      "label": "注釈矢印",
      "createdAt": "2026-03-21T10:32:00Z"
    }
  ],

  "statistics": {
    "totalNodes": 3,
    "totalConnections": 2,
    "manualConnections": 1,
    "mermaidConnections": 1,
    "freeEdges": 1
  }
}
```

### CSV出力形式（3ファイル）

#### 1. nodes.csv

```csv
node_id,node_type,label,display_text,has_custom_label,has_memo
flowchart-A,mermaid,開始,Phase 1開始,true,true
flowchart-B,mermaid,設計,設計,false,true
flowchart-C,mermaid,実装,実装,false,false
```

#### 2. connections.csv

```csv
edge_id,from_node,to_node,label,edge_type,created_at
manual-edge-1710000000000,flowchart-A,flowchart-B,承認後に実施,manual,2026-03-21T10:30:00Z
mermaid-original-edge-1,flowchart-B,flowchart-C,,mermaid-original,
```

#### 3. free_edges.csv

```csv
edge_id,start_x,start_y,start_node_id,end_x,end_y,end_node_id,label,created_at
manual-edge-1710000001234,150,80,,400,250,,注釈矢印,2026-03-21T10:32:00Z
```

## SVG DOM構造

### 手動エッジのSVG要素

```svg
<g class="manual-edge" data-id="manual-edge-1710000000000" data-et="manual-edge">
  <!-- 線要素 -->
  <line
    x1="100"
    y1="50"
    x2="300"
    y2="200"
    stroke="#666"
    stroke-width="2"
    stroke-dasharray="5,5"
    marker-end="url(#manual-edge-arrowhead)">
  </line>

  <!-- ラベル（ある場合のみ） -->
  <text
    class="manual-edge-label"
    x="200"
    y="125"
    text-anchor="middle"
    fill="#666"
    font-size="12px">
    承認後に実施
  </text>
</g>
```

### 矢印マーカー定義

```svg
<defs>
  <marker
    id="manual-edge-arrowhead"
    markerWidth="10"
    markerHeight="10"
    refX="9"
    refY="3"
    orient="auto">
    <polygon points="0,0 0,6 9,3" fill="#666" />
  </marker>
</defs>
```

## データ変換ロジック

### JavaScript → JSON

```javascript
function buildMemoData() {
  const elements = {};

  // 既存のメモ・ラベル情報を収集
  for (const elementId of allElementIds) {
    const element = document.querySelector(`[data-id="${elementId}"]`);
    const type = element?.getAttribute('data-et') || 'unknown';
    const displayText = getElementDisplayText(element);
    const memo = elementMemos[elementId] || '';
    const customLabel = elementCustomLabels[elementId] || '';

    elements[elementId] = {
      type: type,
      displayText: displayText,
      memo: memo,
      customLabel: customLabel,
      originalLabel: originalLabels[elementId] || ''
    };
  }

  return {
    flowchartId: currentSvgFileName?.replace('.svg', '') || 'untitled',
    svgFile: currentSvgFileName,
    savedAt: new Date().toISOString(),
    elements: elements,
    originalLabels: originalLabels,
    manualEdges: manualEdges  // ★追加
  };
}
```

### JSON → JavaScript

```javascript
async function loadFromFile() {
  // ... ファイル読み込み ...

  const data = JSON.parse(text);

  // 既存データの復元
  elementMemos = {};
  elementCustomLabels = {};
  originalLabels = data.originalLabels || {};

  for (const [elementId, elementData] of Object.entries(data.elements)) {
    if (elementData.memo) {
      elementMemos[elementId] = elementData.memo;
    }
    if (elementData.customLabel) {
      elementCustomLabels[elementId] = elementData.customLabel;
    }
  }

  // ★手動エッジの復元
  manualEdges = data.manualEdges || {};

  // SVGに再描画
  for (const edge of Object.values(manualEdges)) {
    drawEdgeToSvg(edge);
  }

  // ... 残りの処理 ...
}
```

### JavaScript → 接続グラフJSON（Phase 3）

```javascript
function extractConnectionGraph() {
  const nodes = new Map();
  const connections = [];
  const freeEdges = [];

  // 1. 全ノード情報を収集
  const allNodes = document.querySelectorAll('[data-et="node"], [data-et="cluster"]');
  allNodes.forEach(node => {
    const id = node.getAttribute('data-id') || node.id;
    const label = originalLabels[id] || getElementDisplayText(node);
    const customLabel = elementCustomLabels[id];

    nodes.set(id, {
      id: id,
      type: 'mermaid',
      label: label,
      displayText: customLabel || label,
      hasCustomLabel: !!customLabel,
      hasMemo: !!elementMemos[id]
    });
  });

  // 2. 手動エッジの接続関係を解析
  for (const edge of Object.values(manualEdges)) {
    if (edge.startNodeId && edge.endNodeId) {
      // 両端がノードに接続
      connections.push({
        id: edge.id,
        from: edge.startNodeId,
        to: edge.endNodeId,
        label: edge.label,
        type: 'manual',
        createdAt: edge.createdAt
      });
    } else {
      // 自由配置エッジ
      freeEdges.push({
        id: edge.id,
        startX: edge.startX,
        startY: edge.startY,
        startNodeId: edge.startNodeId,
        endX: edge.endX,
        endY: edge.endY,
        endNodeId: edge.endNodeId,
        label: edge.label,
        createdAt: edge.createdAt
      });
    }
  }

  return {
    flowchartId: currentSvgFileName?.replace('.svg', '') || 'untitled',
    exportedAt: new Date().toISOString(),
    svgFile: currentSvgFileName,
    nodes: Array.from(nodes.values()),
    connections: connections,
    freeEdges: freeEdges,
    statistics: {
      totalNodes: nodes.size,
      totalConnections: connections.length,
      manualConnections: connections.filter(c => c.type === 'manual').length,
      mermaidConnections: connections.filter(c => c.type === 'mermaid-original').length,
      freeEdges: freeEdges.length
    }
  };
}
```

## バリデーション

### エッジデータの検証

```javascript
function validateEdgeData(edge) {
  // 必須フィールド
  if (!edge.id) return false;
  if (typeof edge.startX !== 'number') return false;
  if (typeof edge.startY !== 'number') return false;
  if (typeof edge.endX !== 'number') return false;
  if (typeof edge.endY !== 'number') return false;

  // スタイル情報
  if (!edge.style || typeof edge.style !== 'object') return false;
  if (!edge.style.color) return false;
  if (typeof edge.style.width !== 'number') return false;

  return true;
}
```

## マイグレーション

### 既存JSONファイルへの対応

```javascript
// manualEdgesフィールドが存在しない古いJSONファイルを読み込んだ場合
function migrateOldFormat(data) {
  if (!data.manualEdges) {
    data.manualEdges = {};
  }
  return data;
}
```

---

## ステータス管理（Phase 4: Status Color Management）

### 概要

フローチャート要素（正規ノード・サブグラフ・メモノード）に対して、視覚的なステータス（完了・進行中・未着手等）を設定し、色分け表示する機能。

**重要**: このステータスは**フローチャート上の視覚的な状態管理**のみに使用され、task-manager.htmlの実タスクステータスとは完全に独立している。

### グローバル変数

```javascript
// ========== 新規追加の変数（Phase 4） ==========
let elementStatuses = {};  // { elementId: statusKey }
```

### elementStatuses の構造

```javascript
// 要素ごとのステータスマップ
elementStatuses = {
  'F_01': 'completed',         // 正規ノード（Mermaidノード）
  'F_02': 'in-progress',       // 正規ノード
  'subgraph_1': 'on-hold',     // サブグラフ
  'memo_node_1': 'postponed',  // メモノード
  'F_03': 'not-started'        // 正規ノード
};

// ステータス未設定の要素はキーが存在しない（undefinedまたはnull）
```

### ステータス定義（task-manager.htmlと同一）

```javascript
/**
 * ステータス色定義
 * task-manager.htmlのステータス仕様と完全同一
 */
const STATUS_COLORS = {
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
};
```

### JSON保存形式（v2.0）

#### ファイル構造（統合版 v2.0）

```json
{
  "version": "2.0",
  "exportedAt": "2026-03-22T12:00:00Z",
  "svgFile": "workflow.svg",

  "elements": {
    "flowchart-A": {
      "type": "node",
      "displayText": "開始",
      "memo": "プロジェクトキックオフ会議で決定",
      "customLabel": "Phase 1開始",
      "originalLabel": "開始"
    },
    "flowchart-B": {
      "type": "node",
      "displayText": "設計",
      "memo": "要件定義書を基に設計",
      "customLabel": "",
      "originalLabel": "設計"
    }
  },

  "originalLabels": {
    "flowchart-A": "開始",
    "flowchart-B": "設計",
    "flowchart-C": "実装"
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

  "elementStatuses": {
    "flowchart-A": "completed",
    "flowchart-B": "in-progress",
    "flowchart-C": "not-started",
    "subgraph_1": "on-hold",
    "memo_node_1": "postponed"
  }
}
```

#### elementStatuses フィールド仕様

| フィールド | 型 | 説明 | 必須 |
|-----------|---|------|------|
| `elementStatuses` | object | 要素IDとステータスキーのマップ | ✅ (v2.0以降) |
| キー | string | 要素ID（正規ノード・サブグラフ・メモノード） | ✅ |
| 値 | string | ステータスキー（`STATUS_COLORS`のキー） | ✅ |

**値の制約**:
- `not-started`, `in-progress`, `completed`, `on-hold`, `postponed`, `cancelled`, `unknown`のいずれか
- ステータス未設定の要素はキーが存在しない

### データ永続化

#### buildMemoData()への追加

```javascript
function buildMemoData() {
  const elements = {};

  // 既存のメモ・ラベル情報を収集
  for (const elementId of allElementIds) {
    const element = document.querySelector(`[data-id="${elementId}"]`);
    const type = element?.getAttribute('data-et') || 'unknown';
    const displayText = getElementDisplayText(element);
    const memo = elementMemos[elementId] || '';
    const customLabel = elementCustomLabels[elementId] || '';

    elements[elementId] = {
      type: type,
      displayText: displayText,
      memo: memo,
      customLabel: customLabel,
      originalLabel: originalLabels[elementId] || ''
    };
  }

  return {
    version: "2.0",                    // ★ v1.0 → v2.0にバージョンアップ
    flowchartId: currentSvgFileName?.replace('.svg', '') || 'untitled',
    svgFile: currentSvgFileName,
    savedAt: new Date().toISOString(),
    elements: elements,
    originalLabels: originalLabels,
    manualEdges: manualEdges,
    manualNodes: manualNodes,
    elementStatuses: elementStatuses   // ★ 新規追加（Phase 4）
  };
}
```

#### SVG埋め込みメタデータへの追加

```javascript
// exportSystemSvg()関数内
const metadata = {
  version: "2.0",                    // ★ v2.0
  exportedAt: new Date().toISOString(),
  svgFile: currentSvgFileName || 'unknown.svg',
  elements: {},
  originalLabels: originalLabels,
  manualNodes: manualNodes,
  manualEdges: manualEdges,
  elementStatuses: elementStatuses   // ★ 新規追加
};
```

#### 復元処理

```javascript
/**
 * SVG読み込み時にステータスを復元
 * @param {Object} metadata - 埋め込みメタデータ
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
```

### SVG要素へのスタイル適用

#### 正規ノード（Mermaidノード）

```javascript
/**
 * 正規ノードにステータス色を適用
 * @param {string} nodeId - ノードID
 * @param {string} status - ステータスキー
 */
function applyStatusToNode(nodeId, status) {
  const nodeGroup = svg.querySelector(`[data-id="${nodeId}"]`);
  if (!nodeGroup) return;

  const colorDef = STATUS_COLORS[status];
  if (!colorDef) return;

  // ノードの図形要素（rect, circle, polygon, path）
  const shape = nodeGroup.querySelector('rect, circle, polygon, path');
  if (shape) {
    shape.setAttribute('fill', colorDef.bg);
    shape.setAttribute('stroke', colorDef.border);
    shape.setAttribute('stroke-width', '2');
  }

  // テキスト要素
  const text = nodeGroup.querySelector('text');
  if (text) {
    text.setAttribute('fill', colorDef.text);
  }

  // data属性でステータスを記録
  nodeGroup.setAttribute('data-status', status);
}
```

#### サブグラフ

```javascript
/**
 * サブグラフにステータス色を適用（薄めの背景）
 * @param {string} clusterId - クラスタID
 * @param {string} status - ステータスキー
 */
function applyStatusToSubgraph(clusterId, status) {
  const cluster = svg.querySelector(`[data-cluster-id="${clusterId}"]`);
  if (!cluster) return;

  const colorDef = STATUS_COLORS[status];
  if (!colorDef) return;

  const rect = cluster.querySelector('rect');
  if (rect) {
    // サブグラフは背景を30%透明化（視認性確保）
    const fadedBg = adjustAlpha(colorDef.bg, 0.3);
    rect.setAttribute('fill', fadedBg);
    rect.setAttribute('stroke', colorDef.border);
    rect.setAttribute('stroke-width', '2');
  }

  cluster.setAttribute('data-status', status);
}

/**
 * 色の透明度を調整
 * @param {string} color - CSSカラー（#RRGGBB）
 * @param {number} alpha - 透明度（0.0-1.0）
 * @returns {string} rgba形式の色
 */
function adjustAlpha(color, alpha) {
  const r = parseInt(color.slice(1, 3), 16);
  const g = parseInt(color.slice(3, 5), 16);
  const b = parseInt(color.slice(5, 7), 16);
  return `rgba(${r}, ${g}, ${b}, ${alpha})`;
}
```

#### メモノード

```javascript
/**
 * メモノードにステータス色を適用
 * @param {string} memoId - メモノードID
 * @param {string} status - ステータスキー
 */
function applyStatusToMemoNode(memoId, status) {
  const memoNode = document.querySelector(`[data-memo-id="${memoId}"]`);
  if (!memoNode) return;

  const colorDef = STATUS_COLORS[status];
  if (!colorDef) return;

  const content = memoNode.querySelector('.memo-content');
  if (content) {
    content.style.background = colorDef.bg;
    content.style.borderColor = colorDef.border;
    content.style.color = colorDef.text;
  }

  memoNode.setAttribute('data-status', status);
}
```

### バージョン管理とマイグレーション

#### v1.0 → v2.0 マイグレーション

```javascript
/**
 * メタデータのバージョンマイグレーション
 * @param {Object} data - メタデータ
 * @returns {Object} マイグレーション後のデータ
 */
function migrateMetadata(data) {
  if (!data.version || data.version === "1.0") {
    console.log('[Migration] v1.0 → v2.0 マイグレーション開始');

    data.version = "2.0";

    // elementStatusesフィールドが存在しない場合は空で初期化
    if (!data.elementStatuses) {
      data.elementStatuses = {};
    }

    console.log('[Migration] ✅ マイグレーション完了');
  }

  return data;
}
```

#### バージョン互換性マトリクス

| バージョン | elementStatuses | 後方互換性 | 前方互換性 |
|-----------|----------------|-----------|-----------|
| v1.0 | なし | - | ✅ v2.0で読み込み可（空で初期化） |
| v2.0 | あり | ❌ v1.0で読み込み不可 | - |

**注意事項**:
- v2.0ファイルをv1.0実装で開くと`elementStatuses`が無視される
- v1.0ファイルをv2.0実装で開くと自動マイグレーションされる
- マイグレーション後は再保存でv2.0形式になる

### タスクステータスとの独立性（設計原則）

#### フローチャートステータス（このドキュメント）

```javascript
// flowchart-editor.html内で管理
elementStatuses = {
  'F_01': 'completed'  // フローチャート要素の視覚的な状態
};
```

**用途**:
- フローチャート図面上の視覚的な進捗表示
- 会議中のメモ書き・議論の可視化
- SVGファイルに埋め込んで保存

**保存場所**: SVGファイル内のメタデータコメント

---

#### タスクステータス（task-manager.html）

```javascript
// task-manager.html内で管理
task = {
  wbs_no: "WBS1.1.0",
  status: "進行中"  // 実タスクの進捗状態
};
```

**用途**:
- WBSタスクの実際の進捗管理
- プロジェクト管理・工数管理
- ガントチャート表示

**保存場所**: JSONファイル（`task-manager-data`）

---

#### 両者の関係

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

**最終更新**: 2026-03-22
**関連ドキュメント**: [README.md](./README.md), [02-manual-edge.md](./02-manual-edge.md), [06-implementation-plan.md](./06-implementation-plan.md)
