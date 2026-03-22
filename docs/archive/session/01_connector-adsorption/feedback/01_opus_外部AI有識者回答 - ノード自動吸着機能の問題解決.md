
# 外部AI有識者 回答書

---
**プロジェクト**: タスク管理ツール - フローチャート手動編集機能  
**対象**: Phase 2 - ノード自動吸着（Node Auto-Snap）  
**回答日**: 2026-03-21  
**回答者**: Claude（外部AI有識者）

---

## 問題1への回答: サブグラフが個別ノードより優先的に吸着される

### 原因分析

現在の `findNearestNode()` は**ユークリッド距離（中心点間距離）のみ**で最寄りノードを判定しています。この方式には構造的な欠陥があります。

**根本原因: 中心距離は「ユーザーが視覚的にどの要素を指しているか」を正しく表現できない**

```
クリック位置 (1257.3, 1898.1) の場合：

┌─────────────────────────────────────┐
│  SCOPEサブグラフ (869x803)          │
│  bbox: (937.2, 1510.0)             │
│                                     │
│      ★中心(1371.7, 1911.5)         │  ← クリック位置からわずか 299.5px
│                                     │
│  ● ← クリック位置はSCOPE内部!       │
│                                     │
└─────────────────────────────────────┘

                    PHASE1ノード (598x858)
                    ★中心(1599.7, 848.0)      ← クリック位置から 1104.5px
```

- クリック位置 `(1257.3, 1898.1)` は **SCOPE の bbox 内部**に位置している（`937.2 ≤ 1257.3 ≤ 1806.2`, `1510.0 ≤ 1898.1 ≤ 2313.0`）
- サブグラフは広い領域を占めるため、その内部でクリックすると中心距離が常に小さくなる
- 個別ノードは面積が小さく、内部に居ない限り中心距離が大きくなりがち

### 推奨解決策: 二段階フォールバック方式（Two-Pass with Priority）

**戦略**: 個別ノードを第一候補として検索し、見つからない場合のみサブグラフを候補にする。距離計算は「境界距離」を採用する。

