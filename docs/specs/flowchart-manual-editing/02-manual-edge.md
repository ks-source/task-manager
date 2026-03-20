# 手動エッジ追加仕様

---
**feature**: フローチャート手動編集
**document**: 手動エッジ追加詳細仕様
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

2点クリックで直線エッジを追加し、SVG DOMに描画する機能の詳細仕様。

## UI操作フロー

### 操作手順（アスキーアート図解）

```
ステップ1: ツールバーで「✏️ エッジ追加」ボタンをクリック
┌──────────────────────────────────────────────┐
│ [🔍+ 拡大] [🔍- 縮小] [✏️ エッジ追加] │ ← ボタン追加
└──────────────────────────────────────────────┘
         ↓
    モードが切り替わる
    カーソルが十字(+)に変化


ステップ2: SVG上の任意の場所をクリック（開始点）
        ┌─────────┐
        │  Node A │ ← クリック1（x: 100, y: 50）
        └─────────┘
             ●     ← 緑色の開始点マーカー表示

        「終点をクリックしてください...」← 状態表示


ステップ3: マウス移動中はプレビュー表示
        ┌─────────┐
        │  Node A │
        └─────────┘
             ●
             ┊\        ← 点線プレビュー（マウスに追従）
             ┊ \
             ┊  \
             ┊   ○  ← マウスカーソル位置


ステップ4: 別の場所をクリック（終了点）
        ┌─────────┐
        │  Node A │
        └─────────┘
             │
             │
             ▼     ← クリック2（x: 300, y: 200）
        ┌─────────┐
        │  Node B │
        └─────────┘

        ダイアログ表示
        ┌──────────────────────┐
        │ エッジラベル（任意）   │
        │ [承認後に実施  ] OK   │
        └──────────────────────┘


ステップ5: エッジが確定し描画
        ┌─────────┐
        │  Node A │
        └─────────┘
             │
             │  「承認後に実施」 ← ラベルテキスト
             ▼
        ┌─────────┐
        │  Node B │
        └─────────┘
```

### モード切り替え

```
通常モード（デフォルト）
→ ノードクリックで要素情報表示
→ SVGドラッグでスクロール

エッジ追加モード
→ カーソルが十字(crosshair)に変化
→ 1クリック目: 開始点設定
→ マウス移動: プレビュー線表示
→ 2クリック目: 終了点設定 → エッジ確定
→ 自動的に通常モードへ戻る
```

## SVG構造

### 生成されるSVG要素

```svg
<svg id="mermaid-svg" viewBox="0 0 800 600">
  <!-- 既存のMermaid要素 -->
  <g class="node" data-id="flowchart-A" data-et="node">
    <rect x="50" y="30" width="100" height="40" rx="5"></rect>
    <text class="nodeLabel" x="100" y="50">Node A</text>
  </g>

  <g class="node" data-id="flowchart-B" data-et="node">
    <rect x="250" y="180" width="100" height="40" rx="5"></rect>
    <text class="nodeLabel" x="300" y="200">Node B</text>
  </g>

  <!-- 矢印マーカー定義 -->
  <defs>
    <marker id="manual-edge-arrowhead"
            markerWidth="10" markerHeight="10"
            refX="9" refY="3"
            orient="auto">
      <polygon points="0,0 0,6 9,3" fill="#666" />
    </marker>
  </defs>

  <!-- ★新しく追加される手動エッジ -->
  <g class="manual-edge" data-id="manual-edge-1710000000000">
    <!-- メイン線 -->
    <line
      x1="100" y1="50"          <!-- 開始点座標 -->
      x2="300" y2="200"         <!-- 終了点座標 -->
      stroke="#666"
      stroke-width="2"
      stroke-dasharray="5,5"    <!-- 点線スタイル（メモ書き感） -->
      marker-end="url(#manual-edge-arrowhead)" />

    <!-- エッジラベル（中点に配置） -->
    <text
      x="200" y="125"           <!-- 中点座標 (x1+x2)/2, (y1+y2)/2 -->
      class="manual-edge-label"
      text-anchor="middle"
      fill="#666"
      font-size="12px">
      承認後に実施
    </text>
  </g>
</svg>
```

