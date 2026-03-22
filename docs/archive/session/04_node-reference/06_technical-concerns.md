# 技術的検討事項・懸念点

## 🚨 セキュリティ懸念

### 1. XSS（クロスサイトスクリプティング）

**問題**:
参照IDや要素ラベルをHTMLに直接挿入する際、悪意のあるスクリプトが実行される可能性

**リスクシナリオ**:
```javascript
// 悪意のある要素ID
const maliciousId = "<img src=x onerror='alert(1)'>";

// 危険な実装例
tag.innerHTML = `<span>[${maliciousId}]</span>`;  // ❌ XSS脆弱性
```

**対策**:
```javascript
// 安全な実装
function escapeHtml(text) {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}

tag.innerHTML = `<span>[${escapeHtml(refId)}]</span>`;  // ✅ 安全
```

**実装優先度**: 🔴 高（Phase 2で必須）

**追加推奨**: `addEventListener` の使用
```javascript
// ❌ 危険: onclick属性でXSSリスク
tag.innerHTML = `<span onclick="removeReference('${id}')">×</span>`;

// ⭕ 安全: addEventListener
const removeBtn = tag.querySelector('.remove-btn');
removeBtn.addEventListener('click', () => removeReference(id));
```

---

### 2. CSSセレクタインジェクション（外部レビュー追加）

**問題**:
`document.querySelector()` に特殊文字を含むIDを渡すと、想定外のセレクタ解釈が起こる

**リスクシナリオ**:
```javascript
const id = 'node[data-dangerous="true"]';  // 悪意のあるID
document.querySelector(`[data-id="${id}"]`);  // ❌ セレクタが壊れる
```

**対策**: `CSS.escape()` を使用
```javascript
// ✅ 安全な実装
const id = 'node[data-dangerous="true"]';
document.querySelector(`[data-id="${CSS.escape(id)}"]`);
```

**実装箇所**:
- `updateReferencesSectionMVP()`: 参照先の存在チェック時
- `showAutocomplete()`: オートコンプリート候補生成時
- `getAllReferenceableIds()`: 全要素ID取得時
- その他、動的にセレクタを生成する全箇所

**実装優先度**: 🔴 高（Phase 1で必須）

---

### 3. LocalStorage容量制限

**問題**:
LocalStorageの容量制限（通常5-10MB）を超える可能性

**リスクシナリオ**:
```javascript
// 大規模プロジェクト: 100タスク × 平均10ノード/タスク = 1000ノード
// taskFlowchartLinks + flowchart-attributes が肥大化
```

**対策**:
```javascript
try {
  localStorage.setItem('flowchart-attributes', JSON.stringify(flowchartAttrs));
} catch (e) {
  if (e.name === 'QuotaExceededError') {
    console.error('❌ LocalStorage容量超過');
    alert('データ容量が大きすぎます。古いデータを削除してください。');
    // フォールバック: 最小限のデータのみ保存
    const minimalData = {
      /* 重要なデータのみ */
    };
    localStorage.setItem('flowchart-attributes', JSON.stringify(minimalData));
  }
}
```

**実装優先度**: 🟡 中（Phase 1で検討）

---

### 3. データインジェクション攻撃

**問題**:
SVGコメントに埋め込まれたJSONメタデータを改ざんされる可能性

**リスクシナリオ**:
```xml
<!-- FLOWCHART-MANUAL-DATA
{
  "version": "2.0",
  "elements": {
    "node_001": {
      "memo": "'; DROP TABLE tasks; --",  // SQLインジェクション風
      "references": ["../../../etc/passwd"]  // パストラバーサル風
    }
  }
}
-->
```

**対策**:
1. **入力検証**: 参照IDのフォーマットチェック
```javascript
function isValidElementId(id) {
  // 英数字、アンダースコア、ハイフンのみ許可
  return /^[a-zA-Z0-9_-]+$/.test(id);
}

function addReference() {
  const refId = input.value.trim();

  if (!isValidElementId(refId)) {
    alert('無効な要素IDです（英数字、_、-のみ使用可能）');
    return;
  }
  // ...
}
```

2. **サニタイゼーション**: メタデータ読込時の検証
```javascript
function sanitizeMetadata(metadata) {
  if (metadata.elements) {
    for (const [elementId, data] of Object.entries(metadata.elements)) {
      if (data.references) {
        data.references = data.references.filter(id => isValidElementId(id));
      }
    }
  }
  return metadata;
}
```

**実装優先度**: 🔴 高（Phase 1で必須）

---

### 4. Prototype Pollution（外部レビュー追加）

**問題**:
外部SVG内のJSONをそのままオブジェクトにマージすると、`__proto__` などを含む不正データで予期しない挙動を招く