```javascript
/**
 * クリック位置から最も近いノードを検索（改良版）
 * 
 * アルゴリズム:
 *   Pass 1: 個別ノード（data-et="node"）を境界距離で検索
 *   Pass 2: Pass 1で見つからない場合、サブグラフ（data-et="cluster"）を境界距離で検索
 * 
 * @param {number} x - SVG座標系のX座標
 * @param {number} y - SVG座標系のY座標
 * @param {number} radius - 吸着半径（デフォルト1000px、スクリーン座標）
 * @returns {Object|null} { nodeId, centerX, centerY, element, distance } または null
 */
function findNearestNode(x, y, radius = 1000) {
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) {
    console.warn('🚫 findNearestNode: SVG not found');
    return null;
  }

  // スケールに応じた閾値調整
  const rect = svg.getBoundingClientRect();
  const viewBox = svg.viewBox.baseVal;
  const scaleX = viewBox.width / rect.width;
  const scaleY = viewBox.height / rect.height;
  const scale = Math.max(scaleX, scaleY);
  const adjustedRadius = radius * scale;

  console.log(`📏 Scale: ${scale.toFixed(2)}x, Screen radius: ${radius}px, SVG radius: ${adjustedRadius.toFixed(1)}px`);

  // ======== Pass 1: 個別ノードを優先検索 ========
  const individualNodes = svg.querySelectorAll('[data-et="node"]');
  console.log(`🔍 Pass 1: Found ${individualNodes.length} individual nodes`);
  
  let result = findNearestInSet(individualNodes, x, y, adjustedRadius, 'Pass1-Node');

  // ======== Pass 2: 個別ノードが見つからなければサブグラフを検索 ========
  if (!result) {
    const clusters = svg.querySelectorAll('[data-et="cluster"]');
    console.log(`🔍 Pass 2: Found ${clusters.length} clusters (fallback)`);
    result = findNearestInSet(clusters, x, y, adjustedRadius, 'Pass2-Cluster');
  }

  if (result) {
    console.log(`🎯 ノード吸着: ${result.nodeId} (境界距離: ${result.distance.toFixed(1)}px)`);
  } else {
    console.log(`❌ 吸着対象なし (半径: ${adjustedRadius.toFixed(1)}px)`);
  }

  return result;
}

/**
 * ノード集合の中から最寄りのノードを「境界距離」で検索
 * 
 * @param {NodeList} nodes - 検索対象のSVGノード群
 * @param {number} x - クリック位置X
 * @param {number} y - クリック位置Y
 * @param {number} maxDistance - 最大吸着距離（SVG座標系）
 * @param {string} passLabel - デバッグ用ラベル
 * @returns {Object|null}
 */
function findNearestInSet(nodes, x, y, maxDistance, passLabel) {
  let nearestNode = null;
  let minDistance = Infinity;
  let debugList = [];

  nodes.forEach((node) => {
    try {
      const bbox = node.getBBox();
      const centerX = bbox.x + bbox.width / 2;
      const centerY = bbox.y + bbox.height / 2;

      // ★核心: 境界距離（Boundary Distance）を計算
      const distance = calcBoundaryDistance(x, y, bbox);

      debugList.push({
        nodeId: node.getAttribute('data-id') || node.id,
        distance: distance.toFixed(1),
        centerDistance: Math.sqrt((x - centerX) ** 2 + (y - centerY) ** 2).toFixed(1),
        bbox: `(${bbox.x.toFixed(0)},${bbox.y.toFixed(0)}) ${bbox.width.toFixed(0)}x${bbox.height.toFixed(0)}`,
        inside: distance === 0 ? '★INSIDE' : ''
      });

      if (distance < minDistance && distance <= maxDistance) {
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

  // デバッグログ（距離順でソート、上位5件表示）
  debugList.sort((a, b) => parseFloat(a.distance) - parseFloat(b.distance));
  if (debugList.length > 0) {
    console.log(`📍 [${passLabel}] Top candidates (boundary distance):`);
    debugList.slice(0, 5).forEach((n, i) => {
      console.log(`  ${i + 1}. ${n.nodeId}: boundary=${n.distance}px, center=${n.centerDistance}px, bbox=${n.bbox} ${n.inside}`);
    });
  }

  return nearestNode;
}

/**
 * クリック位置からbbox境界までの最短距離を計算
 * 
 * - クリックがbbox内部にある場合: 0を返す
 * - クリックがbbox外部にある場合: bbox辺までの最短ユークリッド距離を返す
 * 
 * @param {number} px - クリック位置X
 * @param {number} py - クリック位置Y
 * @param {SVGRect} bbox - ノードの境界ボックス
 * @returns {number} 境界距離（0以上）
 */
function calcBoundaryDistance(px, py, bbox) {
  // bbox の左上・右下座標
  const left   = bbox.x;
  const top    = bbox.y;
  const right  = bbox.x + bbox.width;
  const bottom = bbox.y + bbox.height;

  // X軸方向の最短距離
  const dx = Math.max(left - px, 0, px - right);
  // Y軸方向の最短距離
  const dy = Math.max(top - py, 0, py - bottom);

  // ユークリッド距離（内部なら dx=0, dy=0 → distance=0）
  return Math.sqrt(dx * dx + dy * dy);
}
```

### アルゴリズム解説

#### 1. 境界距離（Boundary Distance）とは

中心距離ではなく、**クリック位置からbbox（境界矩形）の最も近い辺までの距離**を使います。

```
ケース1: クリックがbbox外部
  ┌──────────┐
  │   node   │
  │          │
  └──────────┘
        ↖ dx,dy → distance > 0

ケース2: クリックがbbox内部
  ┌──────────┐
  │  ●click  │  → distance = 0（内部）
  │          │
  └──────────┘
```

**なぜ境界距離が有効か？**

| シナリオ | 中心距離 | 境界距離 |
|---------|---------|---------|
| SCOPEの内部でクリック | 299.5px（中心まで） | **0px**（内部だから） |
| PHASE1の境界から100pxの位置でクリック | 1104.5px（中心まで） | **100px**（境界まで） |

- 中心距離: SCOPE(299.5) < PHASE1(1104.5) → SCOPEが選ばれる ❌
- 境界距離: SCOPE(0) < PHASE1(100) → 両方候補になるが…

**境界距離だけでは不十分**：SCOPEの内部にいるので境界距離は0になり、やはりSCOPEが選ばれてしまいます。そこで**二段階フォールバック**を組み合わせます。

