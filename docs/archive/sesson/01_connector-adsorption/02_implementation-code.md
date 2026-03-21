# 実装コード全文

---
**ファイル**: `flowchart-mockup-interactive.html`
**セクション**: 手動エッジ追加機能（Phase 2）

---

## 主要関数

### 1. findNearestNode() - ノード検索・吸着判定

```javascript
/**
 * クリック位置から最も近いノードを検索
 * @param {number} x - SVG座標系のX座標
 * @param {number} y - SVG座標系のY座標
 * @param {number} radius - 吸着半径（デフォルト1000px）
 * @returns {Object|null} { nodeId, centerX, centerY, element, distance } または null
 */
function findNearestNode(x, y, radius = 1000) {
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) {
    console.warn('🚫 findNearestNode: SVG not found');
    return null;
  }

  // ★スケールに応じた閾値調整（SVG座標系での閾値を計算）
  const rect = svg.getBoundingClientRect();
  const viewBox = svg.viewBox.baseVal;
  const scaleX = viewBox.width / rect.width;
  const scaleY = viewBox.height / rect.height;
  const scale = Math.max(scaleX, scaleY);
  const adjustedRadius = radius * scale; // スクリーン上の1000px → SVG座標系での適切な距離

  console.log(`📏 Scale: ${scale.toFixed(2)}x, Screen radius: ${radius}px, SVG radius: ${adjustedRadius.toFixed(1)}px`);

  // ★問題箇所: サブグラフは除外している（ユーザーからの指摘で修正必要）
  const nodes = svg.querySelectorAll('[data-et="node"]');
  console.log(`🔍 findNearestNode: Found ${nodes.length} nodes with [data-et="node"]`);

  let nearestNode = null;
  let minDistance = Infinity;
  let closestDistances = [];

  nodes.forEach((node, index) => {
    try {
      // ノードの境界ボックスを取得
      const bbox = node.getBBox();

      // ノードの中心座標を計算
      const centerX = bbox.x + bbox.width / 2;
      const centerY = bbox.y + bbox.height / 2;

      // クリック位置との距離を計算（★中心距離で判定している）
      const dx = x - centerX;
      const dy = y - centerY;
      const distance = Math.sqrt(dx * dx + dy * dy);

      // デバッグ用
      if (index < 3 || distance <= radius) {
        closestDistances.push({
          nodeId: node.getAttribute('data-id') || node.id,
          distance: distance.toFixed(1),
          bbox: { x: bbox.x.toFixed(1), y: bbox.y.toFixed(1), w: bbox.width.toFixed(1), h: bbox.height.toFixed(1) },
          center: { x: centerX.toFixed(1), y: centerY.toFixed(1) }
        });
      }

      // ★最も近いノードを記録（問題: サイズや優先度を考慮していない）
      if (distance < minDistance && distance <= adjustedRadius) {
        minDistance = distance;
        nearestNode = {
          nodeId: node.getAttribute('data-id') || node.id,
          centerX: centerX,
          centerY: centerY,
          element: node,
          distance: distance
        };
      }
    } catch (err) {
      console.warn('getBBox failed for node:', node, err);
    }
  });

  // デバッグログ
  console.log(`📍 Click at (${x.toFixed(1)}, ${y.toFixed(1)}), screen radius: ${radius}px, adjusted: ${adjustedRadius.toFixed(1)}px`);
  if (closestDistances.length > 0) {
    console.log(`🎯 Top 5 closest nodes:`);
    closestDistances.slice(0, 5).forEach((n, i) => {
      console.log(`  ${i+1}. ${n.nodeId}: ${n.distance}px - center:(${n.center.x}, ${n.center.y}) bbox:(${n.bbox.x}, ${n.bbox.y}) ${n.bbox.w}x${n.bbox.h}`);
    });
  }

  if (nearestNode) {
    console.log(`🎯 ノード吸着: ${nearestNode.nodeId} (距離: ${nearestNode.distance.toFixed(1)}px)`);
  }

  return nearestNode;
}
```

---

### 2. highlightNodeForSnap() - ノードハイライト

