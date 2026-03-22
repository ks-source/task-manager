# 実装計画

## 🎯 実装フェーズの概要

**外部有識者レビュー（Opus）による修正**:
Phase 1完了時に「動いているもの」が確認できることが重要。データ構造のみではテストがしにくい。

### Phase 1: データ構造＋MVP UI（必須）
**工数**: 3-4時間
**目的**: メタデータの保存・復元機能 + 最小限のUI（参照表示のみ）

**完了条件**: SVG保存→読込→参照が表示される、というE2Eのフローが確認できる

### Phase 2: 完全UI実装（必須）
**工数**: 3-4時間
**目的**: 参照関係の入力・編集UI（タグ入力、オートコンプリート、ピッキングモード）

### Phase 3: 利便性向上（任意）
**工数**: 2-3時間
**目的**: 逆参照表示、ハイライト、画面遷移等

**総工数見積もり**:
- 最小（Phase 1-2のみ）: 6-8時間
- 推奨（Phase 1-3）: 8-11時間

---

## 📋 Phase 1: データ構造＋MVP UI

### 目標
- SVGメタデータv2.0への対応
- 参照関係とタスク連携情報の永続化
- **MVP UI**: 参照の表示のみ（読み取り専用、編集はPhase 2で実装）
- **E2Eフローの確認**: SVG保存→読込→参照表示のテスト可能性

### タスク一覧

#### 1.1 グローバル変数の追加
**ファイル**: `flowchart-editor.html`
**場所**: 既存の変数宣言部分（約3560行目付近）

```javascript
// ★新規追加: v2.0機能用
let elementReferences = {};   // { elementId: [refId1, refId2, ...] } - 参照関係管理
let elementStatuses = {};     // { elementId: statusKey } - Phase 4ステータス管理
```

**注意**: `elementStatuses` はPhase 4で使用するが、Phase 1でグローバル変数として定義のみ行う。

**工数**: 5分

---

#### 1.2 SVG保存時のメタデータ拡張
**ファイル**: `flowchart-editor.html`
**関数**: `exportEditedSvg()` （約5968行目）

**変更箇所1**: メタデータバージョンアップ
```javascript
// Before
const metadata = {
  version: "1.0",
  // ...
};

// After
const metadata = {
  version: "2.0",  // ★変更
  // ...
};
```

**変更箇所2**: taskFlowchartLinks追加
```javascript
// ★新規追加: タスク連携情報をメタデータに含める
const taskFlowchartLinks = {};
mockTasks.forEach(task => {
  taskFlowchartLinks[task.wbs] = {
    taskName: task.taskName,
    flowchartLinks: {
      nodes: task.flowchartLinks.nodes,
      edges: task.flowchartLinks.edges
    }
  };
});

metadata.taskFlowchartLinks = taskFlowchartLinks;
```

**変更箇所3**: elements に references を追加
```javascript
// 既存コード（約5985行目）
for (const elementId of allElementIds) {
  const element = document.querySelector(`[data-id="${elementId}"]`);
  if (!element) continue;

  metadata.elements[elementId] = {
    type: element.getAttribute('data-et') || 'unknown',
    displayText: getElementDisplayText(element),
    memo: elementMemos[elementId] || '',
    customLabel: elementCustomLabels[elementId] || '',
    originalLabel: originalLabels[elementId] || '',
    references: elementReferences[elementId] || []  // ★追加
  };
}
```

**変更箇所4**: manualNodes に references を追加
```javascript
// manualNodesをメタデータに追加する前に、referencesフィールドを確保
const manualNodesWithRefs = {};
for (const [nodeId, nodeData] of Object.entries(manualNodes)) {
  manualNodesWithRefs[nodeId] = {
    ...nodeData,
    references: nodeData.references || []  // ★追加（存在しない場合は空配列）
  };
}

metadata.manualNodes = manualNodesWithRefs;
```

**工数**: 30-45分

---

#### 1.3 SVG読込時の復元処理
**ファイル**: `flowchart-editor.html`
**場所**: SVG読込処理（約4968行目）

