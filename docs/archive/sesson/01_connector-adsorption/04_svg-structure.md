# SVG構造

---
**ソース**: Mermaid.js自動生成SVG
**対象**: フローチャート（ガントチャート含む）

---

## SVG全体構造

```svg
<svg
  id="mermaid-svg"
  xmlns="http://www.w3.org/2000/svg"
  viewBox="0 0 [width] [height]"
  style="max-width: 100%;">

  <defs>
    <!-- Mermaid自動生成のマーカー定義 -->
    <marker id="flowchart-pointEnd" ...></marker>
    <marker id="flowchart-pointStart" ...></marker>

    <!-- ★手動エッジ用マーカー（追加） -->
    <marker id="manual-edge-arrowhead" ...></marker>
  </defs>

  <g class="output">
    <!-- フローチャート要素 -->

    <!-- ノード群 -->
    <g class="nodes">
      <g data-et="node" data-id="PHASE1" ...>...</g>
      <g data-et="node" data-id="PHASE2" ...>...</g>
      <g data-et="node" data-id="PHASE3" ...>...</g>
    </g>

    <!-- サブグラフ群 -->
    <g data-et="cluster" data-id="SCOPE" ...>
      <rect ...></rect>
      <text ...>SCOPE</text>
      <!-- サブグラフ内のノード -->
    </g>

    <!-- エッジ群 -->
    <g class="edgePaths">
      <g data-et="edge" data-id="L-PHASE1-PHASE2" ...>...</g>
      <g data-et="edge" data-id="L-PHASE2-PHASE3" ...>...</g>
    </g>

    <!-- ★手動エッジ（追加される予定） -->
    <g class="manual-edge" data-et="manual-edge" data-id="manual-edge-1234567890">
      <line ...></line>
      <text ...>test</text>
    </g>
  </g>
</svg>
```

---

## ノード構造（`data-et="node"`）

### 例: PHASE1ノード

```svg
<g
  data-et="node"
  data-id="PHASE1"
  id="flowchart-PHASE1"
  class="node default"
  transform="translate(1599.7, 848.0)">

  <rect
    class="basic label-container"
    x="-299"
    y="-429"
    width="598"
    height="858"
    rx="0"
    ry="0">
  </rect>

  <g class="label">
    <text ...>
      <tspan dy="1em" x="1">PHASE 1</tspan>
      <tspan dy="1em" x="1">開始: 2024-01-01</tspan>
      <tspan dy="1em" x="1">終了: 2024-03-31</tspan>
    </text>
  </g>
</g>
```

**特徴**:
- `data-et="node"` 属性でノードと識別
- `data-id` でユニークID
- `transform="translate(x, y)"` で中心座標が指定される
- `<rect>` のx, y はノードの中心からの相対座標（負の値）
- 実際のbbox:
  - origin: (1599.7 - 299, 848.0 - 429) = (1300.7, 419.0)
  - size: 598 x 858

---

## サブグラフ構造（`data-et="cluster"`）

### 例: SCOPEサブグラフ

```svg
<g
  data-et="cluster"
  data-id="SCOPE"
  id="flowchart-SCOPE"
  class="cluster default">

  <rect
    x="937.2"
    y="1510.0"
    width="869.0"
    height="803.0"
    rx="0"
    ry="0"
    fill="lightgray"
    stroke="gray"
    stroke-width="2">
  </rect>

  <text
    x="1371.7"
    y="1540.0"
    font-size="16"
    text-anchor="middle">
    SCOPE
  </text>

  <!-- サブグラフ内のノード -->
  <g data-et="node" data-id="SCOPE-TASK-1" ...>...</g>
  <g data-et="node" data-id="SCOPE-TASK-2" ...>...</g>
</g>
```

**特徴**:
- `data-et="cluster"` 属性でサブグラフと識別
- `<rect>` で境界領域を定義
- bbox:
  - origin: (937.2, 1510.0)
  - size: 869.0 x 803.0
  - center: (937.2 + 869.0/2, 1510.0 + 803.0/2) = (1371.7, 1911.5)
- サブグラフ内に子ノードを含む

---

## エッジ構造（`data-et="edge"`）

### 例: PHASE1 → PHASE2 のエッジ

```svg
<g
  data-et="edge"
  data-id="L-PHASE1-PHASE2"
  class="edge default">

  <path
    d="M 1599.7 1277 L 1599.7 1400 L 977.6 1400 L 977.6 1480"
    fill="none"
    stroke="black"
    stroke-width="2"
    marker-end="url(#flowchart-pointEnd)">
  </path>

  <g class="edgeLabel">
    <text ...>条件A</text>
  </g>
</g>
```

**特徴**:
- `data-et="edge"` 属性でエッジと識別
- `<path>` で曲線または折れ線を描画
- `marker-end` で矢印を表示

