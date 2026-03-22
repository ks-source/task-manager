# 統合実装ロードマップ - v2.0メタデータ

---
**feature**: v2.0メタデータ統合実装
**version**: 1.0
**status**: draft
**created**: 2026-03-22
**author**: Claude Code Session 04
---

## 📋 概要

本ドキュメントは、flowchart-editor.html の v2.0 メタデータスキーマに統合される以下3機能の実装ロードマップを定義する：

1. **参照関係管理機能** - 要素間の参照リンク管理
2. **タスク連携スナップショット** - task-manager.html との連携データ
3. **ステータス色分け機能** (Phase 4) - 7種類のステータス視覚化

## 🎯 実装目標

### 機能目標
- ✅ v2.0 メタデータスキーマの完全実装
- ✅ v1.0 から v2.0 への自動マイグレーション
- ✅ 3機能の独立性確保（相互依存なし）
- ✅ セキュリティ対策の完全実装（XSS、Prototype Pollution防止）
- ✅ データ整合性の保証

### 品質目標
- **データ損失ゼロ**: マイグレーション時の既存データ保全
- **セキュリティ**: 全入力値の適切なサニタイゼーション
- **パフォーマンス**: 1000要素規模でも快適な動作
- **保守性**: 各機能の独立実装（疎結合アーキテクチャ）

## 📅 実装フェーズ

```
Phase 0: 準備・設計確認              [完了]  0.5-1h
  └─ スキーマ設計確定
  └─ ドキュメント整備

Phase 1: 基盤機能実装                [未着手] 2-3h
  ├─ グローバル変数追加
  ├─ buildMemoData() v2.0拡張
  └─ マイグレーション実装

Phase 2: 参照関係管理                [未着手] 3-4h
  ├─ UI実装（参照追加・削除）
  ├─ データ永続化
  └─ 視覚化（矢印表示）

Phase 3: タスク連携                  [未着手] 2-3h
  ├─ buildTaskLinks()実装
  ├─ スナップショットUI
  └─ データ復元処理

Phase 4: ステータス色分け            [未着手] 6-9h
  ├─ 4-1: 基本実装（3-4h）
  ├─ 4-2: 拡張機能（2-3h）
  └─ 4-3: UX改善（1-2h）

Phase 5: 統合テスト・最適化          [未着手] 1-2h
  ├─ 全機能の統合動作確認
  ├─ パフォーマンステスト
  └─ ドキュメント最終化
```

**総見積もり**: **14-20時間** (2-3日間の実装作業)

## 🔧 Phase 0: 準備・設計確認 [完了] (0.5-1時間)

### 完了済みタスク

#### ✅ タスク0.1: スキーマ設計の確定
**成果物**:
- `/docs/archive/session/04_node-reference/04_data-structure.md` (v2.0完全仕様)
- v2.0メタデータ構造の完全定義
- 11個の最上位フィールドの仕様確定

#### ✅ タスク0.2: ドキュメント体系整備
**成果物**:
- `02_proposed-design.md` v1.2更新（Phase 4統合）
- `05_implementation-plan.md` v1.2更新（Phase 4タスク追加）
- `07_integrated-roadmap.md` 新規作成（本ドキュメント）
- `06_technical-concerns.md` 更新予定（次タスク）

#### ✅ タスク0.3: 他セッションとの調整
**結果**:
- Phase 4ステータス色分け機能の仕様を本セッションに統合
- 競合回避のため、スキーマ構造化は本セッションで一元実施
- 実装は別途判断（ドキュメント整備のみ完了）

## 🏗️ Phase 1: 基盤機能実装 (2-3時間)

### Phase 1-1: グローバル変数とデータ構造 (30分)

```javascript
// ★新規追加: v2.0機能用グローバル変数
let elementReferences = {};   // { elementId: [refId1, refId2, ...] }
let elementStatuses = {};     // { elementId: statusKey }

// ★既存変数（参考）
let elementMemos = {};
let elementLabels = {};
let originalLabels = {};
let manualNodes = {};
let manualEdges = {};

// ★タスク連携用（Phase 3で実装）
function buildTaskLinks() {
  // 実装詳細はPhase 3で定義
  return {
    nodes: {},
    _snapshotAt: null
  };
}
```

