# タスク編集UI - フローチャート連携設定 詳細設計

**バージョン**: 1.0.0
**作成日**: 2026-03-20
**対象リリース**: v2.9.0

---

## 概要

タスク編集モーダルにフローチャート連携設定パネルを追加し、ユーザーがタスクとフローチャート要素（ノード/エッジ）を手動で紐付けできるようにします。

---

## UI構成

### 追加場所

既存のタスク編集モーダル内、既存フィールドの後に専用パネルを追加:

```
┌─────────────────────────────────────────────────────────────┐
│ タスク編集 - WBS1.1.0                          [_][□][×]    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ 【既存フィールド】                                           │
│ タスク名: [UI設計                                    ]       │
│ フェーズ: [PH1 ▾]                                            │
│ タスクタイプ: [Planning ▾]                                   │
│ 担当者（主）: [山田太郎                           ]          │
│ 担当者（副）: [佐藤花子                           ]          │
│ 期間（BD）: [5]                                              │
│ 開始日: [2026-03-23]  終了日: [2026-03-28]                  │
│ 前提タスク: [WBS1.0.0                            ]          │
│ ステータス: [Completed ▾]                                    │
│ ...                                                          │
│                                                              │
│ ─────────────────────────────────────────────────────────   │
│                                                              │
│ 【新規追加】                                                 │
│ ┌──────────────────────────────────────────────────────┐   │
│ │ 📊 フローチャート連携設定                             │   │
│ │                                                       │   │
│ │ （ノード連携セクション）                               │   │
│ │ （エッジ連携セクション - 折りたたみ）                  │   │
│ │ （プレビュー）                                         │   │
│ └──────────────────────────────────────────────────────┘   │
│                                                              │
│                            [キャンセル]  [保存]              │
└─────────────────────────────────────────────────────────────┘
```

### パネル詳細構造

```html
<div class="flowchart-link-panel" id="flowchart-link-panel">
  <!-- ヘッダー -->
  <div class="panel-header">
    <h3>📊 フローチャート連携設定</h3>
    <button class="btn-collapse" id="collapse-flowchart-panel">
      <span>▼</span>
    </button>
  </div>

  <!-- ヒント -->
  <p class="panel-hint">
    このタスクに関連するフローチャート要素を設定できます。
    クリック時にハイライト表示されます。
  </p>

  <!-- コンテンツエリア（折りたたみ可能） -->
  <div class="panel-content" id="flowchart-panel-content">

    <!-- ===== ノード連携セクション ===== -->
    <div class="link-section" id="node-link-section">
      <h4 class="section-title">🔵 ノード連携</h4>

      <!-- 現在の紐付け一覧 -->
      <div class="linked-items-container">
        <div class="linked-items-list" id="linked-nodes-list">
          <!-- 動的生成される紐付け済みノード -->
          <!-- <div class="linked-item" data-node-id="F_01">
            <span class="priority-badge primary">主要</span>
            <span class="item-label">F_01 - 要件定義</span>
            <button class="btn-remove" title="削除">×</button>
          </div> -->
        </div>
        <div class="empty-message" id="no-nodes-message" style="display: none;">
          紐付け済みノードはありません
        </div>
      </div>

      <!-- ノード追加コントロール -->
      <div class="add-link-controls">
        <div class="control-row">
          <select id="node-selector" class="form-select">
            <option value="">ノードを選択...</option>
            <!-- フローチャートデータから動的生成 -->
          </select>
          <select id="node-priority" class="form-select-sm">
            <option value="primary">主要</option>
            <option value="secondary">副次</option>
          </select>
          <button class="btn btn-sm btn-add" id="add-node-btn" disabled>
            ➕ 追加
          </button>
        </div>
      </div>
    </div>

    <!-- ===== エッジ連携セクション（折りたたみ） ===== -->
    <details class="link-section collapsible" id="edge-link-section">
      <summary class="section-title">🔗 エッジ連携（オプション）</summary>

      <!-- 現在の紐付け一覧 -->
      <div class="linked-items-container">
        <div class="linked-items-list" id="linked-edges-list">
          <!-- 動的生成される紐付け済みエッジ -->
          <!-- <div class="linked-item" data-edge-id="E_02">
            <span class="priority-badge primary">主要</span>
            <span class="item-label">E_02: F_01 → F_02</span>
            <span class="mode-badge">全体</span>
            <button class="btn-remove" title="削除">×</button>
          </div> -->
        </div>
        <div class="empty-message" id="no-edges-message" style="display: none;">
          紐付け済みエッジはありません
        </div>
      </div>

      <!-- エッジ追加コントロール -->
      <div class="add-link-controls">
        <div class="control-row">
          <select id="edge-selector" class="form-select">
            <option value="">エッジを選択...</option>
            <!-- フローチャートデータから動的生成 -->
          </select>
          <select id="edge-priority" class="form-select-sm">
            <option value="primary">主要</option>
            <option value="secondary">副次</option>
          </select>
          <button class="btn btn-sm btn-add" id="add-edge-btn" disabled>
            ➕ 追加
          </button>
        </div>
        <div class="control-row">
          <label class="checkbox-label">
            <input type="checkbox" id="edge-label-only">
            <span>ラベルのみハイライト（線は含まない）</span>
          </label>
        </div>
      </div>
    </details>

    <!-- ===== プレビュー ===== -->
    <div class="link-preview">
      <strong>📋 プレビュー:</strong>
      <span id="preview-text" class="preview-text">
        連携なし
      </span>
    </div>

  </div>
</div>
```