#### 2. 二段階フォールバック（Two-Pass with Priority）

```
Pass 1: 個別ノード（data-et="node"）のみを境界距離で検索
  → PHASE1, PHASE2, PHASE3, SCOPE-TASK-1, SCOPE-TASK-2 等
  → 最寄りの個別ノードが見つかれば、それを返す

Pass 2: Pass 1で見つからない場合のみ、サブグラフ（data-et="cluster"）を検索
  → SCOPE 等
  → サブグラフしか候補がない場合のフォールバック
```

**これが最適な理由**：

1. **ユーザーの意図に合致**: 個別ノードをクリックしたつもりなら個別ノードが選ばれる
2. **サブグラフも対象**: 個別ノードが遠い場所では、サブグラフに吸着できる
3. **シンプル**: ノードタイプ（`data-et`属性）だけで判定でき、面積計算等のヒューリスティックが不要
4. **境界距離との組み合わせ**: 大きなノードでも公平に距離が計算される

#### 3. 具体的なシナリオでの動作

```
シナリオA: SCOPE内部でPHASE1付近をクリック
  Pass 1: 個別ノードを検索
    - PHASE1 境界距離: 100px → ✅ 範囲内 → PHASE1に吸着
  Pass 2: 実行されない

シナリオB: サブグラフSCOPEの中心付近をクリック（個別ノードから遠い）
  Pass 1: 個別ノードを検索
    - PHASE1 境界距離: 800px → 範囲内だが遠い
    - SCOPE-TASK-1 境界距離: 200px → ✅ 最寄りの個別ノード
  Pass 2: 実行されない（SCOPE-TASK-1が見つかったため）

シナリオC: フローチャートの空白領域をクリック（全ノードから遠い）
  Pass 1: 個別ノードを検索 → 全て範囲外 → null
  Pass 2: サブグラフを検索
    - SCOPE 境界距離: 300px → ✅ 範囲内 → SCOPEに吸着

シナリオD: 完全な空白領域（全要素から遠い）
  Pass 1: null
  Pass 2: null → 吸着なし（自由位置）
```

### 代替案

#### 代替案A: 重み付き距離方式（シンプルだが調整が難しい）

サブグラフにペナルティを加算する方式です。

```javascript
function findNearestNode(x, y, radius = 1000) {
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) return null;

  const rect = svg.getBoundingClientRect();
  const viewBox = svg.viewBox.baseVal;
  const scale = Math.max(viewBox.width / rect.width, viewBox.height / rect.height);
  const adjustedRadius = radius * scale;

  // 全ノード（個別ノード + サブグラフ）を取得
  const allNodes = svg.querySelectorAll('[data-et="node"], [data-et="cluster"]');
  
  let nearestNode = null;
  let minWeightedDistance = Infinity;

  allNodes.forEach((node) => {
    try {
      const bbox = node.getBBox();
      const centerX = bbox.x + bbox.width / 2;
      const centerY = bbox.y + bbox.height / 2;
      const isCluster = node.getAttribute('data-et') === 'cluster';

      // 境界距離を計算
      const boundaryDist = calcBoundaryDistance(x, y, bbox);

      // ★重み付け: サブグラフには距離ペナルティを加算
      // clusterの場合、実際の距離にペナルティ（200px）を加算
      const CLUSTER_PENALTY = 200;
      const weightedDistance = isCluster 
        ? boundaryDist + CLUSTER_PENALTY 
        : boundaryDist;

      if (weightedDistance < minWeightedDistance && boundaryDist <= adjustedRadius) {
        minWeightedDistance = weightedDistance;
        nearestNode = {
          nodeId: node.getAttribute('data-id') || node.id,
          centerX: centerX,
          centerY: centerY,
          element: node,
          distance: boundaryDist // 実際の距離を記録
        };
      }
    } catch (err) {
      console.warn('getBBox failed:', err);
    }
  });

  return nearestNode;
}
```

**メリット**: 一度の走査で完結する  
**デメリット**: `CLUSTER_PENALTY` の値を環境に応じて調整する必要がある。ペナルティが小さすぎるとサブグラフが選ばれ、大きすぎるとサブグラフに一切吸着できなくなる

#### 代替案B: 「クリック位置がbbox内部かどうか」で判定

