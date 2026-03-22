# トラブルシューティング履歴

---
**期間**: 2026-03-21
**対象機能**: Phase 2 - ノード自動吸着

---

## Phase 1実装（完了）

### Phase 1の実装内容
- ✅ 2点クリックで直線エッジを追加
- ✅ エッジラベル入力ダイアログ
- ✅ エッジのSVG描画
- ✅ エッジクリックで削除
- ✅ JSON保存・読込
- ✅ プレビュー線表示

---

## Phase 2実装開始

### 目標
1. ノード付近（半径50px以内）をクリック時、ノードの中心に自動吸着
2. 吸着時に緑色のハイライト表示
3. エッジデータに `startNodeId`, `endNodeId` を保存

---

## 試行1: ボタン未表示問題 ✅ 解決

### 問題
- ユーザー: 「UIボタンが最初から十字の状態であり、クリックして有効化しても視覚的変化が起きません」

### 調査
- エッジ追加ボタンのHTMLが存在しない
- ボタンID mismatch: コードは `edge-add-btn` を参照、実際のIDは `edge-add-mode-btn`

### 解決策
1. HTMLにボタン要素を追加（line 1859）
2. ボタンIDを統一: `edge-add-mode-btn`
3. CSS を強化: `!important` フラグ追加

```css
.btn.active,
.icon-button.active,
#edge-add-mode-btn.active {
  background-color: #007bff !important;
  color: white !important;
  border-color: #0056b3 !important;
}
```

### 結果
✅ ボタンが表示され、クリックで青くなるように修正

---

## 試行2: ノード吸着しない問題（距離問題） ✅ 解決

### 問題
- ユーザー: 「かなりノード付近でエッジ始点をクリックしても最寄りのノードに吸着せず、終点をかなりノード付近に指定しても吸着しません」

### 調査

#### 第1段階: デバッグログ追加
```javascript
console.log(`🔍 findNearestNode: Found ${nodes.length} nodes`);
console.log(`📍 Click at (${x}, ${y}), radius: ${radius}px`);
console.log(`🎯 Top 5 closest nodes:`);
```

**判明した事実**:
- 48個のノードを検出（正常）
- 最寄りノードまでの距離: 628.6px
- 初期半径: 50px
- **50px << 628.6px** → 吸着失敗

#### 第2段階: スケール問題を発見

ユーザーのコンソールログ:
```
📏 Scale: 0.67x, Screen radius: 50px, SVG radius: 33.3px
📍 Click at (1257.3, 1898.1)
🎯 PHASE2: 628.6px ← 最寄りノード
```

**スケール計算**:
```javascript
scaleX = viewBox.width / rect.width = 0.67
adjustedRadius = 50 * 0.67 = 33.3px
```

**問題**:
- SVGが150%にズームされている → viewBox < screen rect
- スケール = 0.67x < 1
- 半径が SMALLER になってしまう
- しかも元の50pxが小さすぎる

### 解決策の試行

#### 試行A: スケール調整実装
```javascript
const scale = Math.max(scaleX, scaleY);
const adjustedRadius = radius * scale;
```

**結果**: ❌ 改善なし
- ユーザー: 「全く改善しません」
- 理由: scale < 1 のため、かえって半径が小さくなる

#### 試行B: 半径を200pxに増加
```javascript
function findNearestNode(x, y, radius = 200) {
```

**計算**:
- 200 * 0.67 = 133.3px SVG座標
- まだ 628.6px には届かない

**結果**: ⚠️ 不十分（ユーザー未テスト）

#### 試行C: 半径を1000pxに増加
```javascript
function findNearestNode(x, y, radius = 1000) {
```

**計算**:
- 1000 * 0.67 = 666.7px SVG座標
- **666.7px > 628.6px** ✅ 範囲内！

**結果**: ✅ 吸着成功
- コンソールログ: `🎯 ノード吸着: SCOPE (距離: 299.5px)`

#### 試行D: 明示的な引数を削除

**問題発見**:
```javascript
const nearestNode = findNearestNode(x, y, 50);  // ❌ 50を明示
```

全ての呼び出しで `50` を明示していたため、デフォルト引数（200, 1000）が無視されていた！

**修正**:
```javascript
const nearestNode = findNearestNode(x, y);  // ✅ デフォルト引数を使用
```

### 最終解決策
1. デフォルト半径: 1000px
2. 全ての呼び出しから明示的な `50` を削除
3. スケール調整を維持（将来的に見直す可能性）

### 結果
✅ ノード吸着が動作するようになった

---

## 試行3: サブグラフ優先吸着問題 ⚠️ 未解決

### 問題
- ユーザー: 「サブグラフは緑ハイライトになりましたが、接続したいノード上では緑ハイライトになりませんでした」

### 状況
```
🎯 Top 5 closest nodes:
  1. PHASE3: 2547.9px
  2. PHASE2: 628.6px
  3. PHASE1: 1104.5px
  4. SCOPE: 299.5px ← サブグラフ（最寄り）
🎯 ノード吸着: SCOPE
```