**リスクシナリオ**:
```javascript
const maliciousMetadata = {
  "__proto__": {
    "isAdmin": true
  },
  "elements": { /* ... */ }
};

// ❌ 危険: プロトタイプ汚染のリスク
Object.assign(globalState, maliciousMetadata);
```

**対策**:
1. **キーのホワイトリスト検証**
```javascript
const ALLOWED_METADATA_KEYS = ['version', 'exportedAt', 'svgFile', 'elements',
                                'manualNodes', 'manualEdges', 'taskFlowchartLinks',
                                'originalLabels'];

function sanitizeMetadata(metadata) {
  const sanitized = {};
  for (const key of ALLOWED_METADATA_KEYS) {
    if (key in metadata) {
      sanitized[key] = metadata[key];
    }
  }
  return sanitized;
}
```

2. **`Object.create(null)` の活用**
```javascript
// プロトタイプチェーンを持たないオブジェクト
const elementReferences = Object.create(null);
```

3. **`Map` ベース管理の検討**
```javascript
// より安全なデータ構造
const elementReferences = new Map();
elementReferences.set(id, [refId1, refId2]);
```

**実装優先度**: 🟡 中（Phase 1で検討、Phase 2で実装）

---

### 5. SVG自体の危険性（外部レビュー追加）

**問題**:
読み込むSVGに `<script>`, `foreignObject`, イベント属性などが含まれていた場合、表示方法によっては危険

**リスクシナリオ**:
```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <script>alert('XSS')</script>  <!-- 実行される可能性 -->
  <circle onload="alert('XSS')" />  <!-- イベントハンドラ -->
  <foreignObject>
    <iframe src="evil.com"></iframe>  <!-- 外部コンテンツ -->
  </foreignObject>
</svg>
```

**対策**: SVG読込時のサニタイズ
```javascript
function sanitizeSVG(svgString) {
  // <script> タグを削除
  svgString = svgString.replace(/<script[^>]*>[\s\S]*?<\/script>/gi, '');

  // on* イベント属性を削除
  svgString = svgString.replace(/\son\w+="[^"]*"/g, '');
  svgString = svgString.replace(/\son\w+='[^']*'/g, '');

  // foreignObject を削除
  svgString = svgString.replace(/<foreignObject[^>]*>[\s\S]*?<\/foreignObject>/gi, '');

  return svgString;
}

// SVG読込時に適用
async function loadSVGFile(file) {
  const svgText = await file.text();
  const sanitizedSvgText = sanitizeSVG(svgText);  // ★サニタイズ

  // メタデータ抽出
  const metadataMatch = sanitizedSvgText.match(/<!--\s*FLOWCHART-MANUAL-DATA\s*([\s\S]*?)-->/);
  // ...
}
```

**重要**: メタデータのJSONだけ安全でも、SVG本体が危険なら意味がありません。

**実装優先度**: 🔴 高（Phase 1で必須）

---

## ⚡ パフォーマンス懸念

### 1. 大規模フローチャートでのオートコンプリート

**問題**:
1000+要素の場合、オートコンプリート検索が遅延する

**リスクシナリオ**:
```javascript
// 1000要素を毎回全検索
const matches = uniqueIds.filter(id =>
  id.toLowerCase().includes(query.toLowerCase())
);  // O(n) × 文字列操作
```

**対策**:
```javascript
// 1. 最初の3文字入力後に検索開始
function showAutocomplete(query) {
  if (!query || query.length < 3) {  // ★変更: 1 → 3
    dropdown.style.display = 'none';
    return;
  }
  // ...
}

// 2. デバウンス（入力後300ms待機）
let autocompleteTimer = null;
input.addEventListener('input', (e) => {
  clearTimeout(autocompleteTimer);
  autocompleteTimer = setTimeout(() => {
    showAutocomplete(e.target.value);
  }, 300);  // 300ms デバウンス
});

// 3. 候補数を制限
const matches = uniqueIds
  .filter(id => id.toLowerCase().includes(query.toLowerCase()))
  .slice(0, 20);  // ★最大20件（10 → 20）

// 4. 前方一致を優先（オプション）
const matches = uniqueIds
  .filter(id => id.toLowerCase().startsWith(query.toLowerCase()))  // 前方一致優先
  .slice(0, 10)
  .concat(
    uniqueIds
      .filter(id => id.toLowerCase().includes(query.toLowerCase()) &&
                    !id.toLowerCase().startsWith(query.toLowerCase()))  // 部分一致
      .slice(0, 10)
  );
```

**実装優先度**: 🟡 中（Phase 2で実装）

---

### 2. 循環参照による無限ループ

**問題**:
参照をたどる際の無限ループリスク