```javascript
function findNearestNode(x, y, radius = 1000) {
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) return null;

  const rect = svg.getBoundingClientRect();
  const viewBox = svg.viewBox.baseVal;
  const scale = Math.max(viewBox.width / rect.width, viewBox.height / rect.height);
  const adjustedRadius = radius * scale;

  const allNodes = svg.querySelectorAll('[data-et="node"], [data-et="cluster"]');
  
  // 候補を分類
  let insideNodes = [];   // bbox内部にクリックがあるノード
  let nearbyNodes = [];   // bbox外部だが範囲内のノード

  allNodes.forEach((node) => {
    try {
      const bbox = node.getBBox();
      const centerX = bbox.x + bbox.width / 2;
      const centerY = bbox.y + bbox.height / 2;
      const boundaryDist = calcBoundaryDistance(x, y, bbox);
      const isCluster = node.getAttribute('data-et') === 'cluster';

      const candidate = {
        nodeId: node.getAttribute('data-id') || node.id,
        centerX, centerY, element: node,
        distance: boundaryDist,
        isCluster,
        area: bbox.width * bbox.height
      };

      if (boundaryDist === 0) {
        insideNodes.push(candidate);
      } else if (boundaryDist <= adjustedRadius) {
        nearbyNodes.push(candidate);
      }
    } catch (err) {}
  });

  // 内部ノードがある場合: 面積が最小のものを選ぶ（最も具体的なノード）
  if (insideNodes.length > 0) {
    // 個別ノードを優先し、同カテゴリ内では面積最小を選ぶ
    const insideIndividual = insideNodes.filter(n => !n.isCluster);
    if (insideIndividual.length > 0) {
      insideIndividual.sort((a, b) => a.area - b.area);
      return insideIndividual[0];
    }
    // 個別ノードがなければサブグラフの中で面積最小を選ぶ
    insideNodes.sort((a, b) => a.area - b.area);
    return insideNodes[0];
  }

  // 内部ノードがない場合: 境界距離が最小のものを選ぶ（個別ノード優先）
  if (nearbyNodes.length > 0) {
    const nearbyIndividual = nearbyNodes.filter(n => !n.isCluster);
    if (nearbyIndividual.length > 0) {
      nearbyIndividual.sort((a, b) => a.distance - b.distance);
      return nearbyIndividual[0];
    }
    nearbyNodes.sort((a, b) => a.distance - b.distance);
    return nearbyNodes[0];
  }

  return null;
}
```

**メリット**: 最も正確。サブグラフ内部の個別ノードを的確に選べる  
**デメリット**: コードが複雑になる

### 推奨順位

1. **推奨**: 二段階フォールバック方式（シンプルかつ効果的）
2. **代替**: 代替案B（精度重視の場合）
3. **非推奨**: 代替案A（ペナルティ値の調整が困難）

---

## 問題2への回答: エッジとラベルがSVGに表示されない

### 原因分析

提供されたコードとSVG構造を詳細に分析した結果、**最も可能性が高い原因は「SVGの追加先の問題」**です。以下の複数の原因が複合的に絡んでいる可能性があります。

#### 原因A（最有力）: `svg.appendChild()` の追加先が間違っている

Mermaid.jsが生成するSVGには、`<g class="output">` という**ラッパーグループ**があり、すべてのフローチャート要素はこの中に配置されます。

```xml
<svg id="mermaid-svg" viewBox="0 0 2500 5000">
  <g class="output">                    ← ★ この中にすべてのコンテンツ
    <g transform="translate(...)">       ← ★ 変換が適用されている可能性
      <g class="nodes">...</g>
      <g class="edgePaths">...</g>
      <g data-et="cluster">...</g>
    </g>
  </g>
</svg>
```

現在のコードでは `svg.appendChild(g)` で**SVGルート直下**に追加しています。しかし：

1. **`<g class="output">` の中に `transform` 属性**（`translate`, `scale`）が設定されている可能性がある
2. SVGルート直下に追加した要素は、この `transform` の影響を受けない
3. 結果として、**座標系がずれて画面外に描画**されている