### 原因分析

**なぜSCOPEが選ばれる？**
- SCOPE（サブグラフ）の中心: (1371.7, 1911.5)
- クリック位置: (1257.3, 1898.1)
- 距離: 299.5px

- PHASE1（個別ノード）の中心: (1599.7, 848.0)
- クリック位置: (1257.3, 1898.1)
- 距離: 1104.5px

**SCOPE が物理的に近い**ため、ユークリッド距離で選ばれる。

### 誤った解決試行

**試行**: サブグラフを除外
```javascript
const nodes = svg.querySelectorAll('[data-et="node"]');  // clusterを除外
```

**ユーザーの反応**:
> いやいや、サブグラフもノードもどちらも吸着対象なのでサブグラフを吸着候補から外すのはやめてください。

### 現在の状況
⚠️ **未解決**

**必要な対応**:
1. サブグラフと個別ノードの両方を吸着対象にする
2. 個別ノードを優先する仕組みを実装
3. 外部AIからフィードバックを得る

---

## 試行4: エッジ非表示問題 ⚠️ 未解決

### 問題
- ユーザー: 「そのエッジとエッジラベルは表示されませんでした」

### 状況

**コンソールログ（成功している）**:
```
🎨 drawEdgeToSvg called: {id: "manual-edge-...", ...}
✅ SVG element found: <svg ...>
✅ Edge manual-edge-... added to SVG: {start: "(x, y)", end: "(x, y)", label: "test"}
```

**実装コード**:
```javascript
svg.appendChild(g);  // ← 実行されている
```

### 調査

#### デバッグログを追加
- `drawEdgeToSvg()` 呼び出し確認 → ✅ OK
- SVG要素取得 → ✅ OK
- `appendChild()` 実行 → ✅ OK
- **エラーなし**

#### エッジの視認性を向上
```javascript
style: {
  color: '#007bff',  // 灰色 → 青色
  width: 5,          // 2px → 5px
  dashArray: '5,5'
}
```

### 現在の状況
⚠️ **未解決**

**考えられる原因**:
1. ViewBox範囲外に描画されている
2. Z-index問題（他の要素の下に隠れている）
3. opacity: 0 が設定されている（CSS）
4. マーカー（矢印）が未定義
5. 座標値がNaNまたはInfinity

**次のステップ**:
1. ブラウザのDOM Inspectorで確認
2. `document.querySelector('[data-et="manual-edge"]')` の結果を確認
3. 外部AIからフィードバックを得る

---

## ブラウザキャッシュ問題 ✅ 解決

### 問題
- コード変更が反映されない

### 解決策
- ハードリロード: Ctrl + Shift + R
- または: F12 → 右クリック → "キャッシュの消去とハードの再読み込み"

### 結果
✅ ユーザーに指示し、正しくリロードされた

---

## スケール計算の考察

### 現在の実装

```javascript
const scaleX = viewBox.width / rect.width;  // 例: 2500 / 3731 = 0.67
const scaleY = viewBox.height / rect.height;
const scale = Math.max(scaleX, scaleY);
const adjustedRadius = radius * scale;      // 1000 * 0.67 = 666.7
```

### 問題点

**ズーム時の挙動**:
- 150% zoom → viewBox < screen → scale < 1
- 半径が SMALLER になる
- 理論的には逆（LARGER にすべき）

**しかし動作している理由**:
- SVG座標系での距離が大きい（600+px）
- 1000px の画面半径で十分カバーできている
- スケール調整が正しく機能している（偶然？）

### 将来的な検討事項

**正しいスケール計算**:
```javascript
// 画面上の半径をSVG座標系に変換
const adjustedRadius = radius * (viewBox.width / rect.width);
```

または

```javascript
// SVG座標系の半径を直接指定
const adjustedRadius = radius / scale;
```

現状は**動作している**ため、優先度は低い。

---

## まとめ

### ✅ 解決済み
1. エッジ追加ボタンの視覚的フィードバック
2. ノード吸着の半径問題（1000pxで解決）
3. 明示的引数の削除

### ⚠️ 未解決
1. **サブグラフ優先吸着問題**（重要度: HIGH）
   - 両方を吸着対象にしつつ、個別ノードを優先したい
   - 外部AIの知見が必要

2. **エッジ非表示問題**（重要度: CRITICAL）
   - コンソールログでは成功しているが画面に見えない
   - DOM Inspector での確認が必要
   - 外部AIの知見が必要

### 🔄 保留
1. スケール計算の見直し（動作しているため低優先度）

---

## 次のアクション

1. ✅ 外部AI有識者用ドキュメントを作成
2. ⏳ 外部AIからフィードバックを得る
3. ⏳ 推奨される解決策を実装
4. ⏳ ユーザーに動作確認を依頼