---

## CSS スタイル定義

```css
/* ==========================================
 * フローチャート連携パネル
 * ========================================== */
.flowchart-link-panel {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 6px;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.panel-header h3 {
  font-size: 16px;
  font-weight: 600;
  color: #495057;
  margin: 0;
}

.btn-collapse {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 14px;
  color: #6c757d;
  padding: 0.25rem 0.5rem;
  transition: transform 0.2s;
}

.btn-collapse.collapsed span {
  display: inline-block;
  transform: rotate(-90deg);
}

.panel-hint {
  font-size: 13px;
  color: #6c757d;
  margin-bottom: 1rem;
  padding: 0.5rem;
  background: #e7f1ff;
  border-left: 3px solid #007bff;
  border-radius: 3px;
}

.panel-content {
  display: block;
  transition: max-height 0.3s ease-out;
  overflow: hidden;
}

.panel-content.collapsed {
  max-height: 0;
}

/* ==========================================
 * リンクセクション
 * ========================================== */
.link-section {
  margin-bottom: 1.5rem;
}

.link-section.collapsible {
  padding: 0;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  background: white;
}

.section-title {
  font-size: 14px;
  font-weight: 600;
  color: #495057;
  margin-bottom: 0.75rem;
  padding: 0.5rem;
  background: white;
  border-radius: 4px;
}

.link-section.collapsible summary {
  cursor: pointer;
  user-select: none;
  list-style: none;
  margin: 0;
}

.link-section.collapsible summary::-webkit-details-marker {
  display: none;
}

.link-section.collapsible summary::before {
  content: '▶';
  display: inline-block;
  margin-right: 0.5rem;
  transition: transform 0.2s;
}

.link-section.collapsible[open] summary::before {
  transform: rotate(90deg);
}

.link-section.collapsible .linked-items-container,
.link-section.collapsible .add-link-controls {
  padding: 0.5rem;
}

/* ==========================================
 * 紐付け済みアイテム一覧
 * ========================================== */
.linked-items-container {
  margin-bottom: 0.75rem;
}

.linked-items-list {
  max-height: 200px;
  overflow-y: auto;
  margin-bottom: 0.5rem;
}

.linked-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  background: white;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  margin-bottom: 0.5rem;
  transition: all 0.2s;
}

.linked-item:hover {
  background: #f8f9fa;
  border-color: #adb5bd;
}

.priority-badge {
  padding: 0.25rem 0.5rem;
  border-radius: 3px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  white-space: nowrap;
}

.priority-badge.primary {
  background: #ffe0e0;
  color: #c82333;
  border: 1px solid #ff6b6b;
}

.priority-badge.secondary {
  background: #fff3cd;
  color: #856404;
  border: 1px solid #ff9800;
}

.item-label {
  flex: 1;
  font-size: 13px;
  color: #495057;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.mode-badge {
  padding: 0.25rem 0.5rem;
  border-radius: 3px;
  font-size: 11px;
  background: #e9ecef;
  color: #495057;
  border: 1px solid #ced4da;
}

.btn-remove {
  background: #dc3545;
  color: white;
  border: none;
  border-radius: 3px;
  padding: 0.25rem 0.5rem;
  cursor: pointer;
  font-size: 12px;
  font-weight: bold;
  transition: background 0.2s;
}

.btn-remove:hover {
  background: #c82333;
}

.empty-message {
  padding: 1rem;
  text-align: center;
  color: #6c757d;
  font-size: 13px;
  font-style: italic;
  background: #f8f9fa;
  border: 1px dashed #dee2e6;
  border-radius: 4px;
}

/* ==========================================
 * 追加コントロール
 * ========================================== */
.add-link-controls {
  padding: 0.75rem;
  background: white;
  border: 1px solid #dee2e6;
  border-radius: 4px;
}

.control-row {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  margin-bottom: 0.5rem;
}

.control-row:last-child {
  margin-bottom: 0;
}

.form-select {
  flex: 1;
  padding: 0.5rem;
  border: 1px solid #ced4da;
  border-radius: 4px;
  font-size: 13px;
  background: white;
  cursor: pointer;
}

.form-select:focus {
  outline: none;
  border-color: #007bff;
  box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25);
}

.form-select-sm {
  padding: 0.5rem;
  border: 1px solid #ced4da;
  border-radius: 4px;
  font-size: 12px;
  background: white;
  cursor: pointer;
  min-width: 80px;
}

.btn-add {
  padding: 0.5rem 1rem;
  background: #28a745;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 13px;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.2s;
}

.btn-add:hover:not(:disabled) {
  background: #218838;
}

.btn-add:disabled {
  background: #6c757d;
  cursor: not-allowed;
  opacity: 0.5;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 13px;
  color: #495057;
  cursor: pointer;
  user-select: none;
}

.checkbox-label input[type="checkbox"] {
  width: 16px;
  height: 16px;
  cursor: pointer;
}

/* ==========================================
 * プレビュー
 * ========================================== */
.link-preview {
  padding: 0.75rem;
  background: #e7f1ff;
  border: 1px solid #b8daff;
  border-radius: 4px;
  font-size: 13px;
  color: #004085;
}

.link-preview strong {
  display: inline-block;
  margin-right: 0.5rem;
}

.preview-text {
  font-style: italic;
}
```