**変更箇所1**: elementReferences の復元
```javascript
// 既存のメモ・ラベル復元コードの後に追加
if (extractedMetadata.elements) {
  elementMemos = {};
  elementCustomLabels = {};
  elementReferences = {};  // ★新規追加

  for (const [elementId, elementData] of Object.entries(extractedMetadata.elements)) {
    if (elementData.memo) {
      elementMemos[elementId] = elementData.memo;
    }
    if (elementData.customLabel) {
      elementCustomLabels[elementId] = elementData.customLabel;
      // SVG DOMに反映
      const element = document.querySelector(`[data-id="${elementId}"]`);
      if (element) {
        updateSvgLabel(element, elementData.customLabel);
      }
    }
    if (elementData.references) {  // ★新規追加
      elementReferences[elementId] = elementData.references;
    }
  }
}
```

**変更箇所2**: manualNodes の references 復元
```javascript
// 既存のmanualNodes復元コード
if (extractedMetadata.manualNodes) {
  manualNodes = extractedMetadata.manualNodes;
  // ★ references が存在しない古い形式への後方互換性
  for (const nodeId of Object.keys(manualNodes)) {
    if (!manualNodes[nodeId].references) {
      manualNodes[nodeId].references = [];
    }
    // イベントハンドラを再アタッチ
    attachDragBehavior(nodeId);
  }
}
```

**変更箇所3**: taskFlowchartLinks の復元
```javascript
// ★新規追加: タスク連携情報の復元
if (extractedMetadata.taskFlowchartLinks) {
  mockTasks = [];
  for (const [wbs, linkData] of Object.entries(extractedMetadata.taskFlowchartLinks)) {
    mockTasks.push({
      wbs: wbs,
      taskName: linkData.taskName,
      mermaidIds: linkData.flowchartLinks.nodes.join(', '),
      flowchartLinks: linkData.flowchartLinks,
      status: "Not Started"  // デフォルト値
    });
  }

  console.log('[FlowchartEditor] Restored task links from SVG:', mockTasks.length, 'tasks');
  renderTaskList();  // タスク一覧を更新
}
```

**工数**: 30-45分

---

#### 1.4 参照の一貫性チェック関数
**ファイル**: `flowchart-editor.html`
**場所**: データ構造管理関数群（新規追加）

```javascript
/**
 * 参照の一貫性をチェック（存在しないIDへの参照を検出）
 */
function validateReferences(metadata) {
  const allIds = new Set([
    ...Object.keys(metadata.elements || {}),
    ...Object.keys(metadata.manualNodes || {}),
    ...Object.keys(metadata.manualEdges || {})
  ]);

  const errors = [];

  // elementsの参照をチェック
  for (const [elementId, elementData] of Object.entries(metadata.elements || {})) {
    if (!elementData.references) continue;

    for (const refId of elementData.references) {
      if (!allIds.has(refId)) {
        errors.push({ elementId, invalidRef: refId, type: 'element' });
      }
    }
  }

  // manualNodesの参照をチェック
  for (const [nodeId, nodeData] of Object.entries(metadata.manualNodes || {})) {
    if (!nodeData.references) continue;

    for (const refId of nodeData.references) {
      if (!allIds.has(refId)) {
        errors.push({ elementId: nodeId, invalidRef: refId, type: 'manualNode' });
      }
    }
  }

  return errors;
}

/**
 * 無効な参照をクリーンアップ
 */
function cleanupInvalidReferences(metadata, errors) {
  errors.forEach(error => {
    if (error.type === 'element') {
      const refs = metadata.elements[error.elementId].references;
      metadata.elements[error.elementId].references =
        refs.filter(id => id !== error.invalidRef);
      console.warn(`⚠️ Removed invalid reference: ${error.elementId} → ${error.invalidRef}`);
    } else if (error.type === 'manualNode') {
      const refs = metadata.manualNodes[error.elementId].references;
      metadata.manualNodes[error.elementId].references =
        refs.filter(id => id !== error.invalidRef);
      console.warn(`⚠️ Removed invalid reference: ${error.elementId} → ${error.invalidRef}`);
    }
  });
}
```

