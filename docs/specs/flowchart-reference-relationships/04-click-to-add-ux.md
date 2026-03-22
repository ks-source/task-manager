# クリック to Add 参照 UX 仕様

---
**feature**: フローチャート参照関係管理
**document**: クリック to Add UX 仕様
**version**: 1.0
**status**: draft
**created**: 2026-03-22
**updated**: 2026-03-22
---

## 概要

メモノードや正規ノードに参照関係を追加する際、**dialog/select ベースではなく、フローチャート上でノードをクリックして直接追加する UX パターン**。

本仕様は、ユーザーからの UX 改善提案を反映し、Phase 2-2 として実装される。

---

## 設計の背景

### ❌ 廃止された実装（dialog 方式）

初期実装では、以下のような dialog/select ベースの UI が試行されていた：

```html
<!-- ❌ 廃止された実装 -->
<div class="info-section" id="memo-node-connections-section">
  <select id="connection-edge-select">
    <option value="">-- エッジを選択 --</option>
    <!-- 動的生成 -->
  </select>
  <button onclick="addConnectionEdge()">+</button>
</div>
```

**問題点**:
1. **意味的に不明確**: 「接続エッジ」ではなく「関連ノード」を追加すべき
2. **UX の煩雑さ**: ドロップダウンから選択するのは操作が多い
3. **Phase 2 との重複**: `elementReferences` との機能重複

### ✅ 採用された設計（クリック to Add 方式）

フローチャート上でノードを直接クリックして参照を追加する、直感的な UX。

**利点**:
1. **操作が直感的**: ビジュアルエディタとしての使いやすさ
2. **ステップ数削減**: ドロップダウン選択が不要
3. **視覚的フィードバック**: クリック時にプレビュー表示

---

## UI フロー

### 1. 参照追加モードの開始

```
[要素情報パネル]
┌──────────────────────────────┐
│ 要素情報                      │
├──────────────────────────────┤
│ タイプ: メモノード            │
│ ラベル: 重要な注意事項        │
│                              │
│ ┌──────────────────────────┐ │
│ │ 📌 参照関係              │ │
│ ├──────────────────────────┤ │
│ │ → flowchart-A (開始)     │ │
│ │ → memo_node_2 (補足)     │ │
│ │                          │ │
│ │ [➕ 参照を追加]          │ │ ← クリック
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

ユーザーが「参照を追加」ボタンをクリックすると、参照追加モードに移行。

### 2. 参照追加モード中の表示

```
[フローチャート画面]
┌──────────────────────────────────────┐
│ 🖱 参照追加モード                    │  ← ヘッダー通知
│ クリックして関連ノードを選択...      │
├──────────────────────────────────────┤
│                                      │
│   ┌────────┐                         │
│   │ 開始   │ ← カーソルがポインター   │
│   └────────┘                         │
│       │                              │
│   ┌────────┐                         │
│   │ 処理A  │ ← ホバー時にハイライト   │
│   └────────┘                         │
│                                      │
│   📝 [重要な注意事項]  ← 選択中の要素 │
│                                      │
│  [ESC: キャンセル]                   │  ← フッター通知
└──────────────────────────────────────┘
```

**視覚的フィードバック**:
- カーソル: `cursor: crosshair` または専用アイコン
- ホバー時: 候補ノードを半透明ハイライト
- ヘッダー: 現在のモードを通知
- フッター: キャンセル方法を表示

### 3. ノードクリック時の動作

```
ユーザーがフローチャート上の "処理A" をクリック
↓
[要素情報パネルが即座に更新]
┌──────────────────────────────┐
│ 要素情報                      │
├──────────────────────────────┤
│ ┌──────────────────────────┐ │
│ │ 📌 参照関係              │ │
│ ├──────────────────────────┤ │
│ │ → flowchart-A (開始)     │ │
│ │ → memo_node_2 (補足)     │ │
│ │ → flowchart-B (処理A)   ★│ │ ← 新規追加
│ │                          │ │
│ │ [➕ 参照を追加]          │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
↓
参照追加モードが自動終了
```

**処理内容**:
1. クリックされたノードのIDを取得
2. 選択中の要素の `elementReferences[elementId]` に追加
3. 要素情報パネルの参照リストを更新
4. 参照追加モードを終了（通常モードに復帰）
5. 自動保存をトリガー

---

## データ構造

### Phase 2 の `elementReferences` をそのまま使用

```javascript
// グローバル変数（既存）
let elementReferences = {};