---

## JavaScript 実装仕様

### 初期化処理

```javascript
/**
 * フローチャート連携パネルの初期化
 * @param {Object} task - 編集中のタスクオブジェクト
 * @param {Object} flowchartData - フローチャート定義データ
 */
function initFlowchartLinkPanel(task, flowchartData) {
  // ノードセレクターを構築
  populateNodeSelector(flowchartData.nodes);

  // エッジセレクターを構築
  populateEdgeSelector(flowchartData.edges);

  // 既存の紐付けを表示
  if (task.flowchartLinks) {
    renderLinkedNodes(task.flowchartLinks.nodes);
    renderLinkedEdges(task.flowchartLinks.edges);
  }

  // イベントリスナー設定
  setupEventListeners();

  // プレビュー更新
  updatePreview();
}

/**
 * ノードセレクターにオプションを追加
 */
function populateNodeSelector(nodes) {
  const selector = document.getElementById('node-selector');
  selector.innerHTML = '<option value="">ノードを選択...</option>';

  nodes.forEach(node => {
    const option = document.createElement('option');
    option.value = node.id;
    option.textContent = `${node.id} - ${node.label}`;
    option.dataset.label = node.label;
    selector.appendChild(option);
  });
}

/**
 * エッジセレクターにオプションを追加
 */
function populateEdgeSelector(edges) {
  const selector = document.getElementById('edge-selector');
  selector.innerHTML = '<option value="">エッジを選択...</option>';

  edges.forEach(edge => {
    const option = document.createElement('option');
    option.value = edge.id;

    // エッジ表示名を構築
    const fromNode = flowchartData.nodes.find(n => n.id === edge.from);
    const toNode = flowchartData.nodes.find(n => n.id === edge.to);
    const label = edge.label ? ` (${edge.label})` : '';
    option.textContent = `${edge.id}: ${fromNode.label} → ${toNode.label}${label}`;

    option.dataset.from = edge.from;
    option.dataset.to = edge.to;
    option.dataset.label = edge.label || '';

    selector.appendChild(option);
  });
}
```

### ノード追加処理