**SVG読込時に検証を実行**:
```javascript
// SVG読込処理の最後に追加
if (extractedMetadata) {
  const errors = validateReferences(extractedMetadata);
  if (errors.length > 0) {
    console.warn(`⚠️ Found ${errors.length} invalid references, cleaning up...`);
    cleanupInvalidReferences(extractedMetadata, errors);
  }
}
```

**工数**: 20-30分

---

#### 1.5 MVP UI実装（参照表示のみ）
**ファイル**: `flowchart-editor.html`
**場所**: 要素情報パネル + パネル更新関数

**目的**: Phase 1完了時に「動いているもの」が確認できるようにする

**HTML追加**:
```html
<!-- 要素情報パネル内に追加 -->
<div class="panel-section" id="references-section-mvp" style="display: none;">
  <h3 class="section-title">🔗 参照関係（References）</h3>
  <div class="section-content">
    <div class="reference-tags-readonly" id="reference-tags-mvp">
      <!-- 動的に生成: [U_START] [node_002] -->
    </div>
    <p class="section-note">
      （編集機能はPhase 2で実装予定）
    </p>
  </div>
</div>
```

**CSS追加**:
```css
.reference-tags-readonly {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  padding: 8px;
  background: #f8f9fa;
  border-radius: 4px;
}

.reference-tag-readonly {
  padding: 4px 8px;
  background: #e3f2fd;
  border: 1px solid #90caf9;
  border-radius: 12px;
  font-size: 12px;
  color: #1976d2;
}

.reference-tag-invalid {
  background: #ffebee;
  border-color: #ef5350;
  color: #c62828;
}
```

**JavaScript追加**:
```javascript
/**
 * MVP: 参照関係を表示（読み取り専用）
 */
function updateReferencesSectionMVP(elementId) {
  const section = document.getElementById('references-section-mvp');
  const tagsDisplay = document.getElementById('reference-tags-mvp');

  if (!elementId) {
    section.style.display = 'none';
    return;
  }

  const refs = elementReferences[elementId] || [];

  if (refs.length === 0) {
    section.style.display = 'none';
    return;
  }

  section.style.display = 'block';
  tagsDisplay.innerHTML = '';

  refs.forEach(refId => {
    const tag = document.createElement('div');

    // 参照先が存在するかチェック
    const refElement = document.querySelector(`[data-id="${CSS.escape(refId)}"]`);
    const exists = refElement || manualNodes[refId];

    tag.className = exists ? 'reference-tag-readonly' : 'reference-tag-readonly reference-tag-invalid';
    tag.textContent = `[${refId}]`;

    if (!exists) {
      tag.title = '⚠️ この要素は見つかりません（削除された可能性があります）';
    }

    tagsDisplay.appendChild(tag);
  });
}

// 既存のupdatePanelInfo()に追加
function updatePanelInfo(element) {
  // ... 既存のパネル更新コード ...

  // ★MVP UI: 参照関係を表示
  const elementId = element.getAttribute('data-id') || element.id;
  updateReferencesSectionMVP(elementId);
}
```

**工数**: 45分-1時間

---

### Phase 1 完了条件
- [ ] elementReferences グローバル変数が追加されている
- [ ] SVG保存時にv2.0メタデータが生成される
- [ ] taskFlowchartLinks がメタデータに含まれる（`_snapshotAt`付き）
- [ ] elements/manualNodes に references が含まれる
- [ ] SVG読込時にすべてのメタデータが復元される
- [ ] 無効な参照が自動的にクリーンアップされる
- [ ] **MVP UI: 要素を選択すると参照が表示される**
- [ ] **MVP UI: 無効な参照が警告色で表示される**

### テストシナリオ
1. SVGを読み込み、タスク連携を設定
2. **手動でelementReferencesにデータを追加**（コンソールから: `elementReferences["U_START"] = ["node_002", "MEMO_001"]`）
3. **要素を選択し、参照が表示されることを確認**
4. SVGを保存（_editable.svg）
5. ページをリロード
6. 保存したSVGを再読込
7. ✅ タスク連携情報が復元されることを確認
8. ✅ **要素を選択し、参照が表示されることを確認**（E2Eフロー確認）