**リスクシナリオ**:
```javascript
// A → B → C → A の循環参照
elementReferences = {
  "A": ["B"],
  "B": ["C"],
  "C": ["A"]
};

// 危険な再帰処理
function getAllRelatedElements(elementId, result = []) {
  result.push(elementId);
  const refs = elementReferences[elementId] || [];
  refs.forEach(refId => {
    getAllRelatedElements(refId, result);  // ❌ 無限ループ
  });
  return result;
}
```

**対策**:
```javascript
// 訪問済み要素を記録
function getAllRelatedElements(elementId, visited = new Set()) {
  if (visited.has(elementId)) {
    return [];  // 既に訪問済み
  }

  visited.add(elementId);
  const result = [elementId];

  const refs = elementReferences[elementId] || [];
  refs.forEach(refId => {
    const subResult = getAllRelatedElements(refId, new Set(visited));
    result.push(...subResult);
  });

  return result;
}

// または深さ制限
function getAllRelatedElements(elementId, visited = new Set(), depth = 0) {
  if (depth > 10) {  // 最大10階層
    console.warn('⚠️ 参照の深さが10を超えました');
    return [];
  }
  // ...
}
```

**実装優先度**: 🔴 高（Phase 1で必須）

---

### 3. SVGファイルサイズの肥大化

**問題**:
メタデータの増加によるSVGファイルサイズの肥大化

**現状分析**:
```
通常のSVGファイル: 200KB
メタデータv1.0: +30KB (elements, manualNodes, manualEdges)
メタデータv2.0: +50KB (references, taskFlowchartLinks追加)

合計: 250KB（+25%増加）
```

**許容範囲**: 300KB以下

**対策**:
```javascript
// 1. 不要なメタデータの削除（空の参照配列等）
function optimizeMetadata(metadata) {
  // 空の references を削除
  for (const [id, data] of Object.entries(metadata.elements)) {
    if (data.references && data.references.length === 0) {
      delete data.references;
    }
  }

  // 空のメモを削除
  for (const [id, data] of Object.entries(metadata.elements)) {
    if (!data.memo) {
      delete data.memo;
    }
  }

  return metadata;
}

// 2. JSON圧縮（改行・スペース削除）
const metadataJson = JSON.stringify(metadata);  // 圧縮形式（改行なし）
// vs
const metadataJson = JSON.stringify(metadata, null, 2);  // 人間可読（改行あり）
```

**実装優先度**: 🟢 低（Phase 3で検討）

---

## 🔍 データ整合性懸念

### 1. 存在しない要素への参照

**問題**:
要素が削除された後も参照が残る

**リスクシナリオ**:
```javascript
// 1. U_START → node_002 の参照を設定
elementReferences["U_START"] = ["node_002"];

// 2. node_002をSVGから削除（新しいSVGをアップロード）

// 3. 参照が孤立（orphaned reference）
elementReferences["U_START"] = ["node_002"];  // ❌ node_002は存在しない
```

**対策**:
Phase 1で実装済みの `validateReferences()` + `cleanupInvalidReferences()` で対応

**追加対策**: UI上で警告表示
```javascript
function createReferenceTag(refId, parentElementId) {
  const tag = document.createElement('div');

  // 参照先が存在するかチェック
  const refElement = document.querySelector(`[data-id="${refId}"]`);
  const exists = refElement || manualNodes[refId];

  if (!exists) {
    tag.className = 'reference-tag reference-tag-invalid';  // ★警告スタイル
    tag.title = '⚠️ この要素は見つかりません（削除された可能性があります）';
  } else {
    tag.className = 'reference-tag';
  }
  // ...
}
```

**CSS**:
```css
.reference-tag-invalid {
  background: #ffebee !important;
  border-color: #ef5350 !important;
  color: #c62828 !important;
}
```

**実装優先度**: 🟡 中（Phase 2で実装）

---

### 2. 双方向参照の管理

**問題**:
A → B と B → A の双方向参照を手動で設定する必要がある

**ユーザー体験**:
```
ユーザーの期待:
- A → B を設定したら、B → A も自動設定されてほしい

現在の仕様:
- 一方向のみ（明示的な設定が必要）
```

**代替案**:
```javascript
// オプション: 双方向参照の自動設定（チェックボックスで制御）
function addReference(createBidirectional = false) {
  // 通常の参照を追加
  elementReferences[elementId].push(refId);

  // 双方向フラグがONの場合
  if (createBidirectional) {
    if (!elementReferences[refId]) {
      elementReferences[refId] = [];
    }
    if (!elementReferences[refId].includes(elementId)) {
      elementReferences[refId].push(elementId);
      console.log(`✅ Bidirectional reference created: ${refId} → ${elementId}`);
    }
  }
}
```

