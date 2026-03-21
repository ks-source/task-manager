# コンソールログ

---
**取得日**: 2026-03-21
**ブラウザ**: Microsoft Edge / Chrome
**SVGズーム**: 150%

---

## 吸着検出ログ（マウス移動中）

ユーザーが SCOPE サブグラフ付近でマウスを動かしていた時のログ（抜粋）：

```
📏 Scale: 0.67x, Screen radius: 1000px, SVG radius: 666.7px
🔍 findNearestNode: Found 48 nodes with [data-et="node|cluster"]
Sample node: Object
📍 Click at (1257.3, 1898.1), screen radius: 1000px, adjusted: 666.7px
🎯 Top 5 closest nodes:
  1. PHASE3: 2547.9px - center:(1603.7, 4422.4) bbox:(1306.9, 3645.0) 593.7x1554.7
  2. PHASE2: 628.6px - center:(977.6, 2461.0) bbox:(8.0, 1348.0) 1939.1x2226.0
  3. PHASE1: 1104.5px - center:(1599.7, 848.0) bbox:(1300.7, 419.0) 598.0x858.0
  4. SCOPE: 299.5px - center:(1371.7, 1911.5) bbox:(937.2, 1510.0) 869.0x803.0
🎯 ノード吸着: SCOPE (距離: 299.5px)
```

### ログ分析

1. **スケール情報**:
   - Scale: 0.67x（SVGが150%にズームされている）
   - Screen radius: 1000px → SVG radius: 666.7px

2. **検出ノード数**:
   - 48 nodes（`[data-et="node"]` + `[data-et="cluster"]`）

3. **距離順位**:
   | 順位 | ノードID | 距離 | 中心座標 | bbox | サイズ |
   |------|---------|------|----------|------|--------|
   | 1 | PHASE3 | 2547.9px | (1603.7, 4422.4) | (1306.9, 3645.0) | 593.7x1554.7 |
   | 2 | PHASE2 | 628.6px | (977.6, 2461.0) | (8.0, 1348.0) | 1939.1x2226.0 |
   | 3 | PHASE1 | 1104.5px | (1599.7, 848.0) | (1300.7, 419.0) | 598.0x858.0 |
   | 4 | **SCOPE** | **299.5px** | **(1371.7, 1911.5)** | **(937.2, 1510.0)** | **869.0x803.0** |

4. **選択されたノード**:
   - SCOPE (距離: 299.5px) ← サブグラフが選ばれている

5. **クリック位置**:
   - (1257.3, 1898.1) ← SCOPE の中心 (1371.7, 1911.5) に近い

---

## 継続的なマウス移動ログ

ユーザーがマウスを少しずつ動かした際のログ（SCOPE に吸着し続けている）：

```
📍 Click at (1468.7, 1627.5), screen radius: 1000px, adjusted: 666.7px
🎯 Top 5 closest nodes:
  1. PHASE3: 2798.2px
  2. PHASE2: 967.5px
  3. PHASE1: 790.4px
  4. SCOPE: 300.1px
🎯 ノード吸着: SCOPE (距離: 300.1px)

📍 Click at (1468.7, 1626.8), screen radius: 1000px, adjusted: 666.7px
🎯 ノード吸着: SCOPE (距離: 300.8px)

📍 Click at (1469.3, 1625.5), screen radius: 1000px, adjusted: 666.7px
🎯 ノード吸着: SCOPE (距離: 302.2px)

... (多数の類似ログ)

📍 Click at (1551.3, 1504.8), screen radius: 1000px, adjusted: 666.7px
🎯 Top 5 closest nodes:
  1. PHASE3: 2918.0px
  2. PHASE2: 1115.1px
  3. PHASE1: 658.6px
  4. SCOPE: 444.6px
🎯 ノード吸着: SCOPE (距離: 444.6px)
```

### 観察

- マウス位置が変わっても**常にSCOPEに吸着**している
- PHASE1, PHASE2, PHASE3 は距離が遠い（600px以上）
- SCOPEは 299.5px ~ 444.6px の範囲で常に最寄り

---

## エッジ作成時のログ（予想）

ユーザーがクリックしてエッジを作成した際のログは提供されていませんが、以下が期待されます：

```
🖱️ onSvgClickForEdgeAdd called, edgeAddMode: true
Click: { x: 1551.3, y: 1504.8, edgeStartPoint: null }
🎯 ノード吸着: SCOPE (距離: 444.6px)

[ユーザーがラベル入力: "test"]

🎨 drawEdgeToSvg called: {
  id: "manual-edge-1234567890",
  startX: 1371.7,
  startY: 1911.5,
  startNodeId: "SCOPE",
  endX: 1468.7,
  endY: 1626.8,
  endNodeId: "SCOPE",
  label: "test",
  style: { color: "#007bff", width: 5, dashArray: "5,5" },
  createdAt: "2026-03-21T..."
}
✅ SVG element found: <svg ...>
✅ Edge manual-edge-1234567890 added to SVG: {
  start: "(1371.7, 1911.5)",
  end: "(1468.7, 1626.8)",
  label: "test"
}

🔄 toggleEdgeAddMode: OFF
✅ Edge add mode disabled, classes removed
```

**問題**: 上記ログが出力されているにも関わらず、画面上にエッジが見えない

---

## モード切り替えログ

```
🔄 toggleEdgeAddMode: OFF
✅ Edge add mode disabled, classes removed
```

- エッジ作成後、自動的にモードがOFFになる（正常動作）

---

## ノード検索の詳細ログ（初回クリック時）

```
🔍 findNearestNode: Found 48 nodes with [data-et="node|cluster"]
Sample node: Object
```

- 48個のノード（個別ノード + サブグラフ）を検出
- サンプルノードの詳細は不明（オブジェクト表示のみ）

---

## bbox情報の詳細

各ノードの bbox（境界ボックス）情報：

### PHASE1（個別ノード）
- Center: (1599.7, 848.0)
- BBox origin: (1300.7, 419.0)
- Size: 598.0 x 858.0
- Area: 513,084 px²

### PHASE2（個別ノード）
- Center: (977.6, 2461.0)
- BBox origin: (8.0, 1348.0)
- Size: 1939.1 x 2226.0
- Area: 4,316,226 px²（非常に大きい）

### PHASE3（個別ノード）
- Center: (1603.7, 4422.4)
- BBox origin: (1306.9, 3645.0)
- Size: 593.7 x 1554.7
- Area: 923,095 px²

### SCOPE（サブグラフ）
- Center: (1371.7, 1911.5)
- BBox origin: (937.2, 1510.0)
- Size: 869.0 x 803.0
- Area: 697,807 px²

### 観察

- PHASE2 が非常に大きい（4,316,226 px²）← サブグラフかもしれない
- SCOPE のサイズは中程度（697,807 px²）
- サイズだけでは個別ノードとサブグラフを区別できない

---

## エラーログ

特にエラーは出力されていない：

- `getBBox()` の失敗なし
- SVG要素の取得失敗なし
- 座標変換エラーなし

---

## まとめ

1. **吸着機能は動作している**: SCOPEに正しく吸着
2. **優先度が問題**: 個別ノードではなくサブグラフが選ばれる
3. **エッジは作成されている**: ログで確認可能
4. **エッジが見えない**: 原因不明（ログにエラーなし）