---

## 🎨 Phase 2: UI実装

### 目標
- 要素情報パネルに参照関係セクションを追加
- タグ形式のUI + オートコンプリート

### タスク一覧

#### 2.1 HTML構造の追加
**ファイル**: `flowchart-editor.html`
**場所**: 要素情報パネルHTML（約2570-2680行目）

```html
<!-- 既存の「カスタム編集」セクションの後に追加 -->
<div class="panel-section" id="references-section" style="display: none;">
  <h3 class="section-title collapsible">
    🔗 参照関係（References）
    <span class="ref-count"></span>
  </h3>

  <div class="section-content">
    <p class="section-description">このノードが参照・関連する要素:</p>

    <!-- 既存の参照タグ表示エリア -->
    <div class="reference-tags" id="reference-tags-display">
      <!-- 動的に生成: [U_START] × [node_002] × -->
    </div>

    <!-- 参照追加UI -->
    <div class="add-reference-ui">
      <input
        type="text"
        id="reference-search-input"
        class="form-control"
        placeholder="要素IDまたは名前を入力..."
      />
      <button class="btn btn-small" onclick="addReference()">+ 追加</button>
    </div>

    <!-- オートコンプリート候補 -->
    <div class="autocomplete-dropdown" id="reference-autocomplete" style="display: none;">
      <!-- 動的に生成 -->
    </div>

    <p class="section-note">
      ℹ️ 参照関係は双方向ではありません（逆参照は別途設定が必要）
    </p>
  </div>
</div>
```

**工数**: 15分

---

#### 2.2 CSS スタイルの追加
**ファイル**: `flowchart-editor.html`
**場所**: `<style>` セクション

```css
/* 参照関係セクション */
.reference-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 10px;
  min-height: 30px;
  padding: 8px;
  background: #f8f9fa;
  border-radius: 4px;
}

.reference-tag {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  background: #e3f2fd;
  border: 1px solid #90caf9;
  border-radius: 12px;
  font-size: 12px;
  color: #1976d2;
}

.reference-tag .remove-btn {
  cursor: pointer;
  font-weight: bold;
  color: #d32f2f;
  margin-left: 4px;
}

.reference-tag .remove-btn:hover {
  color: #b71c1c;
}

.add-reference-ui {
  display: flex;
  gap: 8px;
  margin-bottom: 10px;
}

.add-reference-ui input {
  flex: 1;
}

.add-reference-ui .btn {
  white-space: nowrap;
}

.autocomplete-dropdown {
  position: absolute;
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  max-height: 200px;
  overflow-y: auto;
  z-index: 1000;
  width: 300px;
}

.autocomplete-item {
  padding: 8px 12px;
  cursor: pointer;
  border-bottom: 1px solid #f0f0f0;
}

.autocomplete-item:hover {
  background: #f5f5f5;
}

.autocomplete-item:last-child {
  border-bottom: none;
}

.autocomplete-item .element-id {
  font-weight: bold;
  color: #1976d2;
}

.autocomplete-item .element-label {
  font-size: 12px;
  color: #666;
  margin-left: 4px;
}

.ref-count {
  font-size: 12px;
  color: #666;
  font-weight: normal;
  margin-left: 4px;
}

.section-note {
  font-size: 11px;
  color: #666;
  margin-top: 8px;
  padding: 6px;
  background: #fff3cd;
  border-left: 3px solid #ffc107;
  border-radius: 3px;
}
```

**工数**: 20分

---

#### 2.3 参照関係表示関数
**ファイル**: `flowchart-editor.html`
**場所**: パネル更新関数群