**UI案**:
```html
<div class="add-reference-ui">
  <input type="text" id="reference-search-input" />
  <button onclick="addReference()">+ 追加</button>
  <label>
    <input type="checkbox" id="bidirectional-checkbox" />
    双方向
  </label>
</div>
```

**実装優先度**: 🟢 低（Phase 3で検討）

---

### 3. タスク連携との二重管理

**問題**:
`mockTasks.flowchartLinks` と `elements[id].references` の二重管理

**データの重複**:
```javascript
// タスク連携
mockTasks[0].flowchartLinks.nodes = ["U_START", "node_002"];

// 要素の参照（重複？）
elementReferences["U_START"] = ["task_0.1"];  // タスクへの逆参照？
```

**設計方針**:
- **タスク連携**: タスク → フローチャート要素（所有関係）
- **参照関係**: 要素 ↔ 要素（関連関係）

**明確化**:
両者は異なる意味を持つため、二重管理ではない。
- `mockTasks.flowchartLinks`: "このタスクはこれらの要素を含む"
- `elementReferences`: "この要素はこれらの要素を参照する"

**実装優先度**: N/A（設計上の整理のみ）

---

## 📱 UI/UX懸念

### 1. モバイル対応

**問題**:
タグ入力UIはモバイルで扱いにくい

**対策**:
```css
/* レスポンシブデザイン */
@media (max-width: 768px) {
  .reference-tags {
    flex-direction: column;
  }

  .add-reference-ui {
    flex-direction: column;
  }

  .autocomplete-dropdown {
    width: 100%;
  }
}
```

**実装優先度**: 🟢 低（モバイル対応は優先度低）

---

### 2. アクセシビリティ

**問題**:
キーボードナビゲーション、スクリーンリーダー対応

**対策**:
```html
<div
  class="reference-tag"
  role="button"
  tabindex="0"
  aria-label="参照: U_START - ユーザー。削除するにはEnterキーを押してください"
>
  <span>[U_START]</span>
  <span class="remove-btn" onclick="removeReference(...)">×</span>
</div>
```

**実装優先度**: 🟢 低（Phase 3で検討）

---

### 3. 参照関係の複雑化

**問題**:
多数の参照を持つ要素の視認性

**リスクシナリオ**:
```javascript
// 20個の参照を持つ要素
elementReferences["MEMO_001"] = [
  "U_START", "node_002", "node_003", ...  // 20個
];

// UIが煩雑になる
[U_START] × [node_002] × [node_003] × ... (20個のタグ)
```

**対策**:
```html
<!-- 5個以上の場合は折りたたみ -->
<div class="reference-tags">
  <div class="tag">[U_START] ×</div>
  <div class="tag">[node_002] ×</div>
  <div class="tag">[node_003] ×</div>
  <div class="tag">[node_004] ×</div>
  <div class="tag">[node_005] ×</div>
  <button class="show-more-btn">+15件を表示</button>
</div>
```

**実装優先度**: 🟡 中（Phase 2で検討）

---

## 🧪 テスト懸念

### 1. エッジケースのテスト

**テストすべきシナリオ**:
- [ ] 自己参照の防止（A → A）
- [ ] 循環参照の検出（A → B → A）
- [ ] 存在しない要素への参照
- [ ] 参照数0の要素
- [ ] 参照数100+の要素（UI破綻チェック）
- [ ] 無効なID形式（特殊文字、スペース等）
- [ ] LocalStorage容量超過
- [ ] メタデータv1.0 → v2.0の移行

**実装優先度**: 🔴 高（Phase 2完了時に必須）

---

### 2. パフォーマンステスト

**テストすべきシナリオ**:
- [ ] 1000ノードのフローチャート
- [ ] 500参照関係
- [ ] オートコンプリート応答時間（<300ms）
- [ ] SVGファイルサイズ（<500KB）

**実装優先度**: 🟡 中（Phase 3で実施）

---

## 🔄 後方互換性

### メタデータv1.0からv2.0への移行

**問題**:
既存のv1.0メタデータを持つSVGファイルとの互換性

**対策**:
```javascript
// SVG読込時にバージョンチェック
if (extractedMetadata) {
  if (extractedMetadata.version === "1.0") {
    console.log('ℹ️ メタデータv1.0を検出、v2.0に自動アップグレード');

    // v2.0フィールドを追加（空で初期化）
    if (!extractedMetadata.taskFlowchartLinks) {
      extractedMetadata.taskFlowchartLinks = {};
    }

    // elementsにreferencesフィールドを追加
    for (const [id, data] of Object.entries(extractedMetadata.elements || {})) {
      if (!data.references) {
        data.references = [];
      }
    }

    // manualNodesにreferencesフィールドを追加
    for (const [id, data] of Object.entries(extractedMetadata.manualNodes || {})) {
      if (!data.references) {
        data.references = [];
      }
    }

    // バージョンを更新
    extractedMetadata.version = "2.0";
  }

  // 通常の復元処理へ
  restoreMetadata(extractedMetadata);
}
```