**具体例**：
```xml
<svg viewBox="0 0 2500 5000">
  <g class="output">
    <g transform="translate(100, 50) scale(0.8)">    ← ★ これが問題
      <!-- ノードの座標はこのtransformの中で定義されている -->
      <g data-et="node" data-id="PHASE1" transform="translate(1599.7, 848.0)">
        ...
      </g>
    </g>
  </g>
  
  <!-- ★ ここに追加される → transformの影響を受けない → 座標がずれる -->
  <g class="manual-edge" data-id="manual-edge-xxx">
    <line x1="1371.7" y1="1911.5" x2="1468.7" y2="1626.8" .../>
  </g>
</svg>
```

#### 原因B: Mermaidの `viewBox` が `style` 属性で `overflow: hidden` 相当になっている

SVGの `viewBox` 外の要素は表示されません。Mermaidは `viewBox` を自動計算するため、手動追加要素の座標が正しくても、`viewBox` の設定次第で見えないことがあります。

#### 原因C: `marker-end` のレンダリングエラー

`marker-end="url(#manual-edge-arrowhead)"` で参照されるマーカーが、**SVGの再レンダリングタイミングによっては見つからない**場合があります。これにより、一部ブラウザでは線全体が非表示になることがあります（特にEdge）。

ただし通常は、マーカーが見つからなくても線自体は描画されるため、これが主因である可能性は低いです。

### デバッグ手順

以下の手順で原因を特定してください。**上から順に実行し、各ステップの結果をコンソールで確認**します。

#### ステップ1: DOM上に要素が存在するか確認

```javascript
// ブラウザの開発者ツールコンソールで実行
const manualEdges = document.querySelectorAll('[data-et="manual-edge"]');
console.log('手動エッジ数:', manualEdges.length);
manualEdges.forEach(edge => {
  console.log('Edge:', edge.getAttribute('data-id'));
  console.log('  Parent:', edge.parentElement?.tagName, edge.parentElement?.className);
  console.log('  Line:', edge.querySelector('line')?.outerHTML);
  console.log('  Text:', edge.querySelector('text')?.outerHTML);
  try {
    const bbox = edge.getBBox();
    console.log('  BBox:', bbox.x, bbox.y, bbox.width, bbox.height);
  } catch(e) {
    console.log('  BBox: ERROR', e.message);
  }
});
```

#### ステップ2: 親要素の transform チェーン確認

```javascript
// Mermaidコンテンツの実際のtransformを確認
const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
const output = svg.querySelector('.output') || svg.querySelector('g');
console.log('SVG viewBox:', svg.getAttribute('viewBox'));
console.log('output transform:', output?.getAttribute('transform'));

// outputの子要素のtransformも確認
if (output) {
  Array.from(output.children).forEach(child => {
    console.log('  child:', child.tagName, child.className, 
                'transform:', child.getAttribute('transform'));
  });
}

// 手動エッジの親を確認
const edge = document.querySelector('[data-et="manual-edge"]');
if (edge) {
  let parent = edge.parentElement;
  let chain = [];
  while (parent && parent !== document.body) {
    chain.push({
      tag: parent.tagName,
      class: parent.className?.baseVal || parent.className,
      transform: parent.getAttribute('transform'),
      viewBox: parent.getAttribute('viewBox')
    });
    parent = parent.parentElement;
  }
  console.log('Parent chain:', JSON.stringify(chain, null, 2));
}
```

#### ステップ3: 手動でテスト要素を追加して表示確認

```javascript
// SVGルート直下にテスト用の赤い円を追加
const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
circle.setAttribute('cx', '100');
circle.setAttribute('cy', '100');
circle.setAttribute('r', '50');
circle.setAttribute('fill', 'red');
circle.setAttribute('id', 'test-circle-root');
svg.appendChild(circle);
console.log('赤い円をSVGルートに追加 (100,100)');

// output/g の中にも追加
const output = svg.querySelector('.output > g') || svg.querySelector('.output') || svg.querySelector('g');
if (output) {
  const circle2 = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
  circle2.setAttribute('cx', '100');
  circle2.setAttribute('cy', '100');
  circle2.setAttribute('r', '50');
  circle2.setAttribute('fill', 'blue');
  circle2.setAttribute('id', 'test-circle-output');
  output.appendChild(circle2);
  console.log('青い円をoutput内に追加 (100,100)');
}

// → 赤い円だけ見えて青い円が見えない場合: outputにtransformがある
// → 青い円だけ見えて赤い円が見えない場合: SVGルートに問題がある
// → 両方見えるがフローチャートと異なる場所に表示: 座標系のズレ
// → 両方見えない場合: SVG自体に問題がある
```

