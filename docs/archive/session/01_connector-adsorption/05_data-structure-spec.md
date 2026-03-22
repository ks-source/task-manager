# データ構造仕様（抜粋）

---
**ソース**: `/docs/specs/flowchart-manual-editing/05-data-structure.md`
**バージョン**: 1.0
**更新日**: 2026-03-21

---

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

// ========== 新規追加の変数（Phase 1） ==========
let edgeAddMode = false;                  // エッジ追加モードON/OFF
let edgeStartPoint = null;                // 開始点情報（後述）
let edgePreviewLine = null;               // プレビュー線のSVG要素参照
let manualEdges = {};                     // 手動エッジのマップ
let manualEdgeCounter = 0;                // エッジID生成用カウンター（未使用）
```

---

## edgeStartPoint の構造

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

---

## manualEdges の構造

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

---

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

---

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

---

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
    manualEdges: manualEdges  // ★Phase 1で追加
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

  // ★手動エッジの復元（Phase 1で追加）
  manualEdges = data.manualEdges || {};

  // SVGに再描画
  for (const edge of Object.values(manualEdges)) {
    drawEdgeToSvg(edge);
  }

  // ... 残りの処理 ...
}
```

---

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

---

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

## Phase 2で追加されるデータ

### findNearestNode() の戻り値

```javascript
// ノード吸着に成功した場合
{
  nodeId: "flowchart-A",
  centerX: 150,
  centerY: 80,
  element: <g data-et="node" ...>,  // DOM要素への参照
  distance: 45.6                     // クリック位置からの距離（px）
}

// 吸着失敗（範囲内にノードなし）
null
```

### エッジデータに追加されるフィールド

```javascript
{
  id: "manual-edge-...",
  startX: 150,
  startY: 80,
  startNodeId: "flowchart-A",  // ★Phase 2で追加（吸着先ノード）
  endX: 300,
  endY: 200,
  endNodeId: "flowchart-B",    // ★Phase 2で追加（吸着先ノード）
  label: "test",
  style: { ... },
  createdAt: "..."
}
```

**Phase 1では**:
- `startNodeId`, `endNodeId` は常に `null`

**Phase 2では**:
- ノード付近をクリックした場合、ノードIDが設定される
- 自由位置をクリックした場合、`null` のまま

---

## 関連する型定義（TypeScript形式）

```typescript
interface ManualEdge {
  id: string;
  startX: number;
  startY: number;
  startNodeId: string | null;
  endX: number;
  endY: number;
  endNodeId: string | null;
  label: string;
  style: {
    color: string;
    width: number;
    dashArray: string;
  };
  createdAt: string;
}

interface EdgeStartPoint {
  x: number;
  y: number;
  nodeId: string | null;
}

interface NearestNodeResult {
  nodeId: string;
  centerX: number;
  centerY: number;
  element: SVGGElement;
  distance: number;
}
```

---

## 参考

- 完全な仕様: `/docs/specs/flowchart-manual-editing/05-data-structure.md`
- 全体設計: `/docs/specs/flowchart-manual-editing/01-overview.md`
- エッジ追加仕様: `/docs/specs/flowchart-manual-editing/02-manual-edge.md`