```javascript
/**
 * 参照関係セクションを表示
 */
function updateReferencesSection(elementId) {
  const section = document.getElementById('references-section');
  const tagsDisplay = document.getElementById('reference-tags-display');
  const refCount = section.querySelector('.ref-count');

  if (!elementId) {
    section.style.display = 'none';
    return;
  }

  section.style.display = 'block';

  // 現在の参照リストを取得
  const refs = elementReferences[elementId] || [];

  // 件数表示
  refCount.textContent = refs.length > 0 ? `(${refs.length}件)` : '';

  // タグを生成
  tagsDisplay.innerHTML = '';
  if (refs.length === 0) {
    tagsDisplay.innerHTML = '<span style="color: #999;">（参照なし）</span>';
  } else {
    refs.forEach(refId => {
      const tag = createReferenceTag(refId, elementId);
      tagsDisplay.appendChild(tag);
    });
  }
}

/**
 * 参照タグ要素を生成
 */
function createReferenceTag(refId, parentElementId) {
  const tag = document.createElement('div');
  tag.className = 'reference-tag';
  tag.dataset.refId = refId;

  // ラベルを取得（要素が存在する場合）
  let displayLabel = refId;
  const refElement = document.querySelector(`[data-id="${refId}"]`);
  if (refElement) {
    const labelText = getElementDisplayText(refElement);
    if (labelText) {
      displayLabel = `${refId} - ${labelText}`;
    }
  } else if (manualNodes[refId]) {
    displayLabel = `${refId} - ${manualNodes[refId].label}`;
  }

  tag.innerHTML = `
    <span class="element-id">[${displayLabel}]</span>
    <span class="remove-btn" onclick="removeReference('${parentElementId}', '${refId}')">×</span>
  `;

  return tag;
}
```

**工数**: 30分

---

#### 2.4 オートコンプリート機能
**ファイル**: `flowchart-editor.html`
**場所**: 入力補助関数群

```javascript
/**
 * オートコンプリート候補を表示
 */
function showAutocomplete(query) {
  const dropdown = document.getElementById('reference-autocomplete');

  if (!query || query.length < 1) {
    dropdown.style.display = 'none';
    return;
  }

  // 全要素IDを収集
  const allIds = [
    ...Object.keys(elementMemos),
    ...Object.keys(elementCustomLabels),
    ...Object.keys(manualNodes),
    ...Object.keys(manualEdges)
  ];
  const uniqueIds = [...new Set(allIds)];

  // クエリにマッチする候補を検索（部分一致）
  const matches = uniqueIds.filter(id =>
    id.toLowerCase().includes(query.toLowerCase())
  ).slice(0, 10);  // 最大10件

  if (matches.length === 0) {
    dropdown.style.display = 'none';
    return;
  }

  // 候補リストを生成
  dropdown.innerHTML = '';
  matches.forEach(id => {
    const item = document.createElement('div');
    item.className = 'autocomplete-item';

    let label = '';
    const element = document.querySelector(`[data-id="${id}"]`);
    if (element) {
      label = getElementDisplayText(element);
    } else if (manualNodes[id]) {
      label = manualNodes[id].label;
    }

    item.innerHTML = `
      <span class="element-id">${id}</span>
      ${label ? `<span class="element-label">- ${label}</span>` : ''}
    `;

    item.onclick = () => selectAutocompleteItem(id);
    dropdown.appendChild(item);
  });

  dropdown.style.display = 'block';
}

/**
 * オートコンプリート候補を選択
 */
function selectAutocompleteItem(id) {
  const input = document.getElementById('reference-search-input');
  input.value = id;
  document.getElementById('reference-autocomplete').style.display = 'none';

  // 自動的に追加
  addReference();
}

// 入力イベントリスナーを設定
document.addEventListener('DOMContentLoaded', () => {
  const input = document.getElementById('reference-search-input');
  if (input) {
    input.addEventListener('input', (e) => {
      showAutocomplete(e.target.value);
    });

    // ESCキーでドロップダウンを閉じる
    input.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') {
        document.getElementById('reference-autocomplete').style.display = 'none';
      }
    });
  }
});
```

**工数**: 45分

---

#### 2.5 参照の追加・削除関数
**ファイル**: `flowchart-editor.html`
**場所**: データ操作関数群