**検証項目**:
- [ ] すべてのグローバル変数が適切に初期化される
- [ ] 既存変数との命名衝突がない
- [ ] データ構造が04_data-structure.mdの仕様に準拠

### Phase 1-2: buildMemoData() v2.0拡張 (1-1.5時間)

**実装対象**: flowchart-editor.html の `buildMemoData()` 関数

```javascript
function buildMemoData() {
  const elements = {};

  // 既存のelements構築ロジック（変更なし）
  const allElementIds = new Set([
    ...Object.keys(elementMemos),
    ...Object.keys(elementLabels),
    ...Object.keys(originalLabels)
  ]);

  for (const elementId of allElementIds) {
    const element = document.querySelector(`[data-id="${CSS.escape(elementId)}"]`);
    const type = element?.getAttribute('data-et') || 'unknown';
    const displayText = getElementDisplayText(element);
    const memo = elementMemos[elementId] || '';
    const customLabel = elementLabels[elementId] || '';

    elements[elementId] = {
      type: type,
      displayText: displayText,
      memo: memo,
      customLabel: customLabel,
      originalLabel: originalLabels[elementId] || ''
    };
  }

  // ★v2.0拡張部分
  return {
    version: "2.0",
    schemaVersion: "2.0",
    exportedAt: new Date().toISOString(),
    svgFile: currentSvgFileName || 'unknown.svg',

    // 既存フィールド（v1.0互換）
    elements: elements,
    originalLabels: originalLabels,
    manualNodes: manualNodes,
    manualEdges: manualEdges,

    // ★v2.0新機能
    elementStatuses: elementStatuses,        // Phase 4
    references: elementReferences,           // Phase 2
    taskFlowchartLinks: buildTaskLinks()     // Phase 3
  };
}
```

**検証項目**:
- [ ] v2.0形式でのJSON出力が正しい
- [ ] 既存のv1.0フィールドが保持される
- [ ] 新規フィールドが空でも正常に動作
- [ ] schemaVersionが正しく設定される

### Phase 1-3: マイグレーション実装 (1時間)

**実装対象**: SVG読み込み時の自動マイグレーション

```javascript
/**
 * v1.0 → v2.0 自動マイグレーション
 */
function migrateMetadata(data) {
  // v2.0の場合はそのまま返す
  if (data.version === "2.0" || data.schemaVersion === "2.0") {
    console.log('[Migration] 既にv2.0形式です');
    return data;
  }

  // Deep clone to avoid mutation
  let migrated = structuredClone(data);

  console.log('[Migration] v1.0 → v2.0 マイグレーション開始');

  // バージョン更新
  migrated.version = "2.0";
  migrated.schemaVersion = "2.0";

  // ★Phase 4: ステータス管理の初期化
  if (!migrated.elementStatuses) {
    migrated.elementStatuses = {};
    console.log('[Migration] elementStatuses を空オブジェクトで初期化');
  }

  // ★Phase 2: 参照関係管理の初期化
  if (!migrated.references) {
    migrated.references = {};
    console.log('[Migration] references を空オブジェクトで初期化');
  }

  // ★Phase 3: タスク連携の初期化
  if (!migrated.taskFlowchartLinks) {
    migrated.taskFlowchartLinks = {
      nodes: {},
      _snapshotAt: null
    };
    console.log('[Migration] taskFlowchartLinks を空で初期化');
  }

  // exportedAtが無い場合は現在時刻を設定
  if (!migrated.exportedAt) {
    migrated.exportedAt = new Date().toISOString();
  }

  console.log('[Migration] ✅ v2.0へのマイグレーション完了');

  return migrated;
}

/**
 * SVG読み込み時の呼び出し
 */
function restoreMemoData(data) {
  if (!data) {
    console.log('[Restore] データがnullまたはundefinedです');
    return;
  }

  // ★マイグレーション実行
  const migratedData = migrateMetadata(data);

  // 既存のデータ復元処理
  elementMemos = migratedData.elements ?
    Object.fromEntries(
      Object.entries(migratedData.elements)
        .filter(([_, v]) => v.memo)
        .map(([k, v]) => [k, v.memo])
    ) : {};

  elementLabels = migratedData.elements ?
    Object.fromEntries(
      Object.entries(migratedData.elements)
        .filter(([_, v]) => v.customLabel)
        .map(([k, v]) => [k, v.customLabel])
    ) : {};

  originalLabels = migratedData.originalLabels || {};
  manualNodes = migratedData.manualNodes || {};
  manualEdges = migratedData.manualEdges || {};

  // ★v2.0新機能の復元
  elementStatuses = migratedData.elementStatuses || {};   // Phase 4
  elementReferences = migratedData.references || {};      // Phase 2

  // ★Phase 3: タスク連携データの復元（スナップショット表示のみ）
  if (migratedData.taskFlowchartLinks) {
    // UI更新処理（Phase 3で実装）
    updateTaskLinksUI(migratedData.taskFlowchartLinks);
  }

  console.log('[Restore] v2.0データの復元完了');
}
```

