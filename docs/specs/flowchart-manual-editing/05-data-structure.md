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

**最終更新**: 2026-03-21
**関連ドキュメント**: [README.md](./README.md), [02-manual-edge.md](./02-manual-edge.md)