#### ステップ4: 既存ノードと同じ親に追加してテスト

```javascript
// 既存のPHASE1ノードと同じ親に赤い円を追加
const phase1 = document.querySelector('[data-id="PHASE1"]');
if (phase1) {
  const parent = phase1.parentElement;
  console.log('PHASE1 parent:', parent.tagName, parent.className);
  
  const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
  circle.setAttribute('cx', '1600');
  circle.setAttribute('cy', '848');
  circle.setAttribute('r', '30');
  circle.setAttribute('fill', 'lime');
  circle.setAttribute('stroke', 'black');
  circle.setAttribute('stroke-width', '3');
  parent.appendChild(circle);
  console.log('緑の円をPHASE1と同じ親に追加 (1600,848)');
}
// → 緑の円がPHASE1の上に見えれば、この親が正しい追加先
```

### 推奨解決策

#### 修正版 `drawEdgeToSvg()` — 正しい親要素に追加する

```javascript
/**
 * SVGにエッジを描画（修正版）
 * 
 * 修正点:
 * 1. SVGルートではなく、フローチャートコンテンツと同じ親グループに追加
 * 2. マーカー未定義時のフォールバック
 * 3. デバッグ用の詳細ログ
 */
function drawEdgeToSvg(edge) {
  console.log('🎨 drawEdgeToSvg called:', edge);
  
  const svg = document.querySelector('.svg-content-wrapper svg') || document.querySelector('svg');
  if (!svg) {
    console.error('❌ SVG element not found for edge drawing');
    return;
  }
  console.log('✅ SVG element found');

  // ★修正1: 正しい追加先を特定する
  // Mermaidの構造: <svg> → <g class="output"> → <g transform="..."> → ノード群
  // 手動エッジはノード群と同じ階層に追加する必要がある
  const appendTarget = findCorrectAppendTarget(svg);
  console.log('📎 Append target:', appendTarget.tagName, 
              appendTarget.className?.baseVal || appendTarget.className,
              'transform:', appendTarget.getAttribute('transform'));

  // ★修正2: 矢印マーカーが存在するか確認
  ensureArrowMarkerExists(svg);

  // エッジグループを作成
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  g.setAttribute('class', 'manual-edge');
  g.setAttribute('data-id', edge.id);
  g.setAttribute('data-et', 'manual-edge');

  // 線を作成
  const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  line.setAttribute('x1', String(edge.startX));
  line.setAttribute('y1', String(edge.startY));
  line.setAttribute('x2', String(edge.endX));
  line.setAttribute('y2', String(edge.endY));
  line.setAttribute('stroke', edge.style.color);
  line.setAttribute('stroke-width', String(edge.style.width));
  line.setAttribute('stroke-dasharray', edge.style.dashArray);
  line.setAttribute('marker-end', 'url(#manual-edge-arrowhead)');
  // ★修正: pointer-events を stroke にして線上のクリックは検出可能にする
  // （hit-rectがあるので none のままでもOKだが、念のため）
  line.setAttribute('pointer-events', 'none');
  
  g.appendChild(line);

  // クリック範囲の矩形を追加
  createEdgeHitRect(g, line, edge.startX, edge.startY, edge.endX, edge.endY);

  // ラベルを作成
  if (edge.label) {
    const midX = (edge.startX + edge.endX) / 2;
    const midY = (edge.startY + edge.endY) / 2;

    const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
    text.setAttribute('class', 'manual-edge-label');
    text.setAttribute('x', String(midX));
    text.setAttribute('y', String(midY));
    text.setAttribute('text-anchor', 'middle');
    text.setAttribute('fill', edge.style.color);      // ★修正: fill を明示的に設定
    text.setAttribute('font-size', '14px');             // ★修正: font-size を明示的に設定
    text.setAttribute('font-weight', 'bold');           // ★修正: 視認性向上
    text.setAttribute('dy', '-8');                      // ★修正: 線の少し上に表示
    text.textContent = edge.label;

    g.appendChild(text);
  }

  // クリックで削除
  g.addEventListener('click', (e) => {
    if (!edgeAddMode && confirm('このエッジを削除しますか？')) {
      deleteManualEdge(edge.id);
    }
    e.stopPropagation();
  });

  // ★修正: 正しい親要素に追加
  appendTarget.appendChild(g);
  
  // ★修正: 追加後の検証
  console.log(`✅ Edge ${edge.id} appended to DOM`);
  try {
    const addedBBox = g.getBBox();
    console.log(`  BBox: (${addedBBox.x.toFixed(1)}, ${addedBBox.y.toFixed(1)}) ${addedBBox.width.toFixed(1)}x${addedBBox.height.toFixed(1)}`);
    console.log(`  Parent chain: ${getParentChainStr(g)}`);
    
    // viewBox内に入っているか確認
    const viewBox = svg.viewBox.baseVal;
    const inViewBox = (
      addedBBox.x + addedBBox.width >= viewBox.x &&
      addedBBox.x <= viewBox.x + viewBox.width &&
      addedBBox.y + addedBBox.height >= viewBox.y &&
      addedBBox.y <= viewBox.y + viewBox.height
    );
    console.log(`  In viewBox (${viewBox.x},${viewBox.y},${viewBox.width},${viewBox.height}): ${inViewBox ? '✅' : '❌ OUT OF VIEW'}`);
  } catch(e) {
    console.warn('  BBox check failed:', e.message);
  }
}

/**
 * フローチャートコンテンツの正しい親要素を特定する
 * 
 * Mermaidが生成するSVGの一般的な構造に対応:
 *   パターン1: <svg> → <g class="output"> → <g transform="..."> → ノード群
 *   パターン2: <svg> → <g class="output"> → ノード群
 *   パターン3: <svg> → <g> → ノード群
 *   パターン4: <svg> → ノード群（直下）
 */
function findCorrectAppendTarget(svg) {
  // 戦略: 既存のノード要素の親を見つけて、その親を返す
  const existingNode = svg.querySelector('[data-et="node"]') || 
                       svg.querySelector('[data-et="cluster"]') ||
                       svg.querySelector('[data-et="edge"]');
  
  if (existingNode) {
    const parent = existingNode.parentElement;
    // 親がSVGルートでなければ、その親を使う
    if (parent && parent !== svg) {
      console.log(`📍 Using existing node's parent as append target: <${parent.tagName} class="${parent.className?.baseVal || ''}">`);
      return parent;
    }
  }

  // フォールバック: Mermaidの標準的な構造を探す
  const outputG = svg.querySelector('g.output > g') || 
                  svg.querySelector('g.output') || 
                  svg.querySelector('g');
  
  if (outputG) {
    console.log(`📍 Using Mermaid output group as append target: <${outputG.tagName} class="${outputG.className?.baseVal || ''}">`);
    return outputG;
  }

  // 最終フォールバック: SVGルート
  console.warn('⚠️ Could not find proper append target, using SVG root');
  return svg;
}