**実装優先度**: 🔴 高（Phase 1で必須）

---

## 📋 まとめ：優先順位別実装リスト

### 🔴 高優先度（Phase 1-2で必須）
- [ ] XSS対策（escapeHtml実装）
- [ ] データインジェクション対策（入力検証）
- [ ] 循環参照対策（無限ループ防止）
- [ ] 後方互換性（v1.0 → v2.0移行）
- [ ] エッジケースのテスト

### 🟡 中優先度（Phase 2-3で実装）
- [ ] オートコンプリートのパフォーマンス最適化
- [ ] 存在しない参照の警告表示
- [ ] 多数の参照の折りたたみ表示
- [ ] LocalStorage容量超過対策
- [ ] パフォーマンステスト

### 🟢 低優先度（Phase 3以降で検討）
- [ ] SVGファイルサイズ最適化
- [ ] 双方向参照の自動設定
- [ ] モバイル対応
- [ ] アクセシビリティ向上

---

## 🎨 Phase 4: ステータス色分け機能 固有の懸念

### 1. セキュリティ: ステータスキー検証

**問題**:
悪意のあるステータスキーによるPrototype Pollution

**リスクシナリオ**:
```javascript
// 悪意のあるメタデータ
const maliciousMetadata = {
  "elementStatuses": {
    "node_001": "completed",
    "__proto__": "admin",  // ❌ プロトタイプ汚染
    "constructor": "malicious"
  }
};

// ❌ 危険: そのまま代入
elementStatuses = maliciousMetadata.elementStatuses;
```

**対策**:
```javascript
// ステータスキーのホワイトリスト
const VALID_STATUS_KEYS = [
  'not-started', 'in-progress', 'completed',
  'on-hold', 'postponed', 'cancelled', 'unknown'
];

/**
 * ステータスデータの検証
 */
function validateElementStatuses(statuses) {
  if (typeof statuses !== 'object' || statuses === null) {
    return {};
  }

  const validated = {};

  for (const [elementId, statusKey] of Object.entries(statuses)) {
    // Prototype Pollution対策
    if (elementId === '__proto__' || elementId === 'constructor' || elementId === 'prototype') {
      console.warn(`[Security] プロトタイプ汚染の試みを検出: ${elementId}`);
      continue;
    }

    // ステータスキーの検証
    if (!VALID_STATUS_KEYS.includes(statusKey)) {
      console.warn(`[Validation] 無効なステータスキー: ${statusKey}（要素: ${elementId}）`);
      continue;
    }

    validated[elementId] = statusKey;
  }

  return validated;
}

/**
 * restoreMemoData()内でのステータス復元（安全版）
 */
function restoreMemoData(data) {
  // ... 既存の復元処理 ...

  // ★Phase 4: ステータスデータの復元（検証付き）
  const rawStatuses = migratedData.elementStatuses || {};
  elementStatuses = validateElementStatuses(rawStatuses);

  console.log(`[Restore] ${Object.keys(elementStatuses).length}個のステータスを復元`);

  // ステータス色を再適用
  applyAllStatuses();
}
```

**実装優先度**: 🔴 高（Phase 4-2で必須）

---

### 2. セキュリティ: CSS色値インジェクション（低リスク）

**問題**:
STATUS_COLORSは定数なので実質的にリスクなしだが、将来カスタム色を許可する場合は注意が必要

**リスクシナリオ**（将来拡張時）:
```javascript
// ❌ 危険: ユーザー定義色を許可した場合
const customColor = "red; } body { display: none; } .fake {";
shape.setAttribute('fill', customColor);  // CSSインジェクション
```

**対策**（将来拡張用）:
```javascript
/**
 * 色値の検証（16進数カラーコードのみ許可）
 */
function validateColor(color) {
  // #RRGGBB または #RGB 形式のみ許可
  if (/^#[0-9A-Fa-f]{6}$/.test(color) || /^#[0-9A-Fa-f]{3}$/.test(color)) {
    return color;
  }

  console.warn(`[Security] 無効な色値: ${color}`);
  return '#FFFFFF';  // デフォルト色
}

// 使用例
const bgColor = validateColor(customBg);
shape.setAttribute('fill', bgColor);
```

**実装優先度**: 🟢 低（現状は定数のみなので不要、将来拡張時のみ）

---

### 3. パフォーマンス: 一括ステータス適用の最適化

**問題**:
1000要素に一括でステータスを適用すると、DOM操作が遅延する

