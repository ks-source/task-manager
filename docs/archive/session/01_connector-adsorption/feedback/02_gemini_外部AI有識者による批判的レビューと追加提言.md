
# 外部AI有識者 批判的レビューおよび追加提言書

---
**プロジェクト**: タスク管理ツール - フローチャート手動編集機能  
**対象**: Phase 2 - ノード自動吸着（前回の回答に対するレビューと補完）  
**レビュー日**: 2026-03-21  

---

前回の提案（二段階フォールバック方式、親要素への追加）は論理的な解決策ですが、より批判的な視点・実用的な観点からコードと要件を再評価した結果、**実際のブラウザ環境で実装する際に致命的となる3つの課題（盲点）**が浮き彫りになりました。

本ドキュメントでは、それらの課題に対する新しい視点と、より堅牢な実装のための補完コードを提供します。

## 視点1: 座標系変換の罠（CTMの考慮漏れ）- 【最重要】

### 批判的評価
前回の「問題2（エッジ非表示）」の解決策において、「エッジを `<g class="output">` などの transform が適用された親要素に追加する」と提案しました。しかし、これは**エッジの開始点・終了点（`startX`, `startY`）がどの座標系で計算されているか**によって、致命的なバグを生みます。

Mermaidは多くの場合、ノードを以下のように配置します：
```xml
<g class="output" transform="translate(10, 10) scale(1.5)">
  <g class="node" transform="translate(100, 200)">
    <rect x="-50" y="-30" width="100" height="60" />
  </g>
</g>
```

もし `startX` が「SVGルートに対する絶対座標」である場合、それを `transform` がかかった親要素の中に入れると、**座標が二重に変換（ダブルトランスフォーム）され、画面の遥か彼方に描画されてしまいます**。

### 追加提言: DOMMatrix（CTM）を利用した正確な座標マッピング
SVGでマウスクリック位置や異なるグループ間の座標を扱う場合、`getBBox()` や単なる `x, y` の計算ではなく、ブラウザ標準APIの `getScreenCTM()` を使用して**スクリーン座標からSVGローカル座標への変換**を厳密に行うべきです。

**補完コード: マウス座標から任意のSVG要素内のローカル座標を取得する関数**

```javascript
/**
 * クライアント座標（マウス位置など）を、特定のSVG要素内のローカル座標に変換する
 * @param {SVGSVGElement} svg - ルートのSVG要素
 * @param {SVGGraphicsElement} targetElement - 追加先の要素（<g class="output"> など）
 * @param {number} clientX - マウスのX座標
 * @param {number} clientY - マウスのY座標
 * @returns {DOMPoint} ローカル座標系の点
 */
function getLocalCoordinates(svg, targetElement, clientX, clientY) {
  // SVGの変換マトリックスを取得
  const ctm = targetElement.getScreenCTM();
  if (!ctm) return { x: 0, y: 0 };

  // マトリックスの逆行列を計算
  const inverseCTM = ctm.inverse();
  
  // マウスの座標をDOMPointに設定
  const pt = svg.createSVGPoint();
  pt.x = clientX;
  pt.y = clientY;

  // 逆行列を適用してローカル座標を取得
  return pt.matrixTransform(inverseCTM);
}
```
この視点を取り入れることで、エッジをどこに `appendChild` しても、ズレることなく意図した位置に描画できるようになります。

---

## 視点2: 「吸着判定」と「描画座標」の分離

### 批判的評価
前回の「境界距離」による判定は「どれを選ぶか（Selection）」としては優れていますが、「どこから線を引くか（Connection）」としては不十分です。
境界距離が `0`（ノードの内部をクリックした）場合、そのままクリックした座標から線を引くと、**線がノードの内部から始まってしまい、見た目が美しくありません**。

Mermaidの本来のエッジは、ノードの「境界線上」から描画されます。

### 追加提言: 最寄りの交点（Anchor Point）の計算
ユーザーがノードの内部や近辺をクリックして吸着した場合、線の始点/終点は**「クリックした位置とノードの中心を結んだ線分が、境界ボックス（BBox）と交差する点」**に補正するべきです。

**補完コード: 吸着時の美しい接続点の計算**