### CSS スタイル

```css
/* エッジ追加モード中のカーソル */
.svg-container.edge-add-mode {
  cursor: crosshair;
}

/* 開始点マーカー */
.edge-start-marker {
  fill: #28a745;
  stroke: white;
  stroke-width: 2;
  r: 6;
}

/* プレビュー線 */
.edge-preview-line {
  stroke: #007bff;
  stroke-width: 2;
  stroke-dasharray: 5,5;
  opacity: 0.6;
  pointer-events: none;
}

/* 手動エッジ */
.manual-edge {
  cursor: pointer;
}

.manual-edge:hover line {
  stroke: #007bff;
  stroke-width: 3;
}

.manual-edge-label {
  font-family: Arial, sans-serif;
  font-size: 12px;
  fill: #666;
  pointer-events: none;
}
```

## データ構造

### JavaScript内部データ

```javascript
// グローバル変数
let edgeAddMode = false;          // エッジ追加モードON/OFF
let edgeStartPoint = null;        // { x: number, y: number, nodeId: string|null }
let edgePreviewLine = null;       // SVG <line> 要素への参照
let manualEdges = {};             // 手動エッジのマップ
let manualEdgeCounter = 0;        // ID生成用カウンター

// 手動エッジのデータ構造
manualEdges = {
  "manual-edge-1710000000000": {
    id: "manual-edge-1710000000000",
    startX: 100,
    startY: 50,
    startNodeId: "flowchart-A",    // ノードに接続している場合
    endX: 300,
    endY: 200,
    endNodeId: "flowchart-B",
    label: "承認後に実施",
    style: {
      color: "#666",
      width: 2,
      dashArray: "5,5"
    },
    createdAt: "2026-03-21T10:30:00Z"
  },
  "manual-edge-1710000001234": {
    id: "manual-edge-1710000001234",
    startX: 150,
    startY: 80,
    startNodeId: null,              // 自由位置の場合はnull
    endX: 400,
    endY: 250,
    endNodeId: null,
    label: "",
    style: {
      color: "#666",
      width: 2,
      dashArray: "5,5"
    },
    createdAt: "2026-03-21T10:32:00Z"
  }
};
```

## 実装関数

### 1. モード切り替え

```javascript
/**
 * エッジ追加モードの切り替え
 */
function toggleEdgeAddMode() {
  edgeAddMode = !edgeAddMode;

  const btn = document.getElementById('edge-add-mode-btn');
  const container = document.getElementById('svg-container');

  if (edgeAddMode) {
    btn.classList.add('active');
    container.classList.add('edge-add-mode');
    showNotification('エッジ追加モード: 開始点をクリック');

    // 通常のノード選択を無効化
    disableNodeSelection();
  } else {
    btn.classList.remove('active');
    container.classList.remove('edge-add-mode');

    // 開始点マーカーとプレビューを削除
    clearEdgePreview();
    edgeStartPoint = null;

    // ノード選択を再有効化
    enableNodeSelection();
  }
}
```

### 2. クリック処理