// 参照追加処理
function addReferenceByClick(sourceElementId, targetNodeId) {
  // 初期化
  if (!elementReferences[sourceElementId]) {
    elementReferences[sourceElementId] = [];
  }

  // 重複チェック
  if (elementReferences[sourceElementId].includes(targetNodeId)) {
    console.warn(`[Reference] 既に参照済み: ${sourceElementId} → ${targetNodeId}`);
    return false;
  }

  // 自己参照チェック
  if (sourceElementId === targetNodeId) {
    console.warn(`[Reference] 自己参照は禁止: ${sourceElementId}`);
    return false;
  }

  // 参照を追加
  elementReferences[sourceElementId].push(targetNodeId);

  console.log(`[Reference] 参照追加: ${sourceElementId} → ${targetNodeId}`);
  return true;
}
```

**既存設計との完全互換**:
- `elementReferences` はPhase 2 で既に定義済み
- `buildMemoData()` で自動的に保存される
- `restoreReferencesFromMemoData()` で自動的に復元される

---

## 実装範囲

### ✅ 実装する機能

1. **参照追加モードの管理**
   - グローバルフラグ: `isAddingReference = false`
   - 選択中の要素ID: `addingReferenceSourceId = null`

2. **UI コンポーネント**
   - 要素情報パネルに「参照を追加」ボタン
   - 参照リスト表示（既存参照の一覧）
   - 参照削除ボタン（×アイコン）

3. **モード制御**
   - 参照追加モード開始: `startAddingReference(elementId)`
   - 参照追加モード終了: `cancelAddingReference()`
   - ノードクリックハンドラ: `handleNodeClickInReferenceMode(event)`

4. **視覚的フィードバック**
   - カーソル変更: `cursor: crosshair`
   - ヘッダー通知: "参照追加モード中..."
   - ホバー時ハイライト: 候補ノードを半透明化
   - ESCキーでキャンセル

5. **データ永続化**
   - `buildMemoData()` での自動保存（既存機能を活用）
   - `restoreReferencesFromMemoData()` での復元（既存機能を活用）

### ❌ 実装しない機能

1. **参照矢印の自動描画**
   - Phase 2-3 で実装予定（本フェーズではスコープ外）

2. **参照の種類（type）の指定**
   - Phase 3 で実装予定（現在は汎用的な `references` のみ）

3. **エッジへの参照**
   - MVPではノードとメモノードのみ対象（エッジは除外）

---

## 削除対象の実装

以下の実装を削除する（flowchart-editor.html）:

### 1. HTML (lines 2961-3007)

```html
<!-- ❌ 削除対象 -->
<div class="info-section" id="memo-node-connections-section" style="display: none;">
  <div class="info-section-title">
    <span>接続エッジ</span>
  </div>
  <div id="memo-node-connections-list" class="connections-list">
    <!-- 動的生成 -->
  </div>
  <div class="connection-add-row">
    <select id="connection-edge-select" class="connection-select">
      <option value="">-- エッジを選択 --</option>
    </select>
    <button class="btn-icon-small" onclick="addConnectionEdge()" title="エッジを追加">+</button>
  </div>
</div>

<div class="info-section" id="manual-edge-conceptual-section" style="display: none;">
  <!-- 手動エッジの概念的接続編集UI -->
  <!-- ... 約40行 ... -->
</div>
```

### 2. CSS (lines 1573-1666)

```css
/* ❌ 削除対象 */
.connections-list {
  margin-bottom: 8px;
  max-height: 150px;
  overflow-y: auto;
}

.connection-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 6px 8px;
  margin-bottom: 4px;
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  font-size: 12px;
  gap: 8px;
}

/* ... 約90行の関連スタイル ... */
```

### 3. JavaScript (lines 10174-10471)

```javascript
// ❌ 削除対象の関数
function updateMemoNodeConnectionsUI(elementId) { /* ... */ }
function addConnectionEdge() { /* ... */ }
function removeConnectionEdge(elementId, edgeId) { /* ... */ }
function updateManualEdgeConceptualUI(edgeId) { /* ... */ }
function getAllSelectableNodeIds() { /* ... */ }  // ※ これは再利用可能
function updateConceptualStart(edgeId) { /* ... */ }
function updateConceptualEnd(edgeId) { /* ... */ }
function updateConceptualMemo(edgeId) { /* ... */ }
```

### 4. selectElement() の統合処理 (lines 5913-5931)

```javascript
// ❌ 削除対象
const memoNodeConnectionsSection = document.getElementById('memo-node-connections-section');
if (elementType === 'manual-node' || element.classList.contains('manual-memo-node')) {
  updateMemoNodeConnectionsUI(elementId);
} else if (memoNodeConnectionsSection) {
  memoNodeConnectionsSection.style.display = 'none';
}