**リスクシナリオ**:
```javascript
// ❌ 非効率: 要素ごとにDOM操作
for (const [elementId, status] of Object.entries(elementStatuses)) {
  applyStatusToNode(elementId, status);  // 各要素でDOM更新
  applyStatusToSubgraph(elementId, status);
  applyStatusToManualNode(elementId, status);
}
// 結果: 1000要素 × DOM操作 = 数秒の遅延
```

**対策**:
```javascript
/**
 * 一括ステータス適用（最適化版）
 */
function applyAllStatuses() {
  // DocumentFragmentは不要（既存SVG内の要素を変更するため）
  // 代わりにバッチ処理でリフローを最小化

  const startTime = performance.now();

  // 1回のループで全タイプを処理
  for (const [elementId, statusKey] of Object.entries(elementStatuses)) {
    const colorDef = STATUS_COLORS[statusKey];
    if (!colorDef) continue;

    // 正規ノード
    const nodeGroup = svg.querySelector(`[data-id="${CSS.escape(elementId)}"]`);
    if (nodeGroup && nodeGroup.hasAttribute('data-et')) {
      applyStatusToElement(nodeGroup, colorDef, statusKey);
      continue;
    }

    // サブグラフ
    const subgraph = svg.querySelector(`g.cluster[data-subgraph-name="${CSS.escape(elementId)}"]`);
    if (subgraph) {
      applyStatusToSubgraphElement(subgraph, colorDef, statusKey);
      continue;
    }

    // メモノード
    const memoNode = manualNodes[elementId];
    if (memoNode && memoNode.svgGroup) {
      applyStatusToManualNodeElement(memoNode.svgGroup, colorDef, statusKey);
    }
  }

  const duration = performance.now() - startTime;
  console.log(`[Performance] ${Object.keys(elementStatuses).length}個のステータスを${duration.toFixed(2)}msで適用`);
}

/**
 * 共通の適用処理（リフロー最小化）
 */
function applyStatusToElement(group, colorDef, statusKey) {
  const shape = group.querySelector('rect, circle, polygon, path');
  if (shape) {
    // 一度に複数の属性を設定（リフロー削減）
    shape.setAttribute('fill', colorDef.bg);
    shape.setAttribute('stroke', colorDef.border);
    shape.setAttribute('stroke-width', '2');
  }

  const text = group.querySelector('text');
  if (text) {
    text.setAttribute('fill', colorDef.text);
  }

  group.setAttribute('data-status', statusKey);
}
```

**目標パフォーマンス**:
- 100要素: < 100ms
- 500要素: < 500ms
- 1000要素: < 1秒

**実装優先度**: 🟡 中（Phase 4-1で基本実装、Phase 5でパフォーマンステスト）

---

### 4. データ整合性: 削除要素のステータス残存

**問題**:
要素が削除された後もelementStatusesにステータスが残る

**リスクシナリオ**:
```javascript
// 1. ノードにステータス設定
elementStatuses["node_001"] = "completed";

// 2. 新しいSVGをアップロード（node_001が存在しない）

// 3. ステータスが孤立（orphaned status）
elementStatuses["node_001"] = "completed";  // ❌ node_001は存在しない
```

**対策**:
```javascript
/**
 * 孤立ステータスのクリーンアップ
 */
function cleanupOrphanedStatuses() {
  const validElementIds = new Set();

  // 正規ノードのID収集
  svg.querySelectorAll('[data-id]').forEach(el => {
    validElementIds.add(el.getAttribute('data-id'));
  });

  // サブグラフのID収集
  svg.querySelectorAll('g.cluster[data-subgraph-name]').forEach(el => {
    validElementIds.add(el.getAttribute('data-subgraph-name'));
  });

  // メモノードのID収集
  for (const memoId of Object.keys(manualNodes)) {
    validElementIds.add(memoId);
  }

  // 孤立ステータスの削除
  let orphanedCount = 0;
  for (const elementId of Object.keys(elementStatuses)) {
    if (!validElementIds.has(elementId)) {
      delete elementStatuses[elementId];
      orphanedCount++;
      console.log(`[Cleanup] 孤立ステータスを削除: ${elementId}`);
    }
  }

  if (orphanedCount > 0) {
    console.log(`[Cleanup] ${orphanedCount}個の孤立ステータスを削除しました`);
  }
}

/**
 * SVG読込時に自動クリーンアップ
 */
async function loadSVGFile(file) {
  // ... SVG読込処理 ...

  // メタデータ復元後にクリーンアップ
  if (extractedMetadata) {
    restoreMemoData(extractedMetadata);
    cleanupOrphanedStatuses();  // ★自動クリーンアップ
    cleanupInvalidReferences();  // 参照のクリーンアップも実行
  }
}
```

**実装優先度**: 🟡 中（Phase 4-2で実装）

---

### 5. UI/UX: 色覚アクセシビリティ