**検証項目**:
- [ ] v1.0データが正常にv2.0に変換される
- [ ] v2.0データはそのまま読み込まれる
- [ ] 既存データが損失しない
- [ ] マイグレーションログが適切に出力される

## 🔗 Phase 2: 参照関係管理 (3-4時間)

### Phase 2-1: 参照追加UI (1.5-2時間)

**UI要素**:
```html
<!-- 右クリックメニューに追加 -->
<div id="contextMenu">
  <!-- 既存メニュー項目 -->
  <div class="menu-item" onclick="openMemoDialog()">📝 メモを編集</div>
  <div class="menu-item" onclick="openLabelDialog()">🏷️ ラベルを編集</div>

  <!-- ★新規: 参照管理 -->
  <div class="menu-separator"></div>
  <div class="menu-item" onclick="openReferenceDialog()">🔗 参照を追加</div>
  <div class="menu-item" onclick="showReferences()">📋 参照一覧を表示</div>
</div>

<!-- 参照追加ダイアログ -->
<div id="referenceDialog" class="dialog">
  <h3>参照先を追加</h3>
  <p>このノード「<span id="refSourceName"></span>」が参照する要素を選択:</p>
  <select id="referenceTargetSelect" multiple size="10">
    <!-- 動的に要素リストを生成 -->
  </select>
  <button onclick="addReferences()">追加</button>
  <button onclick="closeReferenceDialog()">キャンセル</button>
</div>
```

**実装関数**:
```javascript
/**
 * 参照追加ダイアログを開く
 */
function openReferenceDialog() {
  const elementId = currentContextElementId;
  if (!elementId) return;

  document.getElementById('refSourceName').textContent =
    getElementDisplayText(document.querySelector(`[data-id="${CSS.escape(elementId)}"]`));

  // すべての要素をセレクトボックスに追加（自分自身を除く）
  const select = document.getElementById('referenceTargetSelect');
  select.innerHTML = '';

  const allElements = svg.querySelectorAll('[data-id]');
  allElements.forEach(el => {
    const id = el.getAttribute('data-id');
    if (id !== elementId) {
      const option = document.createElement('option');
      option.value = id;
      option.textContent = `${id} - ${getElementDisplayText(el)}`;

      // 既存の参照をハイライト
      if (elementReferences[elementId]?.includes(id)) {
        option.selected = true;
        option.style.fontWeight = 'bold';
      }

      select.appendChild(option);
    }
  });

  document.getElementById('referenceDialog').style.display = 'block';
}

/**
 * 参照を追加
 */
function addReferences() {
  const elementId = currentContextElementId;
  const select = document.getElementById('referenceTargetSelect');
  const selectedIds = Array.from(select.selectedOptions).map(opt => opt.value);

  // 参照を保存
  elementReferences[elementId] = selectedIds;

  console.log(`[参照管理] ${elementId} → [${selectedIds.join(', ')}]`);

  // UI更新
  renderReferenceArrows();

  closeReferenceDialog();
}
```

**検証項目**:
- [ ] 要素を右クリックして参照追加メニューが表示される
- [ ] ダイアログで参照先を複数選択できる
- [ ] 既存の参照が太字で表示される
- [ ] 参照が正しくグローバル変数に保存される

### Phase 2-2: 参照の視覚化 (1-1.5時間)

**実装**: SVG上に参照を示す矢印を描画