/**
 * 矢印マーカーが存在することを保証する
 */
function ensureArrowMarkerExists(svg) {
  let defs = svg.querySelector('defs');
  if (!defs) {
    defs = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
    svg.insertBefore(defs, svg.firstChild);
  }

  if (!defs.querySelector('#manual-edge-arrowhead')) {
    const marker = document.createElementNS('http://www.w3.org/2000/svg', 'marker');
    marker.setAttribute('id', 'manual-edge-arrowhead');
    marker.setAttribute('markerWidth', '10');
    marker.setAttribute('markerHeight', '10');
    marker.setAttribute('refX', '9');
    marker.setAttribute('refY', '3');
    marker.setAttribute('orient', 'auto');
    marker.setAttribute('markerUnits', 'strokeWidth'); // ★追加: 明示的に単位を指定

    const polygon = document.createElementNS('http://www.w3.org/2000/svg', 'polygon');
    polygon.setAttribute('points', '0,0 0,6 9,3');
    polygon.setAttribute('fill', '#007bff'); // ★修正: エッジの色と合わせる

    marker.appendChild(polygon);
    defs.appendChild(marker);
    console.log('✅ Arrow marker #manual-edge-arrowhead created');
  }
}

/**
 * デバッグ用: 要素の親チェーンを文字列で返す
 */