```javascript
/**
 * ノード追加ボタンのクリックハンドラー
 */
function onAddNodeClick() {
  const selector = document.getElementById('node-selector');
  const prioritySelector = document.getElementById('node-priority');

  const nodeId = selector.value;
  const priority = prioritySelector.value;

  if (!nodeId) {
    return;
  }

  // 既に紐付け済みかチェック
  const linkedNodesList = document.getElementById('linked-nodes-list');
  const existingItem = linkedNodesList.querySelector(`[data-node-id="${nodeId}"]`);
  if (existingItem) {
    alert('このノードは既に紐付けられています。');
    return;
  }

  // ノードラベルを取得
  const selectedOption = selector.options[selector.selectedIndex];
  const label = selectedOption.textContent;

  // 紐付けアイテムを追加
  addLinkedNode({
    id: nodeId,
    label: label,
    highlightType: priority
  });

  // セレクターをリセット
  selector.value = '';
  document.getElementById('add-node-btn').disabled = true;

  // プレビュー更新
  updatePreview();

  // 空メッセージを非表示
  document.getElementById('no-nodes-message').style.display = 'none';
}

/**
 * 紐付けノードをリストに追加
 */
function addLinkedNode(node) {
  const list = document.getElementById('linked-nodes-list');

  const item = document.createElement('div');
  item.className = 'linked-item';
  item.dataset.nodeId = node.id;

  item.innerHTML = `
    <span class="priority-badge ${node.highlightType}">${
      node.highlightType === 'primary' ? '主要' : '副次'
    }</span>
    <span class="item-label">${escapeHtml(node.label)}</span>
    <button class="btn-remove" title="削除">×</button>
  `;

  // 削除ボタンのイベント
  item.querySelector('.btn-remove').addEventListener('click', () => {
    if (confirm('この紐付けを削除しますか？')) {
      item.remove();
      updatePreview();

      // リストが空になったら空メッセージを表示
      if (list.children.length === 0) {
        document.getElementById('no-nodes-message').style.display = 'block';
      }
    }
  });

  list.appendChild(item);
}
```

### エッジ追加処理

```javascript
/**
 * エッジ追加ボタンのクリックハンドラー
 */
function onAddEdgeClick() {
  const selector = document.getElementById('edge-selector');
  const prioritySelector = document.getElementById('edge-priority');
  const labelOnlyCheckbox = document.getElementById('edge-label-only');

  const edgeId = selector.value;
  const priority = prioritySelector.value;
  const labelOnly = labelOnlyCheckbox.checked;

  if (!edgeId) {
    return;
  }

  // 既に紐付け済みかチェック
  const linkedEdgesList = document.getElementById('linked-edges-list');
  const existingItem = linkedEdgesList.querySelector(`[data-edge-id="${edgeId}"]`);
  if (existingItem) {
    alert('このエッジは既に紐付けられています。');
    return;
  }

  // エッジ情報を取得
  const selectedOption = selector.options[selector.selectedIndex];
  const label = selectedOption.textContent;
  const from = selectedOption.dataset.from;
  const to = selectedOption.dataset.to;
  const edgeLabel = selectedOption.dataset.label;

  // 紐付けアイテムを追加
  addLinkedEdge({
    id: edgeId,
    from: from,
    to: to,
    label: edgeLabel,
    displayLabel: label,
    highlightType: priority,
    highlightMode: labelOnly ? 'label-only' : 'full'
  });

  // セレクターをリセット
  selector.value = '';
  labelOnlyCheckbox.checked = false;
  document.getElementById('add-edge-btn').disabled = true;

  // プレビュー更新
  updatePreview();

  // 空メッセージを非表示
  document.getElementById('no-edges-message').style.display = 'none';
}

/**
 * 紐付けエッジをリストに追加
 */
function addLinkedEdge(edge) {
  const list = document.getElementById('linked-edges-list');

  const item = document.createElement('div');
  item.className = 'linked-item';
  item.dataset.edgeId = edge.id;

  item.innerHTML = `
    <span class="priority-badge ${edge.highlightType}">${
      edge.highlightType === 'primary' ? '主要' : '副次'
    }</span>
    <span class="item-label">${escapeHtml(edge.displayLabel)}</span>
    <span class="mode-badge">${
      edge.highlightMode === 'label-only' ? 'ラベルのみ' : '全体'
    }</span>
    <button class="btn-remove" title="削除">×</button>
  `;

  // 削除ボタンのイベント
  item.querySelector('.btn-remove').addEventListener('click', () => {
    if (confirm('この紐付けを削除しますか？')) {
      item.remove();
      updatePreview();

      // リストが空になったら空メッセージを表示
      if (list.children.length === 0) {
        document.getElementById('no-edges-message').style.display = 'block';
      }
    }
  });

  list.appendChild(item);
}
```

### プレビュー更新