```javascript
/**
 * ノード中心と対象点（クリック位置など）を結ぶ線分と、
 * ノードの境界ボックスとの交点を計算する
 */
function getPerimeterPoint(centerX, centerY, bbox, targetX, targetY) {
  // BBoxの4辺の座標
  const left = bbox.x;
  const right = bbox.x + bbox.width;
  const top = bbox.y;
  const bottom = bbox.y + bbox.height;

  // 中心からターゲットへの角度
  const dx = targetX - centerX;
  const dy = targetY - centerY;
  
  if (dx === 0 && dy === 0) return { x: centerX, y: centerY };

  const slope = dy / dx;
  let px, py;

  // ターゲットが中心の右か左か
  if (dx > 0) {
    px = right;
    py = centerY + slope * (right - centerX);
  } else {
    px = left;
    py = centerY + slope * (left - centerX);
  }

  // 上下境界のチェック
  if (py > bottom) {
    py = bottom;
    px = centerX + (bottom - centerY) / slope;
  } else if (py < top) {
    py = top;
    px = centerX + (top - centerY) / slope;
  }

  return { x: px, y: py };
}
```
この処理を挟むことで、「アバウトにノード付近をクリックしても、線は必ずノードの縁（ふち）から綺麗に繋がる」という、プロフェッショナルな描画ツールのUXを実現できます。

---

## 視点3: MermaidのライフサイクルとDOM破壊（Mutation）

### 批判的評価
要件と設計を見る限り、「DOMに手動でエッジを追加する」アプローチを取っています。しかし、Mermaidはウィンドウのリサイズや内容の再レンダリング時（ツール側で表示更新があった場合など）に、**SVG内部を完全に破棄して再描画する**ことがあります。

この場合、`appendChild` で手動追加した `<g class="manual-edge">` はすべて消え去ります。

### 追加提言: 状態管理と再適用（Re-hydration）メカニズム
「画面上のDOM」と「手動エッジのデータ状態」を完全に切り離し、**SVGが書き換わったことを検知して、自動的にエッジを再描画する仕組み**が必要です。
提供された `05_data-structure-spec.md` にある JSON データ構造を「正」とし、MutationObserverなどでDOMの監視を行うべきです。

**補完コード: SVGの再構築検知と復元**

```javascript
// エッジの描画状態を管理するObserver
function observeMermaidChanges(svgElement, manualEdgesData) {
  const observer = new MutationObserver((mutations) => {
    // manual-edge 以外の大きな変更があったかチェック
    const isMermaidRedraw = mutations.some(m => {
      // 追加されたノードに output グループが含まれるか等で判定
      return Array.from(m.addedNodes).some(node => 
        node.tagName === 'g' && node.classList?.contains('output')
      );
    });

    if (isMermaidRedraw) {
      console.log('🔄 Mermaid redraw detected. Re-applying manual edges...');
      // 一旦既存の manual-edge をクリア（残骸があれば）
      document.querySelectorAll('[data-et="manual-edge"]').forEach(el => el.remove());
      
      // データに基づいてすべて再描画
      manualEdgesData.forEach(edgeData => {
        drawEdgeToSvg(edgeData);
      });
    }
  });

  observer.observe(svgElement, {
    childList: true,
    subtree: true
  });
  
  return observer;
}
```

---

## まとめと推奨アクションプラン

前回の提案（二段階吸着・親要素の変更）を実装する前に、以下の順序でアプローチすることを強く推奨します。

1. **（座標の統一）** まず `getScreenCTM()` を利用して、ユーザーのクリック座標が「SVG内のどのグループの座標系」として取得されているかを確認・統一する。
2. **（表示の修正）** 視点1がクリアできれば、前回提案した `findCorrectAppendTarget()` で適切な親要素に追加するだけで、確実にエッジが表示される（問題2の解決）。
3. **（吸着UXの向上）** 前回提案の「二段階フォールバック」に加え、本稿「視点2」の「交点計算」を導入し、エッジがノードの境界から綺麗に生えるようにする（問題1の根本解決）。
4. **（堅牢性の担保）** 最後に、リサイズ時などにエッジが消えないかテストし、消える場合は「視点3」のObserverを導入する。