```javascript
/**
 * 参照矢印をSVG上に描画
 */
function renderReferenceArrows() {
  // 既存の参照矢印を削除
  svg.querySelectorAll('[data-reference-arrow]').forEach(el => el.remove());

  // 参照矢印用のグループを作成
  let refGroup = svg.querySelector('#referenceArrows');
  if (!refGroup) {
    refGroup = document.createElementNS('http://www.w3.org/2000/svg', 'g');
    refGroup.id = 'referenceArrows';
    refGroup.setAttribute('data-reference-arrow', 'group');
    svg.insertBefore(refGroup, svg.firstChild); // 最背面に配置
  }

  // すべての参照を描画
  for (const [sourceId, targetIds] of Object.entries(elementReferences)) {
    const sourceEl = svg.querySelector(`[data-id="${CSS.escape(sourceId)}"]`);
    if (!sourceEl) continue;

    const sourceBBox = sourceEl.getBBox();
    const sourceX = sourceBBox.x + sourceBBox.width / 2;
    const sourceY = sourceBBox.y + sourceBBox.height / 2;

    for (const targetId of targetIds) {
      const targetEl = svg.querySelector(`[data-id="${CSS.escape(targetId)}"]`);
      if (!targetEl) continue;

      const targetBBox = targetEl.getBBox();
      const targetX = targetBBox.x + targetBBox.width / 2;
      const targetY = targetBBox.y + targetBBox.height / 2;

      // 破線矢印を描画
      const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
      line.setAttribute('x1', sourceX);
      line.setAttribute('y1', sourceY);
      line.setAttribute('x2', targetX);
      line.setAttribute('y2', targetY);
      line.setAttribute('stroke', '#888');
      line.setAttribute('stroke-width', '1.5');
      line.setAttribute('stroke-dasharray', '5,3');
      line.setAttribute('marker-end', 'url(#referenceArrowhead)');
      line.setAttribute('data-reference-arrow', 'true');
      line.setAttribute('data-ref-source', sourceId);
      line.setAttribute('data-ref-target', targetId);

      refGroup.appendChild(line);
    }
  }
}

/**
 * SVG初期化時に矢印マーカーを定義
 */
function initializeReferenceMarkers() {
  let defs = svg.querySelector('defs');
  if (!defs) {
    defs = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
    svg.insertBefore(defs, svg.firstChild);
  }

  const marker = document.createElementNS('http://www.w3.org/2000/svg', 'marker');
  marker.id = 'referenceArrowhead';
  marker.setAttribute('markerWidth', '10');
  marker.setAttribute('markerHeight', '10');
  marker.setAttribute('refX', '9');
  marker.setAttribute('refY', '3');
  marker.setAttribute('orient', 'auto');

  const path = document.createElementNS('http://www.w3.org/2000/svg', 'path');
  path.setAttribute('d', 'M0,0 L0,6 L9,3 z');
  path.setAttribute('fill', '#888');

  marker.appendChild(path);
  defs.appendChild(marker);
}
```

**検証項目**:
- [ ] 参照矢印が破線で描画される
- [ ] 矢印の向きが正しい（source → target）
- [ ] 要素を移動しても矢印が追従しない（仕様通り）
- [ ] 矢印が他の要素の背面に描画される

### Phase 2-3: データ永続化とセキュリティ (30分)

**実装**: 参照データの保存と読み込み

```javascript
/**
 * 参照データの検証（Prototype Pollution対策）
 */
function validateReferences(refs) {
  if (typeof refs !== 'object' || refs === null) {
    return {};
  }

  const validated = {};

  for (const [key, value] of Object.entries(refs)) {
    // プロトタイプ汚染対策
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
      console.warn(`[Security] プロトタイプ汚染の試みを検出: ${key}`);
      continue;
    }

    // 値が配列であることを確認
    if (!Array.isArray(value)) {
      console.warn(`[Validation] references[${key}]が配列ではありません`);
      continue;
    }

    // 配列の各要素が文字列であることを確認
    const validatedArray = value.filter(item => typeof item === 'string');
    if (validatedArray.length !== value.length) {
      console.warn(`[Validation] references[${key}]に非文字列要素が含まれています`);
    }

    validated[key] = validatedArray;
  }

  return validated;
}

/**
 * restoreMemoData()内での参照データ復元（修正版）
 */
function restoreMemoData(data) {
  // ... 既存の復元処理 ...

  // ★参照データの復元（検証付き）
  const rawReferences = migratedData.references || {};
  elementReferences = validateReferences(rawReferences);

  console.log(`[Restore] ${Object.keys(elementReferences).length}個の参照関係を復元`);

  // 参照矢印を描画
  renderReferenceArrows();
}
```