**問題**:
色のみでステータスを示すと、色覚障害のユーザーが識別困難

**リスクシナリオ**:
- 赤緑色覚異常（約5%の男性）: 完了（緑）と進行中（黄）の区別が困難
- 全色覚異常: すべてのステータスが識別不可

**対策**:
```javascript
/**
 * ステータスアイコンバッジの追加（Phase 4-3）
 */
const STATUS_ICONS = {
  'not-started': '○',       // 空丸
  'in-progress': '◐',       // 半丸
  'completed': '●',         // 塗り丸
  'on-hold': '■',          // 四角
  'postponed': '▲',        // 三角
  'cancelled': '✕',        // バツ
  'unknown': '?'           // クエスチョン
};

/**
 * ステータスバッジをSVGに追加
 */
function addStatusBadge(elementGroup, statusKey) {
  const colorDef = STATUS_COLORS[statusKey];
  const icon = STATUS_ICONS[statusKey];

  // 既存のバッジを削除
  const oldBadge = elementGroup.querySelector('.status-badge');
  if (oldBadge) oldBadge.remove();

  // バッジグループを作成
  const badge = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  badge.classList.add('status-badge');

  // バッジの位置（要素の右上）
  const bbox = elementGroup.getBBox();
  const badgeX = bbox.x + bbox.width - 10;
  const badgeY = bbox.y + 5;

  // 背景円
  const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
  circle.setAttribute('cx', badgeX);
  circle.setAttribute('cy', badgeY);
  circle.setAttribute('r', '8');
  circle.setAttribute('fill', colorDef.bg);
  circle.setAttribute('stroke', colorDef.border);
  circle.setAttribute('stroke-width', '1');

  // アイコンテキスト
  const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
  text.setAttribute('x', badgeX);
  text.setAttribute('y', badgeY + 4);
  text.setAttribute('text-anchor', 'middle');
  text.setAttribute('font-size', '10');
  text.setAttribute('fill', colorDef.text);
  text.textContent = icon;

  badge.appendChild(circle);
  badge.appendChild(text);
  elementGroup.appendChild(badge);

  // ツールチップ
  const title = document.createElementNS('http://www.w3.org/2000/svg', 'title');
  title.textContent = `ステータス: ${getStatusLabel(statusKey)}`;
  badge.appendChild(title);
}

/**
 * ステータス表示名の取得
 */
function getStatusLabel(statusKey) {
  const labels = {
    'not-started': '未着手',
    'in-progress': '進行中',
    'completed': '完了',
    'on-hold': '保留',
    'postponed': '延期',
    'cancelled': '中止',
    'unknown': '不明'
  };
  return labels[statusKey] || statusKey;
}
```

**CSS追加**:
```css
/* ステータスバッジのホバーエフェクト */
.status-badge {
  cursor: help;
  transition: transform 0.2s;
}

.status-badge:hover {
  transform: scale(1.2);
}
```

**実装優先度**: 🟡 中（Phase 4-3で実装）

---

### 6. UI/UX: タスクステータスとの混同防止

**問題**:
task-manager.htmlのタスクステータスと、flowchart-editor.htmlのフローチャートステータスを混同するリスク

**ユーザーの誤解**:
```
ユーザーの期待（誤）:
- task-manager.htmlでタスクを「完了」にすると、フローチャートのノードも自動的に「完了」色になる

実際の仕様（正）:
- フローチャートステータスとタスクステータスは完全に独立
- データ連携なし
```

**対策**:

1. **UI上での明示**:
```html
<!-- 右クリックメニューにヘルプテキスト -->
<div id="contextMenu">
  <div class="menu-item" onclick="openMemoDialog()">📝 メモを編集</div>
  <div class="menu-item" onclick="openLabelDialog()">🏷️ ラベルを編集</div>
  <div class="menu-separator"></div>
  <div class="menu-section-header">
    フローチャートステータス
    <span class="help-icon" title="タスクステータスとは独立です">ⓘ</span>
  </div>
  <div class="menu-item submenu" onclick="openStatusMenu()">
    🎨 ステータス設定 ▶
  </div>
</div>
```

2. **ドキュメントでの明記**:
README.md、仕様書、ヘルプ画面に以下を記載:
> **重要**: フローチャートのステータス色分けは、task-manager.htmlのタスクステータスとは**完全に独立**しています。
> - タスクステータス: タスクの進捗管理（task-manager.html内で管理）
> - フローチャートステータス: フローチャート要素の状態管理（flowchart-editor.html内で管理）
>
> 両者の間でデータ同期は行われません。