// ❌ 削除対象
const manualEdgeConceptualSection = document.getElementById('manual-edge-conceptual-section');
if (elementType === 'manual-edge') {
  updateManualEdgeConceptualUI(elementId);
} else if (manualEdgeConceptualSection) {
  manualEdgeConceptualSection.style.display = 'none';
}
```

---

## 実装例

### 1. HTML（要素情報パネルに追加）

```html
<!-- ✅ 新規追加 -->
<div class="info-section" id="element-references-section" style="display: none;">
  <div class="info-section-title">
    <span>📌 参照関係</span>
  </div>
  <div id="element-references-list" class="references-list">
    <!-- 動的生成:
    <div class="reference-item">
      <span class="reference-label">flowchart-A (開始)</span>
      <button class="btn-icon-small" onclick="removeReference('memo_1', 'flowchart-A')">×</button>
    </div>
    -->
  </div>
  <button class="btn-secondary" onclick="startAddingReference()" id="btn-add-reference">
    ➕ 参照を追加
  </button>
</div>
```

### 2. CSS

```css
/* ✅ 新規追加 */
.references-list {
  margin-bottom: 8px;
  max-height: 150px;
  overflow-y: auto;
}

.reference-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 6px 8px;
  margin-bottom: 4px;
  background: #f0f8ff;  /* 薄青 */
  border: 1px solid #b3d9ff;
  border-radius: 4px;
  font-size: 12px;
  gap: 8px;
}

.reference-label {
  flex: 1;
  color: #333;
}

/* 参照追加モード中のカーソル */
body.adding-reference-mode {
  cursor: crosshair !important;
}

body.adding-reference-mode * {
  cursor: crosshair !important;
}

/* 参照追加モード中のノードホバー */
body.adding-reference-mode .node:hover,
body.adding-reference-mode .cluster:hover,
body.adding-reference-mode .manual-memo-node:hover {
  opacity: 0.7;
  outline: 2px dashed #007bff;
}

/* ヘッダー通知 */
.reference-mode-notification {
  position: fixed;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  background: #007bff;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  z-index: 10000;
  font-size: 14px;
}
```

### 3. JavaScript

```javascript
// ✅ 新規追加

// グローバル変数
let isAddingReference = false;
let addingReferenceSourceId = null;

/**
 * 参照追加モードを開始
 */
function startAddingReference() {
  if (!selectedElement) {
    console.warn('[Reference] 要素が選択されていません');
    return;
  }

  const elementId = selectedElement.getAttribute('data-id') || selectedElement.id;

  isAddingReference = true;
  addingReferenceSourceId = elementId;

  // UI更新
  document.body.classList.add('adding-reference-mode');

  // 通知表示
  const notification = document.createElement('div');
  notification.id = 'reference-mode-notification';
  notification.className = 'reference-mode-notification';
  notification.textContent = '🖱 参照追加モード | クリックして関連ノードを選択... (ESCでキャンセル)';
  document.body.appendChild(notification);

  // ESCキーでキャンセル
  document.addEventListener('keydown', handleReferenceModeCancelKey);

  console.log('[Reference] 参照追加モード開始:', elementId);
}

/**
 * 参照追加モードをキャンセル
 */
function cancelAddingReference() {
  isAddingReference = false;
  addingReferenceSourceId = null;

  // UI更新
  document.body.classList.remove('adding-reference-mode');

  // 通知削除
  const notification = document.getElementById('reference-mode-notification');
  if (notification) {
    notification.remove();
  }

  // ESCキーリスナー削除
  document.removeEventListener('keydown', handleReferenceModeCancelKey);

  console.log('[Reference] 参照追加モード終了');
}

/**
 * ESCキーでキャンセル
 */
function handleReferenceModeCancelKey(event) {
  if (event.key === 'Escape') {
    cancelAddingReference();
  }
}

/**
 * ノードクリック時の処理（参照追加モード中）
 */
function handleNodeClickInReferenceMode(event) {
  if (!isAddingReference) return;

  const clickedElement = event.target.closest('[data-et="node"], [data-et="cluster"], .manual-memo-node');
  if (!clickedElement) return;

  const targetNodeId = clickedElement.getAttribute('data-id') || clickedElement.id;

  // 参照を追加
  const success = addReferenceByClick(addingReferenceSourceId, targetNodeId);

  if (success) {
    // UI更新
    updateElementReferencesUI(addingReferenceSourceId);

    // 自動保存
    autoSave();

    // モード終了
    cancelAddingReference();
  }

  event.preventDefault();
  event.stopPropagation();
}

/**
 * 参照を追加（重複・自己参照チェック付き）
 */