**検証項目**:
- [ ] 参照データがJSON保存される
- [ ] SVG読み込み時に参照が復元される
- [ ] Prototype Pollution攻撃が防がれる
- [ ] 不正な形式のデータが検証でフィルタされる

## 📊 Phase 3: タスク連携スナップショット (2-3時間)

### Phase 3-1: buildTaskLinks()実装 (1時間)

**実装**: task-manager.htmlのLocalStorageからデータを取得してスナップショット生成

```javascript
/**
 * タスク連携データをスナップショット形式で構築
 */
function buildTaskLinks() {
  try {
    // task-manager.html の LocalStorage から taskData を取得
    const taskDataStr = localStorage.getItem('task_manager_v2_project_data');
    if (!taskDataStr) {
      console.log('[TaskLinks] task-manager.htmlのデータが見つかりません');
      return {
        nodes: {},
        _snapshotAt: null
      };
    }

    const taskData = JSON.parse(taskDataStr);
    const tasks = taskData.tasks || [];

    // flowchartMetadata から Mermaid ID連携情報を取得
    const flowchartMeta = taskData.flowchartMetadata?.nodes || {};

    const nodes = {};

    for (const [wbsNo, linkData] of Object.entries(flowchartMeta)) {
      // タスク本体を検索
      const task = tasks.find(t => t.wbsNo === wbsNo);
      if (!task) {
        console.warn(`[TaskLinks] WBS ${wbsNo} のタスクが見つかりません`);
        continue;
      }

      // スナップショットデータ作成
      nodes[wbsNo] = {
        taskId: task.id,
        wbsNo: wbsNo,
        taskName: task.name || '',
        mermaidIds: linkData.mermaidIds || [],
        _snapshotAt: new Date().toISOString()
      };
    }

    console.log(`[TaskLinks] ${Object.keys(nodes).length}個のタスク連携をスナップショット`);

    return {
      nodes: nodes,
      _snapshotAt: new Date().toISOString()
    };

  } catch (error) {
    console.error('[TaskLinks] エラー:', error);
    return {
      nodes: {},
      _snapshotAt: null
    };
  }
}
```

**検証項目**:
- [ ] task-manager.htmlのLocalStorageから正しくデータ取得
- [ ] mermaidIdsが正しく抽出される
- [ ] エラー時にも安全に空データを返す
- [ ] スナップショット時刻が記録される

### Phase 3-2: スナップショットUI (1-1.5時間)

**UI要素**:
```html
<!-- タスク連携パネル（折りたたみ式） -->
<div id="taskLinksPanel" class="side-panel collapsed">
  <h3 onclick="toggleTaskLinksPanel()">
    📋 タスク連携スナップショット
    <span class="expand-icon">▶</span>
  </h3>
  <div class="panel-content">
    <div id="taskLinksStatus" class="status-line">
      スナップショット: <span id="snapshotTime">未取得</span>
    </div>
    <div id="taskLinksList" class="links-list">
      <!-- 動的生成 -->
    </div>
    <button onclick="refreshTaskLinks()">🔄 再スナップショット</button>
  </div>
</div>
```