3. **命名の工夫**:
```javascript
// 変数名を明示的に
let elementStatuses = {};  // "フローチャート要素"のステータス（タスクではない）

// コメントで強調
/**
 * フローチャート要素のステータス管理
 * ⚠️ 注意: task-manager.htmlのタスクステータスとは独立
 *
 * @type {Object<string, string>}
 * @example { "flowchart-A": "completed", "memo_node_1": "in-progress" }
 */
let elementStatuses = {};
```

**実装優先度**: 🟡 中（Phase 4-1で命名・コメント、Phase 4-3でUI改善）

---

### 7. データ整合性: ステータスのv1.0 → v2.0移行

**問題**:
v1.0メタデータにはelementStatusesが存在しないため、移行時の初期化が必要

**対策**:
Phase 1で実装済みの`migrateMetadata()`関数が対応（07_integrated-roadmap.md Phase 1-3参照）

```javascript
function migrateMetadata(data) {
  // ... 既存の処理 ...

  // ★Phase 4: ステータス管理の初期化
  if (!migrated.elementStatuses) {
    migrated.elementStatuses = {};
    console.log('[Migration] elementStatuses を空オブジェクトで初期化');
  }

  // ... 後続処理 ...
}
```

**検証項目**:
- [ ] v1.0 → v2.0移行時にelementStatusesが空オブジェクトで初期化される
- [ ] 既存のv1.0データ（elements, manualNodes等）が損失しない
- [ ] マイグレーション後のSVG再保存でv2.0形式になる

**実装優先度**: 🔴 高（Phase 1で実装済み、Phase 4-2で検証）

---

### 8. 統合テスト: ステータス機能の全体検証

**Phase 5で実施すべきテストシナリオ**:

#### シナリオA: 基本的なステータス設定と復元
```
1. ノードAを右クリック → 「ステータス設定」→ 「完了」
2. ノードAの背景色が薄グリーンになることを確認
3. ステータスバッジ「●」が表示されることを確認
4. メタデータをエクスポート
5. SVGを再読み込み
6. ノードAのステータス色とバッジが復元されることを確認
```

#### シナリオB: 異なる要素タイプへのステータス適用
```
1. 正規ノードAに「進行中」を設定
2. サブグラフBに「保留」を設定
3. メモノードCに「延期」を設定
4. それぞれ異なる色で表示されることを確認
5. SVG再読み込み後も正しく復元されることを確認
```

#### シナリオC: v1.0 → v2.0マイグレーション（ステータス関連）
```
1. v1.0形式のメタデータ付きSVGを用意
2. flowchart-editor.htmlで開く
3. マイグレーションログで「elementStatuses を空オブジェクトで初期化」を確認
4. ノードにステータスを設定
5. エクスポートしたメタデータがv2.0形式であることを確認
6. elementStatusesフィールドが存在し、設定したステータスが含まれることを確認
```

#### シナリオD: 大規模フローチャート（パフォーマンス）
```
1. 100ノードのフローチャートを用意
2. すべてのノードに異なるステータスを設定（一括または個別）
3. applyAllStatuses()の実行時間を測定（< 100ms目標）
4. SVGエクスポート時間を測定（< 1秒目標）
5. SVG再読み込み+ステータス復元時間を測定（< 2秒目標）
```

#### シナリオE: 色覚アクセシビリティ
```
1. ノードA: 完了（緑）、ノードB: 進行中（黄）を設定
2. ステータスバッジが表示され、アイコンで識別可能であることを確認
3. グレースケール表示に切り替え（開発者ツール）
4. ステータスバッジのアイコンで識別できることを確認
```

**実装優先度**: 🔴 高（Phase 5で必須）

---

## 📋 Phase 4 まとめ：優先順位別実装リスト

### 🔴 高優先度（Phase 4-1/4-2で必須）
- [ ] ステータスキー検証（validateElementStatuses）
- [ ] Prototype Pollution対策（__proto__等のブロック）
- [ ] v1.0 → v2.0マイグレーション検証
- [ ] Phase 5統合テスト（全シナリオ）

### 🟡 中優先度（Phase 4-2/4-3で実装）
- [ ] 一括ステータス適用の最適化（applyAllStatuses）
- [ ] 孤立ステータスのクリーンアップ
- [ ] ステータスアイコンバッジ（色覚アクセシビリティ）
- [ ] タスクステータスとの混同防止（UI改善）

### 🟢 低優先度（Phase 4-3以降で検討）
- [ ] CSS色値インジェクション対策（カスタム色実装時のみ）
- [ ] ステータスフィルタリング機能の拡張
- [ ] 一括ステータス設定UI

---

**作成日**: 2026-03-22
**バージョン**: 1.1
**更新日**: 2026-03-22
**更新内容**: Phase 4（ステータス色分け機能）固有の懸念事項を追加
**レビュー推奨**: 外部AI有識者によるセキュリティ・パフォーマンスレビュー