```javascript
/**
 * プレビューテキストを更新
 */
function updatePreview() {
  const nodesList = document.getElementById('linked-nodes-list');
  const edgesList = document.getElementById('linked-edges-list');

  const nodes = Array.from(nodesList.children).map(item => {
    const nodeId = item.dataset.nodeId;
    const priority = item.querySelector('.priority-badge').classList.contains('primary')
      ? '主要'
      : '副次';
    return `${nodeId}（${priority}）`;
  });

  const edges = Array.from(edgesList.children).map(item => {
    const edgeId = item.dataset.edgeId;
    const priority = item.querySelector('.priority-badge').classList.contains('primary')
      ? '主要'
      : '副次';
    const mode = item.querySelector('.mode-badge').textContent;
    return `${edgeId}（${priority}・${mode}）`;
  });

  const previewText = document.getElementById('preview-text');

  if (nodes.length === 0 && edges.length === 0) {
    previewText.textContent = '連携なし';
    previewText.style.fontStyle = 'italic';
    return;
  }

  const parts = [];
  if (nodes.length > 0) {
    parts.push(`ノード: ${nodes.join(', ')}`);
  }
  if (edges.length > 0) {
    parts.push(`エッジ: ${edges.join(', ')}`);
  }

  previewText.textContent = `このタスククリック時に ${parts.join(' / ')} がハイライトされます`;
  previewText.style.fontStyle = 'normal';
}
```

### データ保存処理

```javascript
/**
 * タスク保存時にflowchartLinksを構築
 */
function buildFlowchartLinks() {
  const nodesList = document.getElementById('linked-nodes-list');
  const edgesList = document.getElementById('linked-edges-list');

  const flowchartLinks = {
    nodes: [],
    edges: []
  };

  // ノードを収集
  Array.from(nodesList.children).forEach(item => {
    const nodeId = item.dataset.nodeId;
    const label = item.querySelector('.item-label').textContent;
    const highlightType = item.querySelector('.priority-badge').classList.contains('primary')
      ? 'primary'
      : 'secondary';

    flowchartLinks.nodes.push({
      id: nodeId,
      label: label,
      highlightType: highlightType
    });
  });

  // エッジを収集
  Array.from(edgesList.children).forEach(item => {
    const edgeId = item.dataset.edgeId;

    // オリジナルのエッジデータを検索
    const originalEdge = flowchartData.edges.find(e => e.id === edgeId);
    if (!originalEdge) return;

    const highlightType = item.querySelector('.priority-badge').classList.contains('primary')
      ? 'primary'
      : 'secondary';

    const mode = item.querySelector('.mode-badge').textContent === 'ラベルのみ'
      ? 'label-only'
      : 'full';

    flowchartLinks.edges.push({
      id: edgeId,
      from: originalEdge.from,
      to: originalEdge.to,
      label: originalEdge.label || '',
      highlightType: highlightType,
      highlightMode: mode
    });
  });

  return flowchartLinks;
}

/**
 * タスク保存処理
 */
function saveTask() {
  // 既存フィールドの取得...
  const taskData = {
    wbs: document.getElementById('wbs').value,
    taskName: document.getElementById('taskName').value,
    // ...その他のフィールド

    // flowchartLinks を追加
    flowchartLinks: buildFlowchartLinks()
  };

  // mermaidIds を flowchartLinks から自動生成（後方互換性）
  taskData.mermaidIds = taskData.flowchartLinks.nodes
    .map(node => node.id)
    .join(',');

  // LocalStorage に保存
  saveTasks(taskData);

  // モーダルを閉じる
  closeTaskModal();
}
```

---

## イベントハンドラー設定

```javascript
/**
 * イベントリスナーの設定
 */
function setupEventListeners() {
  // ノードセレクター変更時
  document.getElementById('node-selector').addEventListener('change', (e) => {
    document.getElementById('add-node-btn').disabled = !e.target.value;
  });

  // ノード追加ボタン
  document.getElementById('add-node-btn').addEventListener('click', onAddNodeClick);

  // エッジセレクター変更時
  document.getElementById('edge-selector').addEventListener('change', (e) => {
    document.getElementById('add-edge-btn').disabled = !e.target.value;
  });

  // エッジ追加ボタン
  document.getElementById('add-edge-btn').addEventListener('click', onAddEdgeClick);

  // パネル折りたたみボタン
  document.getElementById('collapse-flowchart-panel').addEventListener('click', () => {
    const btn = document.getElementById('collapse-flowchart-panel');
    const content = document.getElementById('flowchart-panel-content');

    btn.classList.toggle('collapsed');
    content.classList.toggle('collapsed');
  });
}
```

---

## ユーティリティ関数