**実装関数**:
```javascript
/**
 * タスク連携UIを更新
 */
function updateTaskLinksUI(taskLinks) {
  if (!taskLinks || !taskLinks.nodes) {
    document.getElementById('snapshotTime').textContent = '未取得';
    document.getElementById('taskLinksList').innerHTML = '<p>データがありません</p>';
    return;
  }

  // スナップショット時刻を表示
  const snapshotTime = taskLinks._snapshotAt ?
    new Date(taskLinks._snapshotAt).toLocaleString('ja-JP') : '不明';
  document.getElementById('snapshotTime').textContent = snapshotTime;

  // タスクリンク一覧を生成
  const listDiv = document.getElementById('taskLinksList');
  listDiv.innerHTML = '';

  const entries = Object.values(taskLinks.nodes);

  if (entries.length === 0) {
    listDiv.innerHTML = '<p>連携タスクがありません</p>';
    return;
  }

  entries.forEach(link => {
    const item = document.createElement('div');
    item.className = 'task-link-item';
    item.innerHTML = `
      <div class="task-wbs">${link.wbsNo}</div>
      <div class="task-name">${link.taskName}</div>
      <div class="mermaid-ids">
        ${link.mermaidIds.map(id => `<span class="mermaid-id">${id}</span>`).join(' ')}
      </div>
    `;

    // クリックでフローチャート要素をハイライト
    item.onclick = () => highlightLinkedNodes(link.mermaidIds);

    listDiv.appendChild(item);
  });
}

/**
 * 連携ノードをハイライト
 */
function highlightLinkedNodes(mermaidIds) {
  // 既存のハイライトを削除
  svg.querySelectorAll('.highlighted').forEach(el =>
    el.classList.remove('highlighted')
  );

  // 対象ノードをハイライト
  mermaidIds.forEach(id => {
    const node = svg.querySelector(`[data-id="${CSS.escape(id)}"]`);
    if (node) {
      node.classList.add('highlighted');

      // 3秒後にハイライト解除
      setTimeout(() => {
        node.classList.remove('highlighted');
      }, 3000);
    }
  });
}
```

**検証項目**:
- [ ] スナップショット時刻が正しく表示される
- [ ] タスク一覧が見やすく表示される
- [ ] タスクをクリックすると対応ノードがハイライトされる
- [ ] 再スナップショットボタンが正常動作する

### Phase 3-3: セキュリティとエラーハンドリング (30分)

**実装**: タスクデータの検証とXSS対策

```javascript
/**
 * タスク名のサニタイゼーション
 */
function sanitizeTaskName(name) {
  if (typeof name !== 'string') {
    return '';
  }

  // HTML特殊文字をエスケープ
  const div = document.createElement('div');
  div.textContent = name;
  return div.innerHTML;
}

/**
 * buildTaskLinks()の安全版（修正）
 */
function buildTaskLinks() {
  try {
    const taskDataStr = localStorage.getItem('task_manager_v2_project_data');
    if (!taskDataStr) {
      return { nodes: {}, _snapshotAt: null };
    }

    const taskData = JSON.parse(taskDataStr);
    const tasks = taskData.tasks || [];
    const flowchartMeta = taskData.flowchartMetadata?.nodes || {};

    const nodes = {};

    for (const [wbsNo, linkData] of Object.entries(flowchartMeta)) {
      // Prototype Pollution対策
      if (wbsNo === '__proto__' || wbsNo === 'constructor' || wbsNo === 'prototype') {
        console.warn(`[Security] プロトタイプ汚染の試みを検出: ${wbsNo}`);
        continue;
      }

      const task = tasks.find(t => t.wbsNo === wbsNo);
      if (!task) continue;

      // ★XSS対策: taskNameをサニタイズ
      const sanitizedName = sanitizeTaskName(task.name);

      // ★配列検証: mermaidIdsが配列であることを確認
      const mermaidIds = Array.isArray(linkData.mermaidIds) ?
        linkData.mermaidIds.filter(id => typeof id === 'string') : [];

      nodes[wbsNo] = {
        taskId: task.id,
        wbsNo: wbsNo,
        taskName: sanitizedName,
        mermaidIds: mermaidIds,
        _snapshotAt: new Date().toISOString()
      };
    }

    return {
      nodes: nodes,
      _snapshotAt: new Date().toISOString()
    };

  } catch (error) {
    console.error('[TaskLinks] エラー:', error);
    return { nodes: {}, _snapshotAt: null };
  }
}
```

**検証項目**:
- [ ] XSS攻撃が防がれる（taskName経由）
- [ ] Prototype Pollution攻撃が防がれる
- [ ] LocalStorage読み込みエラーが適切にハンドリングされる
- [ ] 不正なJSON形式でもクラッシュしない

## 🎨 Phase 4: ステータス色分け機能 (6-9時間)

**詳細**: `05_implementation-plan.md` の Phase 4セクション参照

### Phase 4-1: 基本実装 (3-4時間)
- ステータス色定義（STATUS_COLORS）
- buildMemoData()拡張（elementStatuses追加）
- 右クリックメニューUI（ステータス選択）
- 正規ノードへのステータス適用