---

## 手動エッジ構造（Phase 2で追加予定）

### 期待される構造

```svg
<g
  class="manual-edge"
  data-et="manual-edge"
  data-id="manual-edge-1234567890">

  <!-- 透明なクリック領域（回転矩形） -->
  <rect
    class="edge-hit-area"
    x="..."
    y="..."
    width="..."
    height="20"
    fill="transparent"
    transform="rotate(...)"
    cursor="pointer">
  </rect>

  <!-- 表示用の線 -->
  <line
    x1="1371.7"
    y1="1911.5"
    x2="1468.7"
    y2="1626.8"
    stroke="#007bff"
    stroke-width="5"
    stroke-dasharray="5,5"
    marker-end="url(#manual-edge-arrowhead)"
    pointer-events="none">
  </line>

  <!-- ラベル -->
  <text
    class="manual-edge-label"
    x="1420.2"
    y="1769.15"
    text-anchor="middle"
    fill="#007bff"
    font-size="12px">
    test
  </text>
</g>
```

---

## ViewBox情報

### 例: 150%ズーム時

```svg
<svg viewBox="0 0 2500 5000" ...>
```

- viewBox width: 2500
- viewBox height: 5000
- Screen width (getBoundingClientRect): 3731 (150% zoom)
- Screen height: 7462

**スケール計算**:
```javascript
scaleX = viewBox.width / rect.width = 2500 / 3731 = 0.67
scaleY = viewBox.height / rect.height = 5000 / 7462 = 0.67
```

**座標変換**:
```javascript
// Screen → SVG
svgX = (screenX - rect.left) * scaleX + viewBox.x
svgY = (screenY - rect.top) * scaleY + viewBox.y

// Screen radius → SVG radius
svgRadius = screenRadius * scale = 1000 * 0.67 = 666.7
```

---

## DOM順序とZ-index

SVG内ではz-indexが使えず、**DOM順序が描画順**になる：

```svg
<svg>
  <g><!-- 最初に描画（下層） --></g>
  <g><!-- 次に描画 --></g>
  <g><!-- 最後に描画（最前面） --></g>
</svg>
```

**問題の可能性**:
- 手動エッジが他の要素の下に隠れている？
- `appendChild()` で追加した要素は最前面に来るはず
- しかし、親要素（`<g class="output">`等）の影響で隠れる可能性

---

## CSS適用

### Mermaid自動生成のCSS

```css
.node rect {
  fill: #ECECFF;
  stroke: #9370DB;
  stroke-width: 2px;
}

.cluster rect {
  fill: #ffffde;
  stroke: #aaaa33;
  stroke-width: 1px;
}

.edge path {
  stroke: #333;
  stroke-width: 2px;
}
```

### 手動エッジ用CSS（Phase 2で追加）

```css
/* ノード吸着時のハイライト */
.node-snap-highlight {
  filter: drop-shadow(0 0 8px #28a745) brightness(1.2);
  animation: pulse-snap 0.6s ease-in-out;
}

@keyframes pulse-snap {
  0%, 100% { filter: drop-shadow(0 0 8px #28a745) brightness(1.2); }
  50% { filter: drop-shadow(0 0 15px #28a745) brightness(1.4); }
}

/* 手動エッジのスタイル */
.manual-edge line {
  stroke: #007bff;
  stroke-width: 5;
  stroke-dasharray: 5,5;
}

.manual-edge-label {
  fill: #007bff;
  font-size: 12px;
  font-weight: bold;
}
```

---

## 問題点

### 1. サブグラフとノードの区別

- `data-et="node"` と `data-et="cluster"` で区別可能
- しかし、両方とも `getBBox()` で座標取得可能
- 中心距離だけでは優先度を判定できない

### 2. エッジの非表示

- `svg.appendChild(g)` は成功している（ログ確認済み）
- DOM inspector で確認する必要あり
- 考えられる原因:
  - viewBox範囲外
  - opacity: 0
  - マーカー未定義
  - 座標値がNaN

---

## 推奨される確認手順

1. **ブラウザの開発者ツールでDOM確認**:
   ```
   document.querySelectorAll('[data-et="manual-edge"]')
   ```

2. **追加された要素の属性確認**:
   ```javascript
   const edge = document.querySelector('[data-et="manual-edge"]');
   console.log(edge);
   console.log(edge.getBBox());
   ```

3. **計算された座標の確認**:
   ```javascript
   const line = edge.querySelector('line');
   console.log({
     x1: line.getAttribute('x1'),
     y1: line.getAttribute('y1'),
     x2: line.getAttribute('x2'),
     y2: line.getAttribute('y2')
   });
   ```

4. **マーカーの存在確認**:
   ```javascript
   const marker = document.querySelector('#manual-edge-arrowhead');
   console.log(marker);
   ```