```javascript
/**
 * HTMLエスケープ
 */
function escapeHtml(text) {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}

/**
 * 既存の紐付けノードを表示
 */
function renderLinkedNodes(nodes) {
  const list = document.getElementById('linked-nodes-list');
  const noNodesMessage = document.getElementById('no-nodes-message');

  list.innerHTML = '';

  if (!nodes || nodes.length === 0) {
    noNodesMessage.style.display = 'block';
    return;
  }

  noNodesMessage.style.display = 'none';
  nodes.forEach(node => addLinkedNode(node));
}

/**
 * 既存の紐付けエッジを表示
 */
function renderLinkedEdges(edges) {
  const list = document.getElementById('linked-edges-list');
  const noEdgesMessage = document.getElementById('no-edges-message');

  list.innerHTML = '';

  if (!edges || edges.length === 0) {
    noEdgesMessage.style.display = 'block';
    return;
  }

  noEdgesMessage.style.display = 'none';
  edges.forEach(edge => {
    // エッジ表示ラベルを構築
    const fromNode = flowchartData.nodes.find(n => n.id === edge.from);
    const toNode = flowchartData.nodes.find(n => n.id === edge.to);
    const edgeLabel = edge.label ? ` (${edge.label})` : '';
    const displayLabel = `${edge.id}: ${fromNode.label} → ${toNode.label}${edgeLabel}`;

    addLinkedEdge({
      ...edge,
      displayLabel: displayLabel
    });
  });
}
```

---

## 実装チェックリスト

### Phase 1: 基本UI実装

- [ ] HTML構造追加（パネル、セクション、コントロール）
- [ ] CSS スタイル適用
- [ ] ノードセレクター構築
- [ ] エッジセレクター構築
- [ ] 初期化処理実装

### Phase 2: ノード連携機能

- [ ] ノード追加処理
- [ ] ノード削除処理
- [ ] 紐付け済みノード一覧表示
- [ ] 空メッセージ表示制御

### Phase 3: エッジ連携機能

- [ ] エッジ追加処理
- [ ] エッジ削除処理
- [ ] 紐付け済みエッジ一覧表示
- [ ] ラベルのみハイライトチェックボックス

### Phase 4: データ処理

- [ ] flowchartLinks構築処理
- [ ] mermaidIds自動同期
- [ ] タスク保存時の統合
- [ ] タスク読み込み時の表示

### Phase 5: UX改善

- [ ] プレビュー機能
- [ ] パネル折りたたみ機能
- [ ] 入力バリデーション
- [ ] エラーメッセージ表示
- [ ] 確認ダイアログ

---

## テストシナリオ

### 1. ノード追加

1. タスク編集モーダルを開く
2. ノードセレクターから「F_01 - 要件定義」を選択
3. 優先度「主要」を選択
4. 「➕追加」ボタンをクリック
5. **期待結果**: 紐付け済みノード一覧に表示される

### 2. ノード削除

1. 紐付け済みノードの「×削除」ボタンをクリック
2. 確認ダイアログで「OK」
3. **期待結果**: ノードが一覧から削除される

### 3. エッジ追加（全体ハイライト）

1. エッジ連携セクションを展開
2. エッジセレクターから「E_02: F_01 → F_02」を選択
3. 優先度「主要」を選択
4. ラベルのみチェックボックスは**オフ**
5. 「➕追加」ボタンをクリック
6. **期待結果**: モードバッジが「全体」で表示される

### 4. エッジ追加（ラベルのみ）

1. エッジセレクターから「E_03: F_02 → F_03 (YES)」を選択
2. 優先度「副次」を選択
3. ラベルのみチェックボックスを**オン**
4. 「➕追加」ボタンをクリック
5. **期待結果**: モードバッジが「ラベルのみ」で表示される

### 5. プレビュー確認

1. ノード「F_01」（主要）を追加
2. ノード「F_02」（副次）を追加
3. **期待結果**: プレビューが「このタスククリック時に ノード: F_01（主要）、F_02（副次） がハイライトされます」と表示される

### 6. 保存＆復元

1. ノードとエッジを複数紐付け
2. タスクを保存
3. タスク編集を再度開く
4. **期待結果**: 紐付けた内容が正しく復元される

---

## 変更履歴

| バージョン | 日付 | 変更内容 | 著者 |
|-----------|------|---------|------|
| 1.0.0 | 2026-03-20 | 初版作成 | ks-source |

---

**作成者**: ks-source
**最終更新**: 2026-03-20