```javascript
/**
 * 参照を追加
 */
function addReference() {
  const input = document.getElementById('reference-search-input');
  const refId = input.value.trim();

  if (!refId) {
    return;
  }

  if (!selectedElement) {
    alert('要素が選択されていません');
    return;
  }

  const elementId = selectedElement.getAttribute('data-id') || selectedElement.id;

  // 自己参照チェック
  if (refId === elementId) {
    alert('自分自身への参照はできません');
    input.value = '';
    return;
  }

  // 既に参照済みかチェック
  if (!elementReferences[elementId]) {
    elementReferences[elementId] = [];
  }

  if (elementReferences[elementId].includes(refId)) {
    alert('既に参照されています');
    input.value = '';
    return;
  }

  // 参照を追加
  elementReferences[elementId].push(refId);

  // UIを更新
  updateReferencesSection(elementId);

  // 入力欄をクリア
  input.value = '';
  document.getElementById('reference-autocomplete').style.display = 'none';

  // 自動保存をトリガー
  saveToFileAuto().catch(err => {
    console.warn('Auto-save failed:', err);
  });

  console.log(`✅ Reference added: ${elementId} → ${refId}`);
}

/**
 * 参照を削除
 */
function removeReference(elementId, refId) {
  if (!confirm(`参照を削除しますか？\n${elementId} → ${refId}`)) {
    return;
  }

  if (elementReferences[elementId]) {
    elementReferences[elementId] = elementReferences[elementId].filter(id => id !== refId);
  }

  // UIを更新
  updateReferencesSection(elementId);

  // 自動保存をトリガー
  saveToFileAuto().catch(err => {
    console.warn('Auto-save failed:', err);
  });

  console.log(`✅ Reference removed: ${elementId} → ${refId}`);
}
```

**工数**: 30分

---

#### 2.6 パネル更新処理の統合
**ファイル**: `flowchart-editor.html`
**関数**: `updatePanelInfo()` （既存関数を修正）

```javascript
function updatePanelInfo(element) {
  // 既存のパネル更新コード...

  // ★新規追加: 参照関係セクションを更新
  const elementId = element.getAttribute('data-id') || element.id;
  updateReferencesSection(elementId);
}
```

**工数**: 10分

---

### Phase 2 完了条件
- [ ] 要素情報パネルに参照関係セクションが表示される
- [ ] タグ形式で既存の参照が表示される
- [ ] オートコンプリートで要素を検索できる
- [ ] 参照を追加・削除できる
- [ ] 変更が自動保存される

### テストシナリオ
1. ノードを選択し、要素情報パネルを開く
2. 参照関係セクションで「node_」と入力
3. ✅ オートコンプリート候補が表示される
4. 候補を選択して追加
5. ✅ タグが表示される
6. ×ボタンをクリック
7. ✅ 参照が削除される

---

## 🚀 Phase 3: 利便性向上（任意）

### 目標
- 逆参照の自動表示
- 参照先ハイライト機能
- 参照先への画面遷移

### タスク一覧

#### 3.1 逆参照の表示
**工数**: 1時間

#### 3.2 参照先ハイライト
**工数**: 45分

#### 3.3 参照先への画面遷移
**工数**: 30分

---

## 🎨 Phase 4: ステータス色分け機能（6-9時間）

### 概要

フローチャート要素（正規ノード・サブグラフ・メモノード）に対して、視覚的なステータス（完了・進行中・未着手等）を設定し、task-manager.htmlと同一の色で色分け表示する機能。

**重要な設計原則**:
- フローチャートステータスとタスクステータスは**完全に独立**
- フローチャートステータス = 視覚的な状態管理（SVGに保存）
- タスクステータス = 実タスク進捗管理（JSONに保存）
- 両者のデータ連携は行わない

### タスク一覧

#### 4.1 Phase 4-1: 基本実装（3-4時間）

**タスク4.1.1: ステータス色定義**（30分）