```javascript
/**
 * ノードのハイライトをクリア
 */
function clearNodeSnapHighlight() {
  const highlighted = document.querySelectorAll('.node-snap-highlight');
  highlighted.forEach(el => el.classList.remove('node-snap-highlight'));
}

/**
 * ノードをハイライト表示（吸着時の視覚的フィードバック）
 */
function highlightNodeForSnap(nodeElement) {
  if (nodeElement) {
    clearNodeSnapHighlight();
    nodeElement.classList.add('node-snap-highlight');
  }
}
```

**CSS**:
```css
/* ノード吸着時のハイライト（Phase 2） */
.node-snap-highlight {
  filter: drop-shadow(0 0 8px #28a745) brightness(1.2);
  animation: pulse-snap 0.6s ease-in-out;
}

@keyframes pulse-snap {
  0%, 100% {
    filter: drop-shadow(0 0 8px #28a745) brightness(1.2);
  }
  50% {
    filter: drop-shadow(0 0 15px #28a745) brightness(1.4);
  }
}
```

---

### 3. createManualEdge() - エッジ作成

```javascript
/**
 * 手動エッジを作成
 * @param {number} endX - 終点X座標
 * @param {number} endY - 終点Y座標
 * @param {string|null} endNodeId - 終点ノードID（Phase 2で追加）
 */
function createManualEdge(endX, endY, endNodeId = null) {
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
    endNodeId: endNodeId, // ★Phase 2で実装完了
    label: label.trim(),
    style: {
      color: '#007bff',  // 青色（視認性向上）
      width: 5,          // 5px（視認性向上）
      dashArray: '5,5'
    },
    createdAt: new Date().toISOString()
  };

  // 内部データに保存
  manualEdges[edgeId] = edge;

  // ★SVGに描画（問題: 表示されない）
  drawEdgeToSvg(edge);

  // プレビューをクリア
  clearEdgePreview();
  edgeStartPoint = null;

  // エッジ追加モードを解除
  toggleEdgeAddMode();

  // 自動保存
  if (currentFileHandle) {
    saveToFileAuto().catch(err => {
      console.warn('自動保存に失敗:', err);
    });
  }

  showNotification('エッジを追加しました');
}
```

---

### 4. drawEdgeToSvg() - SVG描画

```javascript
/**
 * SVGにエッジを描画
 */
function drawEdgeToSvg(edge) {
  console.log('🎨 drawEdgeToSvg called:', edge);
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) {
    console.error('❌ SVG element not found for edge drawing');
    return;
  }
  console.log('✅ SVG element found:', svg);

  // エッジグループを作成
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  g.setAttribute('class', 'manual-edge');
  g.setAttribute('data-id', edge.id);
  g.setAttribute('data-et', 'manual-edge');

  // 線を作成（表示用）
  const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  line.setAttribute('x1', edge.startX);
  line.setAttribute('y1', edge.startY);
  line.setAttribute('x2', edge.endX);
  line.setAttribute('y2', edge.endY);
  line.setAttribute('stroke', edge.style.color);
  line.setAttribute('stroke-width', edge.style.width);
  line.setAttribute('stroke-dasharray', edge.style.dashArray);
  line.setAttribute('marker-end', 'url(#manual-edge-arrowhead)');
  line.setAttribute('pointer-events', 'none');

  g.appendChild(line);

  // ★矩形のクリック範囲を追加（Phase 1実装済み）
  createEdgeHitRect(g, line, edge.startX, edge.startY, edge.endX, edge.endY);

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

  // ★SVGに追加（成功しているがエッジが見えない）
  svg.appendChild(g);
  console.log(`✅ Edge ${edge.id} added to SVG:`, {
    start: `(${edge.startX}, ${edge.startY})`,
    end: `(${edge.endX}, ${edge.endY})`,
    label: edge.label
  });
}
```

---

### 5. createArrowMarkerDef() - 矢印マーカー定義