function addReferenceByClick(sourceElementId, targetNodeId) {
  // 初期化
  if (!elementReferences[sourceElementId]) {
    elementReferences[sourceElementId] = [];
  }

  // 重複チェック
  if (elementReferences[sourceElementId].includes(targetNodeId)) {
    alert('既に参照済みです');
    return false;
  }

  // 自己参照チェック
  if (sourceElementId === targetNodeId) {
    alert('自己参照はできません');
    return false;
  }

  // 参照を追加
  elementReferences[sourceElementId].push(targetNodeId);

  console.log(`[Reference] 参照追加: ${sourceElementId} → ${targetNodeId}`);
  return true;
}

/**
 * 参照リストUIを更新
 */
function updateElementReferencesUI(elementId) {
  const section = document.getElementById('element-references-section');
  const list = document.getElementById('element-references-list');

  if (!section || !list) return;

  const refs = elementReferences[elementId] || [];

  if (refs.length === 0) {
    section.style.display = 'none';
    return;
  }

  section.style.display = 'block';

  // リストをクリア
  list.innerHTML = '';

  // 各参照を表示
  for (const refId of refs) {
    const refElement = document.querySelector(`[data-id="${CSS.escape(refId)}"]`) || manualNodes[refId];
    const refLabel = refElement ? getElementDisplayText(refElement) : refId;

    const item = document.createElement('div');
    item.className = 'reference-item';
    item.innerHTML = `
      <span class="reference-label">→ ${refLabel}</span>
      <button class="btn-icon-small" onclick="removeReference('${elementId}', '${refId}')" title="参照を削除">×</button>
    `;

    list.appendChild(item);
  }
}

/**
 * 参照を削除
 */
function removeReference(sourceElementId, targetNodeId) {
  if (!elementReferences[sourceElementId]) return;

  const index = elementReferences[sourceElementId].indexOf(targetNodeId);
  if (index > -1) {
    elementReferences[sourceElementId].splice(index, 1);

    // 空配列になったらキーを削除
    if (elementReferences[sourceElementId].length === 0) {
      delete elementReferences[sourceElementId];
    }

    // UI更新
    updateElementReferencesUI(sourceElementId);

    // 自動保存
    autoSave();

    console.log(`[Reference] 参照削除: ${sourceElementId} → ${targetNodeId}`);
  }
}

/**
 * selectElement() に統合
 */
function selectElement(element) {
  // 既存処理...

  const elementId = element.getAttribute('data-id') || element.id;

  // ✅ 参照関係UIを更新
  updateElementReferencesUI(elementId);

  // 既存処理...
}

/**
 * DOMContentLoaded で初期化
 */
document.addEventListener('DOMContentLoaded', function() {
  // 既存処理...

  // ✅ フローチャートエリアにクリックリスナー追加
  const svgContainer = document.getElementById('svg-container');
  if (svgContainer) {
    svgContainer.addEventListener('click', handleNodeClickInReferenceMode, true);
  }
});
```

---

## 工数見積もり

| 項目 | 内容 | 工数 |
|------|------|------|
| 削除作業 | 既存実装の削除（300行） | 30分 |
| HTML/CSS実装 | 参照リストUI、モード通知 | 1時間 |
| JavaScript実装 | モード管理、クリックハンドラ、UI更新 | 1-1.5時間 |
| テスト | 動作確認、エッジケース検証 | 30分 |
| **合計** | | **3-3.5時間** |

**注**: 当初見積もり（2-3時間）から微増しているが、削除作業を含めても許容範囲内。

---

## テストケース

### 正常系

1. **参照追加**: メモノードから正規ノードへの参照を追加
2. **複数参照**: 1つの要素から複数の要素への参照を追加
3. **参照削除**: 既存の参照を削除
4. **モードキャンセル**: ESCキーで参照追加モードをキャンセル

### 異常系

1. **重複参照**: 既に参照済みの要素を再度追加（アラート表示）
2. **自己参照**: 同じ要素への参照を追加（アラート表示）
3. **モード中の他操作**: 参照追加モード中に他のボタンをクリック（モード継続）

### 永続化

1. **保存・復元**: SVG保存 → 再読込で参照関係が復元される
2. **マイグレーション**: v1.0ファイル読込で `elementReferences` が空で初期化される

---

## 更新履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-22 | 1.0 | 初版作成 - クリックto Add UX仕様を定義 |

---

**関連ドキュメント**:
- [01-design-principles.md](./01-design-principles.md) - 設計原則
- [02-data-structure.md](./02-data-structure.md) - v2.0メタデータスキーマ
- [03-implementation-plan.md](./03-implementation-plan.md) - 実装計画

**最終更新**: 2026-03-22
**ステータス**: 実装待ち（仕様確定済み）