```javascript
/**
 * ステータス色定義
 * task-manager.htmlのステータス仕様と完全同一
 */
const STATUS_COLORS = Object.freeze({
  'not-started': {
    bg: '#e2e3e5',
    text: '#383d41',
    border: '#6c757d',
    label: '未着手'
  },
  'in-progress': {
    bg: '#fff3cd',
    text: '#856404',
    border: '#ffc107',
    label: '進行中'
  },
  'completed': {
    bg: '#d4edda',
    text: '#155724',
    border: '#28a745',
    label: '完了'
  },
  'on-hold': {
    bg: '#ffe5d0',
    text: '#8a4a0e',
    border: '#fd7e14',
    label: '保留'
  },
  'postponed': {
    bg: '#f8d7da',
    text: '#721c24',
    border: '#dc3545',
    label: '延期'
  },
  'cancelled': {
    bg: '#f5f5f5',
    text: '#666',
    border: '#999',
    label: '中止'
  },
  'unknown': {
    bg: '#e9ecef',
    text: '#495057',
    border: '#adb5bd',
    label: '不明'
  }
});
```

**タスク4.1.2: buildMemoData()の拡張**（30分）

```javascript
function buildMemoData() {
  // ... 既存の処理 ...

  return {
    version: "2.0",
    schemaVersion: "2.0",
    exportedAt: new Date().toISOString(),
    svgFile: currentSvgFileName || 'unknown.svg',

    // 既存フィールド
    elements: elements,
    originalLabels: originalLabels,
    manualNodes: manualNodes,
    manualEdges: manualEdges,

    // v2.0新機能
    references: elementReferences,
    taskFlowchartLinks: buildTaskLinks(),
    elementStatuses: elementStatuses  // ★Phase 4追加
  };
}
```

**タスク4.1.3: 右クリックメニューUI**（1.5-2時間）

HTML構造:
```html
<div id="status-context-menu" class="status-context-menu" style="display: none;">
  <div class="status-menu-header">ステータス設定</div>
  <div class="status-menu-item" data-status="not-started">
    <span class="status-dot" style="background: #6c757d;"></span>
    <span>未着手</span>
  </div>
  <!-- 他のステータス項目 -->
  <div class="status-menu-divider"></div>
  <div class="status-menu-item status-clear">
    <span>✗ ステータスをクリア</span>
  </div>
</div>
```

イベントハンドラ:
```javascript
// 右クリックでメニュー表示
svg.addEventListener('contextmenu', (e) => {
  const target = e.target.closest('[data-id], [data-cluster-id], [data-memo-id]');
  if (!target) return;

  e.preventDefault();

  const elementId = target.getAttribute('data-id') ||
                    target.getAttribute('data-cluster-id') ||
                    target.getAttribute('data-memo-id');

  showStatusContextMenu(e.clientX, e.clientY, elementId);
});
```

**タスク4.1.4: 正規ノードへのステータス適用**（1-1.5時間）

```javascript
/**
 * 正規ノードにステータス色を適用（CSS.escape使用）
 */
function applyStatusToNode(nodeId, status) {
  const nodeGroup = svg.querySelector(`[data-id="${CSS.escape(nodeId)}"]`);
  if (!nodeGroup) return;

  const colorDef = STATUS_COLORS[status];
  if (!colorDef) return;

  const shape = nodeGroup.querySelector('rect, circle, polygon, path');
  if (shape) {
    shape.setAttribute('fill', colorDef.bg);
    shape.setAttribute('stroke', colorDef.border);
    shape.setAttribute('stroke-width', '2');
  }

  const text = nodeGroup.querySelector('text');
  if (text) {
    text.setAttribute('fill', colorDef.text);
  }

  nodeGroup.setAttribute('data-status', status);
}
```

#### 4.2 Phase 4-2: 拡張機能（2-3時間）

**タスク4.2.1: サブグラフへのステータス適用**（1時間）

```javascript
/**
 * サブグラフにステータス色を適用（背景を30%透明化）
 */
function applyStatusToSubgraph(clusterId, status) {
  const cluster = svg.querySelector(`[data-cluster-id="${CSS.escape(clusterId)}"]`);
  if (!cluster) return;

  const colorDef = STATUS_COLORS[status];
  if (!colorDef) return;

  const rect = cluster.querySelector('rect');
  if (rect) {
    // 背景を30%透明化
    const fadedBg = adjustAlpha(colorDef.bg, 0.3);
    rect.setAttribute('fill', fadedBg);
    rect.setAttribute('stroke', colorDef.border);
    rect.setAttribute('stroke-width', '2');
  }

  cluster.setAttribute('data-status', status);
}
```