```javascript
/**
 * 矢印マーカーを定義
 */
function createArrowMarkerDef() {
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) return;

  let defs = svg.querySelector('defs');
  if (!defs) {
    defs = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
    svg.insertBefore(defs, svg.firstChild);
  }

  // 既に存在する場合は作成しない
  if (defs.querySelector('#manual-edge-arrowhead')) return;

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

---

### 6. onSvgClickForEdgeAdd() - クリックイベントハンドラ

```javascript
/**
 * SVGクリック時の処理（エッジ追加用）
 */
function onSvgClickForEdgeAdd(event) {
  if (!edgeAddMode) return;

  console.log('🖱️ onSvgClickForEdgeAdd called, edgeAddMode:', edgeAddMode);

  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) return;

  // クリック位置をSVG座標系に変換
  const rect = svg.getBoundingClientRect();
  const viewBox = svg.viewBox.baseVal;

  const scaleX = viewBox.width / rect.width;
  const scaleY = viewBox.height / rect.height;

  const x = (event.clientX - rect.left) * scaleX + viewBox.x;
  const y = (event.clientY - rect.top) * scaleY + viewBox.y;

  console.log('Click:', { x, y, edgeStartPoint });

  if (!edgeStartPoint) {
    // 開始点を設定
    // ★ノード吸着判定（Phase 2）
    const nearestNode = findNearestNode(x, y);

    if (nearestNode) {
      // ノードに吸着
      edgeStartPoint = {
        x: nearestNode.centerX,
        y: nearestNode.centerY,
        nodeId: nearestNode.nodeId
      };
      showStartMarker(nearestNode.centerX, nearestNode.centerY);
      highlightNodeForSnap(nearestNode.element); // ★ビジュアルフィードバック
      showNotification(`ノード "${nearestNode.nodeId}" から開始 - 終点をクリックしてください`);
    } else {
      // 自由位置
      edgeStartPoint = { x: x, y: y, nodeId: null };
      showStartMarker(x, y);
      clearNodeSnapHighlight(); // ★ハイライトクリア
      showNotification('終点をクリックしてください');
    }
  } else {
    // 終了点を設定してエッジ作成
    // ★ノード吸着判定（Phase 2）
    const nearestNode = findNearestNode(x, y);

    if (nearestNode) {
      // ノードに吸着
      highlightNodeForSnap(nearestNode.element); // ★ビジュアルフィードバック
      createManualEdge(nearestNode.centerX, nearestNode.centerY, nearestNode.nodeId);
    } else {
      // 自由位置
      createManualEdge(x, y, null);
    }
  }
}
```

---

## グローバル変数

```javascript
// 既存の変数
let selectedElement = null;
let elementMemos = {};
let elementCustomLabels = {};
let originalLabels = {};
let autoSaveTimer = null;
let currentFileHandle = null;
let currentSvgFileName = null;

// Phase 1で追加
let edgeAddMode = false;
let edgeStartPoint = null;
let edgePreviewLine = null;
let manualEdges = {};

// Phase 2では新規変数なし
```

---

## 初期化

```javascript
/**
 * 手動エッジ機能の初期化
 */
function setupManualEdgeFeature() {
  const container = document.getElementById('flowchart-container');
  if (!container) return;

  // クリックイベント
  container.addEventListener('click', onSvgClickForEdgeAdd);

  // マウス移動イベント（プレビュー用）
  container.addEventListener('mousemove', onMouseMoveForPreview);

  // 矢印マーカーを定義
  createArrowMarkerDef();
}

// SVG読み込み時に呼び出される
// loadSVGFile() 内で setupManualEdgeFeature(); が実行される
```

---

## 問題箇所まとめ

1. **findNearestNode() 5635行目**:
   ```javascript
   const nodes = svg.querySelectorAll('[data-et="node"]');
   ```
   → サブグラフを除外しているが、ユーザーは両方を対象にしたい

2. **findNearestNode() 5692-5701行目**:
   ```javascript
   if (distance < minDistance && distance <= adjustedRadius) {
     minDistance = distance;
     nearestNode = { ... };
   }
   ```
   → 中心距離のみで判定、優先度やサイズを考慮していない

3. **drawEdgeToSvg() 5845行目**:
   ```javascript
   svg.appendChild(g);
   ```
   → 実行されているがエッジが表示されない（原因不明）