### Phase 4-2: 拡張機能 (2-3時間)
- サブグラフへのステータス適用
- メモノードへのステータス適用
- ステータスクリア機能
- マイグレーション処理統合

### Phase 4-3: UX改善 (1-2時間)
- ステータスアイコンバッジ
- フィルタリング機能拡張
- 一括ステータス設定（オプション）

**実装詳細**: `05_implementation-plan.md` Lines 879-1150 を参照

## 🧪 Phase 5: 統合テスト・最適化 (1-2時間)

### Phase 5-1: 統合動作確認 (45分)

**テストシナリオ**:

#### シナリオ1: v1.0 → v2.0マイグレーション
```
1. 既存のv1.0メタデータ付きSVGを読み込み
2. マイグレーションログを確認
3. すべてのフィールドが正しく初期化されることを確認
4. データ損失がないことを確認
```

#### シナリオ2: 参照関係の作成と保存
```
1. ノードAを右クリック → 「参照を追加」
2. ノードB, Cを選択して追加
3. 参照矢印が表示されることを確認
4. メタデータをエクスポートして保存
5. SVGを再読み込みして参照が復元されることを確認
```

#### シナリオ3: タスク連携スナップショット
```
1. task-manager.htmlでタスクとフローチャートノードを連携
2. flowchart-editor.htmlでSVGを開く
3. 「再スナップショット」ボタンをクリック
4. タスク連携パネルに正しい情報が表示されることを確認
5. タスクをクリックしてノードハイライトが動作することを確認
```

#### シナリオ4: ステータス色分け
```
1. ノードAを右クリック → 「ステータス設定」→ 「完了」
2. ノードの背景色が薄グリーンになることを確認
3. メタデータをエクスポート
4. SVGを再読み込みしてステータス色が復元されることを確認
```

#### シナリオ5: 統合操作
```
1. ノードAにメモ追加
2. ノードAのラベル変更
3. ノードAのステータスを「進行中」に設定
4. ノードAからノードBへの参照を追加
5. タスクWBS1.1.0とノードAを連携（task-manager.html側）
6. flowchart-editor.htmlでスナップショット
7. すべての情報がメタデータに保存されることを確認
```

### Phase 5-2: パフォーマンステスト (30分)

**テスト環境**:
- 要素数: 100ノード、150エッジ
- 参照数: 50個
- ステータス設定数: 80個

**測定項目**:
- [ ] SVG読み込み時間 < 2秒
- [ ] メタデータエクスポート時間 < 1秒
- [ ] 参照矢印描画時間 < 500ms
- [ ] ステータス一括適用時間 < 1秒

### Phase 5-3: ドキュメント最終化 (15分)

**更新対象**:
- [ ] README.md - 実装完了ステータス更新
- [ ] 06_technical-concerns.md - 既知の問題・制約事項記載
- [ ] 本ドキュメント (07_integrated-roadmap.md) - 実装完了マーク

## 🌳 Git ブランチ戦略

### ブランチ構成

```
main (安定版)
  └─ feature/v2.0-metadata (統合ブランチ)
       ├─ feature/phase-1-foundation (Phase 1)
       ├─ feature/phase-2-references (Phase 2)
       ├─ feature/phase-3-tasklinks (Phase 3)
       └─ feature/phase-4-status (Phase 4)
```

### マージ戦略

1. **Phase 1完了時**:
   ```bash
   git checkout feature/v2.0-metadata
   git merge feature/phase-1-foundation
   ```

2. **Phase 2完了時**:
   ```bash
   git checkout feature/v2.0-metadata
   git merge feature/phase-2-references
   ```

3. **Phase 3完了時**:
   ```bash
   git checkout feature/v2.0-metadata
   git merge feature/phase-3-tasklinks
   ```

4. **Phase 4完了時**:
   ```bash
   git checkout feature/v2.0-metadata
   git merge feature/phase-4-status
   ```

5. **全Phase完了・統合テスト完了後**:
   ```bash
   git checkout main
   git merge feature/v2.0-metadata
   git tag v2.0.0
   ```

## ✅ データ整合性チェックリスト