```javascript
/**
 * SVGクリック処理（エッジ追加モード中）
 */
function onSvgClickForEdgeAdd(event) {
  if (!edgeAddMode) return;

  const svg = document.getElementById('mermaid-svg');
  const pt = svg.createSVGPoint();
  pt.x = event.clientX;
  pt.y = event.clientY;

  // スクリーン座標をSVG座標に変換
  const svgPoint = pt.matrixTransform(svg.getScreenCTM().inverse());

  if (!edgeStartPoint) {
    // 開始点を設定
    setEdgeStartPoint(svgPoint.x, svgPoint.y);
  } else {
    // 終了点を設定してエッジ作成
    createManualEdge(svgPoint.x, svgPoint.y);
  }
}

/**
 * 開始点を設定
 */
function setEdgeStartPoint(x, y) {
  edgeStartPoint = { x: x, y: y, nodeId: null };

  // Phase 2: ノード吸着機能（後で実装）
  // const nearNode = findNearestNode(x, y, 50);
  // if (nearNode) {
  //   edgeStartPoint.nodeId = nearNode.getAttribute('data-id');
  //   // ノード中心座標に調整
  // }

  // 開始点マーカーを表示
  showStartMarker(x, y);

  showNotification('終点をクリックしてください');
}

/**
 * 開始点マーカーを表示
 */
function showStartMarker(x, y) {
  const svg = document.getElementById('mermaid-svg');
  const marker = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
  marker.setAttribute('id', 'edge-start-marker');
  marker.setAttribute('class', 'edge-start-marker');
  marker.setAttribute('cx', x);
  marker.setAttribute('cy', y);
  marker.setAttribute('r', 6);
  svg.appendChild(marker);
}
```

### 3. プレビュー表示

```javascript
/**
 * マウス移動時のプレビュー線表示
 */
function onMouseMoveForPreview(event) {
  if (!edgeAddMode || !edgeStartPoint) return;

  const svg = document.getElementById('mermaid-svg');
  const pt = svg.createSVGPoint();
  pt.x = event.clientX;
  pt.y = event.clientY;
  const svgPoint = pt.matrixTransform(svg.getScreenCTM().inverse());

  if (!edgePreviewLine) {
    // プレビュー線を作成
    edgePreviewLine = document.createElementNS('http://www.w3.org/2000/svg', 'line');
    edgePreviewLine.setAttribute('class', 'edge-preview-line');
    edgePreviewLine.setAttribute('x1', edgeStartPoint.x);
    edgePreviewLine.setAttribute('y1', edgeStartPoint.y);
    svg.appendChild(edgePreviewLine);
  }

  // プレビュー線の終点を更新
  edgePreviewLine.setAttribute('x2', svgPoint.x);
  edgePreviewLine.setAttribute('y2', svgPoint.y);
}

/**
 * プレビューをクリア
 */
function clearEdgePreview() {
  const marker = document.getElementById('edge-start-marker');
  if (marker) marker.remove();

  if (edgePreviewLine) {
    edgePreviewLine.remove();
    edgePreviewLine = null;
  }
}
```

### 4. エッジ作成

```javascript
/**
 * 手動エッジを作成
 */
function createManualEdge(endX, endY) {
  // ラベル入力ダイアログ
  const label = prompt('エッジラベルを入力してください（省略可）:', '');

  if (label === null) {
    // キャンセル時
    clearEdgePreview();
    edgeStartPoint = null;
    return;
  }

  // エッジIDを生成
  const edgeId = `manual-edge-${Date.now()}`;

  // エッジデータを作成
  const edge = {
    id: edgeId,
    startX: edgeStartPoint.x,
    startY: edgeStartPoint.y,
    startNodeId: edgeStartPoint.nodeId,
    endX: endX,
    endY: endY,
    endNodeId: null,  // Phase 2で実装
    label: label.trim(),
    style: {
      color: '#666',
      width: 2,
      dashArray: '5,5'
    },
    createdAt: new Date().toISOString()
  };

  // 内部データに保存
  manualEdges[edgeId] = edge;

  // SVGに描画
  drawEdgeToSvg(edge);

  // プレビューをクリア
  clearEdgePreview();
  edgeStartPoint = null;

  // エッジ追加モードを解除
  toggleEdgeAddMode();

  // 自動保存
  saveToFileAuto();

  showNotification('エッジを追加しました');
}

/**
 * SVGにエッジを描画
 */
function drawEdgeToSvg(edge) {
  const svg = document.getElementById('mermaid-svg');

  // エッジグループを作成
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  g.setAttribute('class', 'manual-edge');
  g.setAttribute('data-id', edge.id);
  g.setAttribute('data-et', 'manual-edge');

  // 線を作成
  const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  line.setAttribute('x1', edge.startX);
  line.setAttribute('y1', edge.startY);
  line.setAttribute('x2', edge.endX);
  line.setAttribute('y2', edge.endY);
  line.setAttribute('stroke', edge.style.color);
  line.setAttribute('stroke-width', edge.style.width);
  line.setAttribute('stroke-dasharray', edge.style.dashArray);
  line.setAttribute('marker-end', 'url(#manual-edge-arrowhead)');

  g.appendChild(line);

  // ラベルを作成（ラベルがある場合）
  if (edge.label) {
    const midX = (edge.startX + edge.endX) / 2;
    const midY = (edge.startY + edge.endY) / 2;

    const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
    text.setAttribute('class', 'manual-edge-label');
    text.setAttribute('x', midX);
    text.setAttribute('y', midY);
    text.setAttribute('text-anchor', 'middle');
    text.textContent = edge.label;

    g.appendChild(text);
  }

  // クリックで削除できるようにする
  g.addEventListener('click', (e) => {
    if (!edgeAddMode && confirm('このエッジを削除しますか？')) {
      deleteManualEdge(edge.id);
    }
    e.stopPropagation();
  });

  svg.appendChild(g);
}
```