**タスク4.2.2: メモノードへのステータス適用**（30分）

```javascript
/**
 * メモノードにステータス色を適用
 */
function applyStatusToMemoNode(memoId, status) {
  const memoNode = document.querySelector(`[data-memo-id="${CSS.escape(memoId)}"]`);
  if (!memoNode) return;

  const colorDef = STATUS_COLORS[status];
  if (!colorDef) return;

  const content = memoNode.querySelector('.memo-content');
  if (content) {
    content.style.background = colorDef.bg;
    content.style.borderColor = colorDef.border;
    content.style.color = colorDef.text;
  }

  memoNode.setAttribute('data-status', status);
}
```

**タスク4.2.3: ステータスクリア機能**（30分）

**タスク4.2.4: マイグレーション処理統合**（30分）

```javascript
function migrateMetadata(data) {
  if (data.version === "2.0" || data.schemaVersion === "2.0") {
    return data;
  }

  let migrated = structuredClone(data);

  migrated.version = "2.0";
  migrated.schemaVersion = "2.0";

  // ★Phase 4: ステータス管理の初期化
  if (!migrated.elementStatuses) {
    migrated.elementStatuses = {};
  }

  // ★参照関係管理の初期化
  if (!migrated.references) {
    migrated.references = {};
  }

  // ★タスク連携の初期化
  if (!migrated.taskFlowchartLinks) {
    migrated.taskFlowchartLinks = {
      nodes: {},
      _snapshotAt: null
    };
  }

  return migrated;
}
```

#### 4.3 Phase 4-3: UX改善（1-2時間）

**タスク4.3.1: ステータスアイコンバッジ**（45分）

**タスク4.3.2: フィルタリング機能拡張**（45分）

**タスク4.3.3: 一括ステータス設定**（30分、オプション）

### 工数見積もり

| フェーズ | 内容 | 工数 |
|---------|------|------|
| Phase 4-1 | 基本実装（正規ノード） | 3-4h |
| Phase 4-2 | 拡張（サブグラフ・メモノード・マイグレーション） | 2-3h |
| Phase 4-3 | UX改善（バッジ・フィルタリング） | 1-2h |
| **合計** | | **6-9h** |

### 成功基準

- ✅ 正規ノード・サブグラフ・メモノードにステータスを設定できる
- ✅ task-manager.htmlと同一の7種類のステータス色が適用される
- ✅ 右クリックメニューでステータス変更できる
- ✅ ステータス情報がSVGメタデータに保存・復元される
- ✅ v1.0ファイルがv2.0へ正常にマイグレーションされる

---

## 📝 更新履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-22 | v1.2 | Phase 4（ステータス色分け機能）の統合<br>- Phase 4セクション追加（基本実装・拡張・UX改善）<br>- グローバル変数にelementStatuses追加<br>- buildMemoData()にelementStatuses追加<br>- マイグレーション処理統合版を追加<br>- 総工数見積もりを更新（12-17h or 14-20h） |
| 2026-03-22 | v1.1 | 外部有識者レビュー（Opus）を反映<br>- Phase 1にMVP UI（参照表示のみ）を追加<br>- タスク1.5「MVP UI実装」を追加<br>- 完了条件にMVP UI関連項目を追加<br>- テストシナリオを更新（E2Eフロー確認）<br>- 総工数見積もりを更新（6-8時間） |
| 2026-03-22 | v1.0 | 初版作成（外部レビュー前） |

---

**作成日**: 2026-03-22
**最終更新日**: 2026-03-22
**現在バージョン**: v1.2
**推奨実装順序**: Phase 1（データ＋MVP UI） → Phase 2（完全UI） → Phase 3（利便性） → Phase 4（ステータス色分け）

**総工数見積もり（Phase 4統合版）**:
- **最小（Phase 1-2のみ）**: 6-8時間
- **推奨（Phase 1-3）**: 8-11時間
- **Phase 4追加（Phase 1-3 + Phase 4）**: 14-20時間
- **完全版（Phase 1-4）**: 14-20時間