### 保存時チェック
- [ ] すべてのグローバル変数がbuildMemoData()に含まれる
- [ ] version, schemaVersionが"2.0"に設定される
- [ ] exportedAtが現在時刻に設定される
- [ ] JSON形式が正しい（JSON.parse可能）
- [ ] ファイルサイズが適切（< 10MB）

### 読み込み時チェック
- [ ] マイグレーション関数が正しく実行される
- [ ] v1.0データが損失しない
- [ ] v2.0新フィールドが初期化される
- [ ] 参照矢印が正しく描画される
- [ ] ステータス色が正しく適用される
- [ ] タスク連携UIが更新される

### セキュリティチェック
- [ ] CSS.escape()がすべての要素ID参照で使用される
- [ ] Prototype Pollution対策が実装される
- [ ] XSS対策（HTML特殊文字エスケープ）が実装される
- [ ] LocalStorage読み込み時のエラーハンドリング
- [ ] 配列・オブジェクト型検証が実装される

## ⚠️ リスク管理マトリックス

| リスク | 影響度 | 発生確率 | 対策 | 担当フェーズ |
|-------|--------|----------|------|-------------|
| v1.0データ損失 | 高 | 中 | マイグレーション時のDeep Clone + 検証 | Phase 1 |
| Prototype Pollution攻撃 | 高 | 低 | キー名検証（__proto__等をブロック） | Phase 2, 3 |
| XSS攻撃（taskName経由） | 高 | 低 | HTML特殊文字エスケープ | Phase 3 |
| LocalStorage読み込み失敗 | 中 | 中 | try-catchとデフォルト値設定 | Phase 3 |
| パフォーマンス劣化（1000要素） | 中 | 中 | 矢印描画の最適化、DOM操作削減 | Phase 5 |
| ステータス色のアクセシビリティ | 低 | 中 | アイコンバッジ追加 | Phase 4-3 |
| 参照矢印の視認性低下 | 低 | 高 | 破線・色調整、背面配置 | Phase 2-2 |

## 📊 進捗トラッキング

### 完了済み
- ✅ Phase 0: 準備・設計確認 (1時間)
  - スキーマ設計確定
  - ドキュメント整備（4/5ファイル完了）
  - 他セッションとの調整

### 未着手
- ⬜ Phase 1: 基盤機能実装 (2-3時間)
- ⬜ Phase 2: 参照関係管理 (3-4時間)
- ⬜ Phase 3: タスク連携スナップショット (2-3時間)
- ⬜ Phase 4: ステータス色分け機能 (6-9時間)
- ⬜ Phase 5: 統合テスト・最適化 (1-2時間)

### 総進捗
- **完了**: 1時間 / 14-20時間 (5-7%)
- **残り**: 13-19時間
- **予定**: 2-3日間の実装作業

## 🔄 次のアクション

### 直近のタスク（Phase 0完了）
1. ✅ 04_data-structure.md 作成完了
2. ✅ 02_proposed-design.md 更新完了
3. ✅ 05_implementation-plan.md 更新完了
4. ✅ 07_integrated-roadmap.md 作成完了（本ドキュメント）
5. ⏳ 06_technical-concerns.md 更新（次タスク）

### Phase 1 開始前の準備
- [ ] 06_technical-concerns.md を Phase 4 懸念事項で更新
- [ ] ドキュメント体系の最終確認
- [ ] 実装開始判断（ユーザー確認）

## 📚 関連ドキュメント

| ドキュメント | 用途 |
|------------|------|
| [04_data-structure.md](./04_data-structure.md) | v2.0スキーマの完全仕様（Single Source of Truth） |
| [02_proposed-design.md](./02_proposed-design.md) | 設計方針・アーキテクチャ原則 |
| [05_implementation-plan.md](./05_implementation-plan.md) | 詳細実装タスク（Phase 1-4） |
| [06_technical-concerns.md](./06_technical-concerns.md) | 技術的懸念事項・リスク詳細 |
| [本ドキュメント](./07_integrated-roadmap.md) | 統合実装ロードマップ |

## 変更履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-22 | 1.0 | 初版作成 - Phase 0-5統合ロードマップ |

---

**最終更新**: 2026-03-22
**次のアクション**: 06_technical-concerns.md を更新（Phase 4 懸念事項追加）