function getParentChainStr(element) {
  let chain = [];
  let el = element;
  while (el && el.tagName !== 'HTML') {
    const cls = el.className?.baseVal || el.className || '';
    const transform = el.getAttribute?.('transform') || '';
    chain.push(`<${el.tagName}${cls ? '.' + cls.split(' ').join('.') : ''}${transform ? ' T:' + transform.substring(0, 30) : ''}>`);
    el = el.parentElement;
  }
  return chain.join(' → ');
}
```

### 補足: その他の原因と対策

#### CSS による非表示の可能性

Mermaid生成のCSSやアプリケーションのCSSが、手動追加要素に意図しない影響を与えている可能性があります。

```javascript
// 追加後にcomputed styleを確認
const edge = document.querySelector('[data-et="manual-edge"]');
if (edge) {
  const style = window.getComputedStyle(edge);
  console.log('Computed style:', {
    display: style.display,
    visibility: style.visibility,
    opacity: style.opacity,
    overflow: style.overflow,
    clipPath: style.clipPath
  });
  
  const line = edge.querySelector('line');
  if (line) {
    const lineStyle = window.getComputedStyle(line);
    console.log('Line computed style:', {
      stroke: lineStyle.stroke,
      strokeWidth: lineStyle.strokeWidth,
      display: lineStyle.display,
      visibility: lineStyle.visibility,
      opacity: lineStyle.opacity
    });
  }
}
```

#### `createEdgeHitRect` がエラーで処理を中断している可能性

`drawEdgeToSvg()` 内で呼ばれている `createEdgeHitRect()` が例外を投げた場合、以後の処理（ラベル追加、appendChild）がスキップされます。ただしログでは `appendChild` 成功が出ているため、可能性は低いです。

念のため、try-catch で囲むことを推奨します：

```javascript
// createEdgeHitRect の呼び出しを安全にする
try {
  createEdgeHitRect(g, line, edge.startX, edge.startY, edge.endX, edge.endY);
} catch (err) {
  console.warn('createEdgeHitRect failed (non-critical):', err);
}
```

---

## まとめ

### 問題1: サブグラフ優先吸着

| 項目 | 内容 |
|------|------|
| **根本原因** | 中心距離のみで判定しているため、大きなサブグラフの中心が常に近くなる |
| **推奨解決策** | 二段階フォールバック方式（個別ノード優先 + 境界距離） |
| **修正関数** | `findNearestNode()`, 新規: `findNearestInSet()`, `calcBoundaryDistance()` |
| **修正規模** | 既存関数の書き換え + ヘルパー関数2つ追加 |

### 問題2: エッジ非表示

| 項目 | 内容 |
|------|------|
| **最有力原因** | `svg.appendChild()` で SVGルート直下に追加しているが、既存ノードは `<g class="output">` 配下の transform 付きグループ内にあり、座標系がずれている |
| **推奨解決策** | 既存ノードの親要素を検出し、同じ親に `appendChild()` する |
| **修正関数** | `drawEdgeToSvg()`, 新規: `findCorrectAppendTarget()`, `ensureArrowMarkerExists()` |
| **デバッグ** | 上記デバッグ手順のステップ1〜4を実行して原因を確定 |

### 実装優先順位

1. **まずデバッグ手順を実行**して問題2の原因を確定する（5分程度）
2. **問題2を修正**する（`drawEdgeToSvg()` の追加先変更）
3. **問題1を修正**する（`findNearestNode()` の二段階フォールバック化）
4. 動作確認・テスト

---

## 参考: `calcBoundaryDistance` の数学的説明

```
与えられた点 P(px, py) と矩形 R(left, top, right, bottom) に対して：

dx = max(left - px, 0, px - right)
dy = max(top - py, 0, py - bottom)
distance = sqrt(dx² + dy²)

解説:
- P が矩形の左側にある場合:   dx = left - px > 0
- P が矩形のX範囲内にある場合: dx = 0
- P が矩形の右側にある場合:   dx = px - right > 0
- Y方向も同様

これにより:
- P が矩形内部: dx = 0, dy = 0 → distance = 0
- P が矩形の辺の延長上: dx or dy のいずれかが0
- P が矩形の角の外側: dx > 0, dy > 0 → 角までのユークリッド距離
```

この計算は O(1) であり、パフォーマンスへの影響はありません。

---

*以上、ご質問いただいた2つの問題に対する分析と解決策です。追加の質問やデバッグ結果の共有がありましたら、お気軽にお知らせください。*