### 5. エッジ削除

```javascript
/**
 * 手動エッジを削除
 */
function deleteManualEdge(edgeId) {
  // 内部データから削除
  delete manualEdges[edgeId];

  // SVGから削除
  const edgeElement = document.querySelector(`[data-id="${edgeId}"]`);
  if (edgeElement) {
    edgeElement.remove();
  }

  // 自動保存
  saveToFileAuto();

  showNotification('エッジを削除しました');
}
```

## イベント登録

```javascript
/**
 * エッジ追加機能の初期化
 */
function setupManualEdgeFeature() {
  const container = document.getElementById('svg-container');

  // クリックイベント
  container.addEventListener('click', onSvgClickForEdgeAdd);

  // マウス移動イベント（プレビュー用）
  container.addEventListener('mousemove', onMouseMoveForPreview);

  // 矢印マーカーを定義
  createArrowMarkerDef();
}

/**
 * 矢印マーカーを定義
 */
function createArrowMarkerDef() {
  const svg = document.getElementById('mermaid-svg');
  let defs = svg.querySelector('defs');

  if (!defs) {
    defs = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
    svg.insertBefore(defs, svg.firstChild);
  }

  const marker = document.createElementNS('http://www.w3.org/2000/svg', 'marker');
  marker.setAttribute('id', 'manual-edge-arrowhead');
  marker.setAttribute('markerWidth', '10');
  marker.setAttribute('markerHeight', '10');
  marker.setAttribute('refX', '9');
  marker.setAttribute('refY', '3');
  marker.setAttribute('orient', 'auto');

  const polygon = document.createElementNS('http://www.w3.org/2000/svg', 'polygon');
  polygon.setAttribute('points', '0,0 0,6 9,3');
  polygon.setAttribute('fill', '#666');

  marker.appendChild(polygon);
  defs.appendChild(marker);
}
```

## JSON統合

### buildMemoData()への統合

```javascript
function buildMemoData() {
  // ... 既存のコード ...

  return {
    flowchartId: flowchartId,
    svgFile: svgFile,
    savedAt: new Date().toISOString(),
    elements: elements,           // 既存のメモ・ラベル
    originalLabels: originalLabels,
    manualEdges: manualEdges      // ★新規追加
  };
}
```

### loadFromFile()への統合

```javascript
async function loadFromFile() {
  // ... ファイル読み込み ...

  const data = JSON.parse(text);

  // 既存データの復元
  elementMemos = {};
  elementCustomLabels = {};

  // ★手動エッジの復元
  if (data.manualEdges) {
    manualEdges = data.manualEdges;

    // SVGに再描画
    for (const edge of Object.values(manualEdges)) {
      drawEdgeToSvg(edge);
    }
  }

  // ... 残りの処理 ...
}
```

---

**最終更新**: 2026-03-21
**関連ドキュメント**: [README.md](./README.md), [01-overview.md](./01-overview.md), [05-data-structure.md](./05-data-structure.md)
