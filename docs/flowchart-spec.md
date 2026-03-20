# フローチャート機能 設計仕様書

**バージョン**: 1.0.0
**作成日**: 2026-03-20
**対象リリース**: v2.9.0（予定）

---

## 📋 目次

1. [概要](#概要)
2. [機能要件](#機能要件)
3. [データ構造設計](#データ構造設計)
4. [タスク・フローチャート手動連携](#タスクフローチャート手動連携)
5. [UI/UX設計](#uiux設計)
6. [双方向連携仕様](#双方向連携仕様)
7. [SVGレンダリング仕様](#svgレンダリング仕様)
8. [実装計画](#実装計画)
9. [技術制約](#技術制約)
10. [将来拡張](#将来拡張)

---

## 概要

### 目的

タスク管理システムに **フローチャート（プロセスフロー）の可視化機能** を追加し、各タスクがプロジェクト全体のどのプロセスに関連しているかを視覚的に把握できるようにする。

### 主要機能

1. **フローチャート別ウィンドウ表示**: ガントチャートと同様に独立ウィンドウで表示
2. **タスク連携**: タスクとフローチャートノードを `Mermaid_IDs` で紐付け
3. **双方向ハイライト**: 3画面（タスク/ガント/フロー）間の相互連携
4. **SVGベースレンダリング**: 外部ライブラリ不使用、純粋なSVG描画
5. **ビジュアル編集**: ノード/エッジのドラッグ&ドロップ編集
6. **ステータス同期**: タスクステータスに基づくノード色分け
7. **エクスポート機能**: SVG/PNG/JSON形式での出力

### 技術方針

- **外部ライブラリ不使用**: Mermaid.js等は使用せず、純粋なSVG DOMで実装
- **単一HTMLファイル**: 既存アーキテクチャを維持
- **LocalStorage連携**: ガントチャートと同じ通信方式
- **Edge標準API**: File System Access API、Canvas API等

---

## 機能要件

### 必須機能（v2.9.0）

| ID | 機能名 | 説明 | 優先度 |
|----|--------|------|--------|
| FC-001 | フローチャート別ウィンドウ表示 | `window.open()` で独立ウィンドウを開く | 高 |
| FC-002 | ノード基本描画 | 矩形、菱形、角丸矩形、楕円の描画 | 高 |
| FC-003 | エッジ基本描画 | 直線矢印、曲線矢印の描画 | 高 |
| FC-004 | タスク連携ハイライト | タスククリック時にノードをハイライト | 高 |
| FC-005 | ノードクリック連携 | ノードクリック時にタスク/ガントをハイライト | 高 |
| FC-006 | ステータス色分け | タスクステータスに応じたノード色変更 | 高 |
| FC-007 | LocalStorage保存 | フローチャート定義の永続化 | 高 |
| FC-008 | 自動更新 | 5秒間隔でタスクデータを同期 | 中 |
| FC-009 | 手動更新ボタン | ユーザー操作での即時更新 | 中 |

### オプション機能（v2.10.0以降）

| ID | 機能名 | 説明 | 優先度 |
|----|--------|------|--------|
| FC-101 | ノードドラッグ&ドロップ | マウスでノード位置を変更 | 中 |
| FC-102 | ノード追加UI | GUIでノードを追加 | 中 |
| FC-103 | エッジ追加UI | ノード間をクリックで接続 | 中 |
| FC-104 | SVGエクスポート | SVGファイルとして保存 | 中 |
| FC-105 | PNGエクスポート | Canvas変換でPNG出力 | 低 |
| FC-106 | 自動レイアウト | ノード配置の自動最適化 | 低 |
| FC-107 | ズーム/パン | 拡大縮小・スクロール操作 | 低 |
| FC-108 | ミニマップ | 全体像の縮小表示 | 低 |

---

## データ構造設計

### フローチャート定義（JSON）

```json
{
  "flowchart": {
    "metadata": {
      "version": "1.0",
      "created": "2026-03-20T10:00:00Z",
      "modified": "2026-03-20T15:30:00Z",
      "author": "ks-source"
    },
    "nodes": [
      {
        "id": "F_01",
        "type": "process",
        "label": "要件定義",
        "position": {
          "x": 100,
          "y": 50
        },
        "size": {
          "width": 140,
          "height": 60
        },
        "linkedTasks": ["WBS1.1.0", "WBS1.1.1"],
        "style": {
          "fill": "auto",
          "stroke": "#333",
          "strokeWidth": 2,
          "fontSize": 14
        }
      },
      {
        "id": "F_02",
        "type": "decision",
        "label": "承認OK?",
        "position": {
          "x": 300,
          "y": 50
        },
        "size": {
          "width": 100,
          "height": 80
        },
        "linkedTasks": ["WBS1.2.0"],
        "style": {
          "fill": "auto",
          "stroke": "#333",
          "strokeWidth": 2,
          "fontSize": 14
        }
      },
      {
        "id": "F_03",
        "type": "process",
        "label": "設計",
        "position": {
          "x": 100,
          "y": 200
        },
        "size": {
          "width": 140,
          "height": 60
        },
        "linkedTasks": ["WBS1.3.0"],
        "style": {
          "fill": "auto",
          "stroke": "#333",
          "strokeWidth": 2,
          "fontSize": 14
        }
      },
      {
        "id": "START",
        "type": "start",
        "label": "開始",
        "position": {
          "x": 100,
          "y": 0
        },
        "size": {
          "width": 80,
          "height": 40
        },
        "linkedTasks": [],
        "style": {
          "fill": "#4CAF50",
          "stroke": "#2E7D32",
          "strokeWidth": 2,
          "fontSize": 12
        }
      },
      {
        "id": "END",
        "type": "end",
        "label": "終了",
        "position": {
          "x": 100,
          "y": 400
        },
        "size": {
          "width": 80,
          "height": 40
        },
        "linkedTasks": [],
        "style": {
          "fill": "#f44336",
          "stroke": "#c62828",
          "strokeWidth": 2,
          "fontSize": 12
        }
      }
    ],
    "edges": [
      {
        "id": "E_01",
        "from": "START",
        "to": "F_01",
        "type": "arrow",
        "label": "",
        "style": {
          "stroke": "#333",
          "strokeWidth": 2,
          "strokeDasharray": "none"
        }
      },
      {
        "id": "E_02",
        "from": "F_01",
        "to": "F_02",
        "type": "arrow",
        "label": "",
        "style": {
          "stroke": "#333",
          "strokeWidth": 2,
          "strokeDasharray": "none"
        }
      },
      {
        "id": "E_03",
        "from": "F_02",
        "to": "F_03",
        "type": "arrow",
        "label": "YES",
        "style": {
          "stroke": "#4CAF50",
          "strokeWidth": 2,
          "strokeDasharray": "none"
        }
      },
      {
        "id": "E_04",
        "from": "F_02",
        "to": "F_01",
        "type": "arrow",
        "label": "NO",
        "style": {
          "stroke": "#f44336",
          "strokeWidth": 2,
          "strokeDasharray": "5,5"
        }
      },
      {
        "id": "E_05",
        "from": "F_03",
        "to": "END",
        "type": "arrow",
        "label": "",
        "style": {
          "stroke": "#333",
          "strokeWidth": 2,
          "strokeDasharray": "none"
        }
      }
    ],
    "canvas": {
      "width": 800,
      "height": 600,
      "backgroundColor": "#f8f9fa",
      "gridSize": 20,
      "showGrid": true
    }
  }
}
```

### ノードタイプ定義

| type | 形状 | 用途 | SVG要素 |
|------|------|------|---------|
| `start` | 角丸矩形 | 開始 | `<rect rx="20">` |
| `end` | 角丸矩形 | 終了 | `<rect rx="20">` |
| `process` | 矩形 | 処理 | `<rect>` |
| `decision` | 菱形 | 分岐 | `<path>` (ダイヤモンド) |
| `data` | 平行四辺形 | データ | `<path>` (傾斜矩形) |
| `manual` | 台形 | 手作業 | `<path>` |
| `document` | 波線下辺矩形 | ドキュメント | `<path>` |

### エッジタイプ定義

| type | 線種 | 用途 | SVG要素 |
|------|------|------|---------|
| `arrow` | 実線矢印 | 通常フロー | `<line>` + `<marker>` |
| `dashed-arrow` | 破線矢印 | 条件付きフロー | `<line stroke-dasharray>` |
| `bidirectional` | 双方向矢印 | 相互関係 | `<line>` + 2つの`<marker>` |

### ステータス色マッピング

```javascript
const STATUS_COLORS = {
  'Completed': {
    fill: '#d4edda',
    stroke: '#28a745',
    textColor: '#155724'
  },
  'In Progress': {
    fill: '#cfe2ff',
    stroke: '#007bff',
    textColor: '#004085'
  },
  'Not Started': {
    fill: '#e9ecef',
    stroke: '#6c757d',
    textColor: '#495057'
  },
  'On Hold': {
    fill: '#fff3cd',
    stroke: '#fd7e14',
    textColor: '#856404'
  },
  'Cancelled': {
    fill: '#f8d7da',
    stroke: '#dc3545',
    textColor: '#721c24'
  },
  'Postponed': {
    fill: '#f8d7da',
    stroke: '#e83e8c',
    textColor: '#721c24'
  }
};

// 複数タスク紐付き時のステータス優先順位
const STATUS_PRIORITY = [
  'In Progress',   // 最優先
  'On Hold',
  'Not Started',
  'Postponed',
  'Cancelled',
  'Completed'      // 最低優先
];
```

---

## タスク・フローチャート手動連携

### 背景と課題

フローチャートノード/エッジとタスクの紐付けを自動化することは困難であるため、ユーザーが手動で連携設定を行える仕組みを提供します。

### 設計方針

1. **タスク編集画面での紐付け設定**: タスク作成・編集時にフローチャート要素を選択
2. **拡張データ構造**: 既存の `mermaidIds` フィールドとの互換性を維持しつつ、詳細な連携情報を保持
3. **ハイライト優先度**: Primary（主要）/ Secondary（副次）の2段階でハイライト強度を制御
4. **双方向管理**: タスク→フローチャート、フローチャート→タスクの両方向で紐付け情報を保持

### 拡張データ構造

#### タスクデータ拡張

```json
{
  "wbs": "WBS1.1.0",
  "phase": "PH1",
  "taskType": "Planning",
  "taskName": "UI設計",
  "mermaidIds": "F_01,F_02",  // 既存フィールド（後方互換性維持）
  "primaryOwner": "山田太郎",
  "status": "Completed",

  // 新規フィールド: 詳細な連携情報
  "flowchartLinks": {
    "nodes": [
      {
        "id": "F_01",
        "label": "要件定義",
        "highlightType": "primary"  // "primary" | "secondary"
      },
      {
        "id": "F_02",
        "label": "要件承認OK?",
        "highlightType": "secondary"
      }
    ],
    "edges": [
      {
        "id": "E_02",
        "from": "F_01",
        "to": "F_02",
        "label": "",
        "highlightType": "primary",
        "highlightMode": "full"  // "full" (線+ラベル) | "label-only" (ラベルのみ)
      },
      {
        "id": "E_03",
        "from": "F_02",
        "to": "F_03",
        "label": "YES",
        "highlightType": "primary",
        "highlightMode": "label-only"
      }
    ]
  }
}
```

#### フローチャートデータ拡張

```json
{
  "flowchart": {
    "nodes": [
      {
        "id": "F_01",
        "type": "process",
        "label": "要件定義",
        "position": { "x": 100, "y": 50 },
        "size": { "width": 140, "height": 60 },

        // 逆方向の紐付け（オプション - 双方向管理の場合）
        "linkedTasks": [
          {
            "wbs": "WBS1.1.0",
            "taskName": "UI設計",
            "linkType": "primary"
          },
          {
            "wbs": "WBS1.1.1",
            "taskName": "要件定義書作成",
            "linkType": "secondary"
          }
        ]
      }
    ],
    "edges": [
      {
        "id": "E_02",
        "from": "F_01",
        "to": "F_02",
        "label": "",

        // エッジへのタスク紐付け（オプション）
        "linkedTasks": [
          {
            "wbs": "WBS1.2.0",
            "taskName": "レビュー承認",
            "linkType": "primary"
          }
        ]
      }
    ]
  }
}
```

### データマイグレーション戦略

既存の `mermaidIds` フィールドから `flowchartLinks` への自動変換ロジック:

```javascript
/**
 * 既存タスクデータを新フォーマットに変換
 */
function migrateTaskData(task) {
  // 既に flowchartLinks が存在する場合はスキップ
  if (task.flowchartLinks) {
    return task;
  }

  // mermaidIds が存在しない場合は空の構造を作成
  if (!task.mermaidIds || task.mermaidIds.trim() === '') {
    task.flowchartLinks = { nodes: [], edges: [] };
    return task;
  }

  // mermaidIds をカンマ区切りで分割してノードIDとして扱う
  const nodeIds = task.mermaidIds.split(',').map(id => id.trim()).filter(id => id);

  task.flowchartLinks = {
    nodes: nodeIds.map(id => ({
      id: id,
      label: getNodeLabel(id),  // フローチャートデータから取得
      highlightType: 'primary'  // デフォルトは primary
    })),
    edges: []  // 既存データではエッジ情報なし
  };

  return task;
}

/**
 * flowchartLinks から mermaidIds を同期
 */
function syncMermaidIds(task) {
  if (!task.flowchartLinks || !task.flowchartLinks.nodes) {
    task.mermaidIds = '';
    return task;
  }

  // ノードIDを mermaidIds フィールドに書き戻し（後方互換性）
  task.mermaidIds = task.flowchartLinks.nodes
    .map(node => node.id)
    .join(',');

  return task;
}
```

### ハイライト優先度仕様

#### Primary（主要）ハイライト

- **用途**: タスクに直接関連する最重要ノード/エッジ
- **視覚効果**:
  - 枠線: 4px、赤色（#ff6b6b）
  - ドロップシャドウ: 10px blur、赤色
  - アニメーション: パルス（1秒周期）
  - 自動スクロール: あり

```css
.node.highlighted-primary,
.edge.highlighted-primary {
  stroke: #ff6b6b !important;
  stroke-width: 4 !important;
  filter: drop-shadow(0 0 10px rgba(255, 107, 107, 0.8));
  animation: pulse-primary 1s infinite;
}

@keyframes pulse-primary {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}
```

#### Secondary（副次）ハイライト

- **用途**: タスクに間接的に関連するノード/エッジ
- **視覚効果**:
  - 枠線: 2px、オレンジ色（#ff9800）
  - ドロップシャドウ: なし
  - アニメーション: なし
  - 自動スクロール: なし

```css
.node.highlighted-secondary,
.edge.highlighted-secondary {
  stroke: #ff9800 !important;
  stroke-width: 2 !important;
}
```

### タスク編集UI詳細設計

#### フローチャート連携設定パネル

タスク編集モーダル内に専用パネルを追加:

```html
<!-- タスク編集モーダル内 -->
<div class="flowchart-link-panel">
  <h3>📊 フローチャート連携設定</h3>
  <p class="hint">このタスクに関連するフローチャート要素を設定できます</p>

  <!-- ノード連携セクション -->
  <div class="link-section">
    <h4>🔵 ノード連携</h4>

    <!-- 現在の紐付け一覧 -->
    <div class="linked-items-list" id="linked-nodes-list">
      <!-- 動的生成: -->
      <!-- <div class="linked-item">
        <span class="priority-badge primary">主要</span>
        <span class="node-info">F_01 - 要件定義</span>
        <button class="btn-remove" data-node-id="F_01">×削除</button>
      </div> -->
    </div>

    <!-- ノード追加UI -->
    <div class="add-link-controls">
      <select id="node-selector" class="form-select">
        <option value="">ノードを選択...</option>
        <option value="F_01">F_01 - 要件定義</option>
        <option value="F_02">F_02 - 要件承認OK?</option>
        <!-- フローチャートデータから動的生成 -->
      </select>
      <select id="node-priority" class="form-select-sm">
        <option value="primary">主要</option>
        <option value="secondary">副次</option>
      </select>
      <button class="btn btn-sm" id="add-node-btn">➕追加</button>
    </div>
  </div>

  <!-- エッジ連携セクション（折りたたみ可能） -->
  <details class="link-section">
    <summary><h4>🔗 エッジ連携（オプション）</h4></summary>

    <div class="linked-items-list" id="linked-edges-list">
      <!-- 同様の構造 -->
    </div>

    <div class="add-link-controls">
      <select id="edge-selector" class="form-select">
        <option value="">エッジを選択...</option>
        <option value="E_02">E_02: F_01 → F_02</option>
        <option value="E_03">E_03: F_02 → F_03 (YES)</option>
        <!-- フローチャートデータから動的生成 -->
      </select>
      <select id="edge-priority" class="form-select-sm">
        <option value="primary">主要</option>
        <option value="secondary">副次</option>
      </select>
      <label>
        <input type="checkbox" id="edge-label-only">
        ラベルのみハイライト
      </label>
      <button class="btn btn-sm" id="add-edge-btn">➕追加</button>
    </div>
  </details>

  <!-- プレビュー -->
  <div class="link-preview">
    <strong>📋 プレビュー:</strong>
    <span id="preview-text">このタスククリック時に F_01（主要）、F_02（副次）がハイライトされます</span>
  </div>
</div>
```

#### UI操作フロー

```
1. タスク編集モーダルを開く
   ├─ 新規作成: 空の状態
   └─ 編集: 既存の flowchartLinks を表示

2. ノード追加
   ├─ [ノード選択]ドロップダウンから選択
   ├─ [主要/副次]を選択
   ├─ [➕追加]ボタンをクリック
   └─ linked-nodes-list に追加表示

3. ノード削除
   ├─ 削除したいノードの[×削除]ボタンをクリック
   └─ 確認ダイアログ → 削除

4. エッジ追加（オプション）
   ├─ [エッジ連携]セクションを展開
   ├─ [エッジ選択]ドロップダウンから選択
   ├─ [主要/副次]を選択
   ├─ [ラベルのみハイライト]チェックボックス（任意）
   ├─ [➕追加]ボタンをクリック
   └─ linked-edges-list に追加表示

5. 保存
   ├─ [保存]ボタンをクリック
   ├─ flowchartLinks オブジェクトを構築
   ├─ mermaidIds フィールドを自動同期
   └─ LocalStorage に保存
```

### フローチャート画面での表示

#### TODOタスク一覧の表示仕様

左側サイドバーに、従来のノード一覧の代わりにTODOタスク一覧を表示:

```html
<div class="sidebar">
  <div class="sidebar-header">
    📋 TODOタスク一覧
    <button class="btn-toggle-completed">完了済みを隠す</button>
  </div>

  <div class="task-list">
    <!-- 各タスク -->
    <div class="task-item" data-wbs="WBS1.1.0" data-status="completed">
      <div class="task-header">
        <input type="checkbox" class="task-checkbox" checked>
        <span class="task-wbs">WBS1.1.0</span>
        <span class="status-badge completed">完了</span>
      </div>
      <div class="task-name">UI設計</div>
      <div class="task-links">
        <span class="link-icon" title="フローチャート連携: 2ノード">📌 F_01, F_02</span>
      </div>
    </div>

    <div class="task-item" data-wbs="WBS1.2.0" data-status="in-progress">
      <div class="task-header">
        <input type="checkbox" class="task-checkbox">
        <span class="task-wbs">WBS1.2.0</span>
        <span class="status-badge in-progress">進行中</span>
      </div>
      <div class="task-name">基本設計</div>
      <div class="task-links">
        <span class="link-icon" title="フローチャート連携: 1ノード">📌 F_03</span>
      </div>
    </div>

    <div class="task-item" data-wbs="WBS1.3.0" data-status="not-started">
      <div class="task-header">
        <input type="checkbox" class="task-checkbox">
        <span class="task-wbs">WBS1.3.0</span>
        <span class="status-badge not-started">未着手</span>
      </div>
      <div class="task-name">詳細設計</div>
      <div class="task-links">
        <span class="link-icon gray" title="連携なし">📌 未設定</span>
      </div>
    </div>
  </div>
</div>
```

#### タスククリック時のハイライト動作

```javascript
/**
 * タスク一覧項目クリック時の処理
 */
function onTaskItemClick(wbs) {
  // タスクデータを取得
  const task = tasks.find(t => t.wbs === wbs);
  if (!task || !task.flowchartLinks) {
    return;
  }

  // 全てのハイライトをクリア
  clearAllHighlights();

  // タスク項目をアクティブ化
  document.querySelectorAll('.task-item').forEach(item => {
    item.classList.remove('active');
  });
  document.querySelector(`[data-wbs="${wbs}"]`).classList.add('active');

  // Primary ノードをハイライト
  const primaryNodes = task.flowchartLinks.nodes
    .filter(n => n.highlightType === 'primary');

  primaryNodes.forEach(node => {
    const svgNode = document.querySelector(`[data-node-id="${node.id}"]`);
    if (svgNode) {
      svgNode.classList.add('highlighted-primary');
    }
  });

  // 最初の Primary ノードまでスクロール
  if (primaryNodes.length > 0) {
    const firstNode = document.querySelector(`[data-node-id="${primaryNodes[0].id}"]`);
    if (firstNode) {
      firstNode.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }
  }

  // Secondary ノードをハイライト
  const secondaryNodes = task.flowchartLinks.nodes
    .filter(n => n.highlightType === 'secondary');

  secondaryNodes.forEach(node => {
    const svgNode = document.querySelector(`[data-node-id="${node.id}"]`);
    if (svgNode) {
      svgNode.classList.add('highlighted-secondary');
    }
  });

  // エッジをハイライト
  task.flowchartLinks.edges.forEach(edge => {
    const svgEdge = document.querySelector(`[data-edge-id="${edge.id}"]`);
    if (svgEdge) {
      const highlightClass = edge.highlightType === 'primary'
        ? 'highlighted-primary'
        : 'highlighted-secondary';

      if (edge.highlightMode === 'label-only') {
        // ラベルのみハイライト
        const edgeLabel = svgEdge.querySelector('text');
        if (edgeLabel) {
          edgeLabel.classList.add(highlightClass);
        }
      } else {
        // エッジ全体をハイライト
        svgEdge.classList.add(highlightClass);
      }
    }
  });

  // LocalStorage に選択状態を保存（他ウィンドウとの連携）
  publishSelection({
    source: 'flowchart-task-list',
    wbs: task.wbs,
    taskName: task.taskName,
    flowchartLinks: task.flowchartLinks,
    timestamp: Date.now()
  });
}
```

### 実装優先順位

#### Phase 1: 基本的な手動連携（v2.9.0必須）

1. **データ構造拡張**: 8-12時間
   - [ ] `flowchartLinks` フィールド定義
   - [ ] マイグレーション関数実装
   - [ ] `mermaidIds` との同期ロジック
   - [ ] LocalStorage 読み書き対応

2. **タスク編集UI**: 10-14時間
   - [ ] フローチャート連携設定パネル追加
   - [ ] ノード選択ドロップダウン実装
   - [ ] 紐付け済みノード一覧表示
   - [ ] 追加/削除ボタン実装
   - [ ] Primary/Secondary 選択UI

3. **フローチャート画面TODOリスト**: 6-8時間
   - [ ] 従来のノード一覧をTODOタスク一覧に置き換え
   - [ ] タスク項目表示（WBS、タスク名、ステータス、連携情報）
   - [ ] タスククリック時のハイライト処理
   - [ ] Primary/Secondary 別ハイライトスタイル適用

**Phase 1 合計工数**: 24-34時間（3-4日）

#### Phase 2: エッジ連携（v2.10.0）

4. **エッジ連携機能**: 6-8時間
   - [ ] タスク編集UIにエッジ選択追加
   - [ ] エッジハイライト処理実装
   - [ ] ラベルのみハイライトモード実装

#### Phase 3: 逆方向紐付け（v2.11.0以降 - オプション）

5. **フローチャート画面から紐付け**: 8-12時間
   - [ ] ノード右クリックメニュー
   - [ ] タスク選択ダイアログ
   - [ ] 双方向同期ロジック

---

## UI/UX設計

### ウィンドウレイアウト

```
┌────────────────────────────────────────────────────────────────┐
│ フローチャート - プロジェクト名                    [_][□][×]   │
├────────────────────────────────────────────────────────────────┤
│ [🔄更新] [💾保存] [📤エクスポート▾] [➕ノード] [🔗エッジ] [⚙設定] │
├────────────────────────────────────────────────────────────────┤
│ ┌──────────┐                                                   │
│ │ ノード   │  凡例: ■完了 ■進行中 ■未着手 ■保留 ■中止        │
│ │ リスト   │                                                   │
│ ├──────────┤                                                   │
│ │□ START   │  ┌─────────────────────────────────────────────┐ │
│ │□ F_01    │  │         SVGキャンバス                        │ │
│ │□ F_02    │  │                                              │ │
│ │□ F_03    │  │   ┌────────┐                                │ │
│ │□ END     │  │   │ 開始   │                                │ │
│ │          │  │   └───┬────┘                                │ │
│ │[追加]    │  │       ↓                                      │ │
│ │[削除]    │  │   ┌────────┐        ┌────────┐             │ │
│ │          │  │   │ 要件定義│───────→│承認OK? │             │ │
│ │          │  │   │  F_01  │        │  F_02  │             │ │
│ │          │  │   └───┬────┘        └───┬─┬──┘             │ │
│ │          │  │       ↑                 │ │ NO             │ │
│ │          │  │       └─────────────────┘ │                │ │
│ │          │  │                   YES     ↓                │ │
│ │          │  │                       ┌────────┐           │ │
│ │          │  │                       │  設計  │           │ │
│ │          │  │                       │  F_03  │           │ │
│ │          │  │                       └───┬────┘           │ │
│ │          │  │                           ↓                │ │
│ │          │  │                       ┌────────┐           │ │
│ │          │  │                       │  終了  │           │ │
│ │          │  │                       └────────┘           │ │
│ └──────────┘  └─────────────────────────────────────────────┘ │
│                                                                │
│ ステータス: 5ノード | 選択: F_01 | 座標: (100, 50)             │
└────────────────────────────────────────────────────────────────┘
```

### ツールバー機能

| ボタン | アイコン | 機能 | ショートカット |
|--------|---------|------|---------------|
| 更新 | 🔄 | タスクデータを再読み込み | F5 |
| 保存 | 💾 | フローチャート定義を保存 | Ctrl+S |
| エクスポート | 📤 | SVG/PNG/JSONで出力 | - |
| 設定 | ⚙ | 表示設定・グリッド等 | - |

**注**: ノード追加・エッジ追加機能は将来バージョン（v2.11.0以降）で実装予定です。

### ノード表示仕様

```svg
<!-- プロセスノード（矩形） -->
<g class="node" data-node-id="F_01" data-status="in-progress">
  <rect x="100" y="50" width="140" height="60"
        fill="#cfe2ff" stroke="#007bff" stroke-width="2" rx="5"/>
  <text x="170" y="75" text-anchor="middle"
        font-size="12" font-weight="600" fill="#004085">
    F_01
  </text>
  <text x="170" y="90" text-anchor="middle"
        font-size="14" fill="#004085">
    要件定義
  </text>
</g>

<!-- 分岐ノード（菱形） -->
<g class="node" data-node-id="F_02" data-status="not-started">
  <path d="M 300 50 L 350 90 L 300 130 L 250 90 Z"
        fill="#e9ecef" stroke="#6c757d" stroke-width="2"/>
  <text x="300" y="90" text-anchor="middle"
        font-size="14" fill="#495057">
    承認OK?
  </text>
</g>

<!-- エッジ（矢印） -->
<g class="edge" data-edge-id="E_02">
  <line x1="240" y1="80" x2="250" y2="90"
        stroke="#333" stroke-width="2" marker-end="url(#arrowhead)"/>
  <text x="245" y="75" font-size="12" fill="#333">YES</text>
</g>

<!-- 矢印マーカー定義 -->
<defs>
  <marker id="arrowhead" markerWidth="10" markerHeight="10"
          refX="9" refY="3" orient="auto">
    <polygon points="0 0, 10 3, 0 6" fill="#333"/>
  </marker>
  <marker id="arrowhead-green" markerWidth="10" markerHeight="10"
          refX="9" refY="3" orient="auto">
    <polygon points="0 0, 10 3, 0 6" fill="#4CAF50"/>
  </marker>
  <marker id="arrowhead-red" markerWidth="10" markerHeight="10"
          refX="9" refY="3" orient="auto">
    <polygon points="0 0, 10 3, 0 6" fill="#f44336"/>
  </marker>
</defs>
```

### ハイライト表現

```css
/* 通常状態 */
.node {
  cursor: pointer;
  transition: all 0.3s ease;
}

/* ハイライト状態 */
.node.highlighted {
  filter: drop-shadow(0 0 10px rgba(255, 107, 107, 0.8));
  animation: pulse 1s infinite;
}

.node.highlighted rect,
.node.highlighted path {
  stroke: #ff6b6b !important;
  stroke-width: 4 !important;
}

/* パルスアニメーション */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

/* ホバー時 */
.node:hover {
  filter: drop-shadow(0 0 5px rgba(0, 123, 255, 0.5));
}

.node:hover rect,
.node:hover path {
  stroke-width: 3;
}
```

---

## 双方向連携仕様

### 通信プロトコル（LocalStorage）

#### メッセージフォーマット

```javascript
// LocalStorageキー
const STORAGE_KEY = 'taskSelection';

// メッセージ構造
interface SelectionMessage {
  // 必須フィールド
  source: 'task' | 'gantt' | 'flowchart';
  timestamp: number;
  action: 'select' | 'deselect' | 'hover';

  // タスク関連（source: 'task' | 'gantt'）
  wbs?: string;              // "WBS1.1.0"
  taskName?: string;         // "要件定義"
  mermaidIds?: string[];     // ["F_01", "F_02"]

  // ノード関連（source: 'flowchart'）
  nodeId?: string;           // "F_01"
  nodeName?: string;         // "要件定義"
  relatedTasks?: string[];   // ["WBS1.1.0", "WBS1.1.1"]
}
```

#### 送信例

```javascript
// フローチャート画面でノードクリック時
function onNodeClick(nodeId) {
  const node = flowchartData.nodes.find(n => n.id === nodeId);

  const message = {
    source: 'flowchart',
    timestamp: Date.now(),
    action: 'select',
    nodeId: node.id,
    nodeName: node.label,
    relatedTasks: node.linkedTasks
  };

  localStorage.setItem(STORAGE_KEY, JSON.stringify(message));
  highlightNode(nodeId);  // 自画面でもハイライト
}
```

#### 受信例

```javascript
// ガントチャート画面での受信
window.addEventListener('storage', (e) => {
  if (e.key === STORAGE_KEY && e.newValue) {
    const message = JSON.parse(e.newValue);

    // 他画面からの通知のみ処理
    if (message.source !== 'gantt') {
      if (message.relatedTasks) {
        // フローチャートからの通知
        highlightTasks(message.relatedTasks);
        scrollToTask(message.relatedTasks[0]);
      } else if (message.wbs) {
        // タスク画面からの通知
        highlightTask(message.wbs);
        scrollToTask(message.wbs);
      }
    }
  }
});
```

### 連携シーケンス

```
┌──────────┐        ┌──────────┐        ┌──────────┐
│フローチャート│        │LocalStorage│        │ガント    │
└─────┬────┘        └─────┬────┘        └────┬─────┘
      │                   │                  │
      │ ノードクリック     │                  │
      │ (F_01)            │                  │
      ├──────────────────>│                  │
      │ setItem()         │                  │
      │                   │                  │
      │                   │ storage event    │
      │                   ├─────────────────>│
      │                   │                  │
      │                   │                  │ relatedTasks
      │                   │                  │ = ["WBS1.1.0"]
      │                   │                  │
      │                   │                  ├─ highlight
      │                   │                  │  WBS1.1.0
      │                   │                  │
      │                   │                  ├─ scrollToTask
      │                   │                  │  WBS1.1.0
      │                   │                  │
```

### ハイライト自動解除

```javascript
const HIGHLIGHT_DURATION = 5000; // 5秒
let highlightTimer = null;

function highlightWithTimeout(elements, duration = HIGHLIGHT_DURATION) {
  // 既存のタイマーをクリア
  if (highlightTimer) {
    clearTimeout(highlightTimer);
  }

  // ハイライトを追加
  elements.forEach(el => {
    el.classList.add('highlighted');
  });

  // 一定時間後に自動解除
  highlightTimer = setTimeout(() => {
    elements.forEach(el => {
      el.classList.remove('highlighted');
    });
  }, duration);
}
```

---

## SVGレンダリング仕様

### SVG基本構造

```svg
<svg id="flowchart-canvas" width="800" height="600"
     xmlns="http://www.w3.org/2000/svg"
     viewBox="0 0 800 600">

  <!-- グリッド背景（オプション） -->
  <defs>
    <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
      <path d="M 20 0 L 0 0 0 20" fill="none" stroke="#e0e0e0" stroke-width="0.5"/>
    </pattern>
  </defs>
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- 矢印マーカー定義 -->
  <defs>
    <!-- 通常矢印 -->
    <marker id="arrowhead" markerWidth="10" markerHeight="10"
            refX="9" refY="3" orient="auto">
      <polygon points="0 0, 10 3, 0 6" fill="#333"/>
    </marker>

    <!-- ハイライト矢印 -->
    <marker id="arrowhead-highlight" markerWidth="10" markerHeight="10"
            refX="9" refY="3" orient="auto">
      <polygon points="0 0, 10 3, 0 6" fill="#ff6b6b"/>
    </marker>
  </defs>

  <!-- エッジレイヤー（ノードの下） -->
  <g id="edges-layer">
    <g class="edge" data-edge-id="E_01">
      <line x1="140" y1="20" x2="100" y2="50"
            stroke="#333" stroke-width="2"
            marker-end="url(#arrowhead)"/>
    </g>
  </g>

  <!-- ノードレイヤー（エッジの上） -->
  <g id="nodes-layer">
    <g class="node" data-node-id="F_01" data-status="in-progress">
      <rect x="100" y="50" width="140" height="60"
            fill="#cfe2ff" stroke="#007bff" stroke-width="2" rx="5"/>
      <text x="170" y="75" text-anchor="middle"
            font-size="12" font-weight="600" fill="#004085">F_01</text>
      <text x="170" y="90" text-anchor="middle"
            font-size="14" fill="#004085">要件定義</text>
    </g>
  </g>

</svg>
```

### ノード形状生成関数

```javascript
// 矩形ノード
function createRectNode(node) {
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  g.classList.add('node');
  g.setAttribute('data-node-id', node.id);

  const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect');
  rect.setAttribute('x', node.position.x);
  rect.setAttribute('y', node.position.y);
  rect.setAttribute('width', node.size.width);
  rect.setAttribute('height', node.size.height);
  rect.setAttribute('fill', node.style.fill);
  rect.setAttribute('stroke', node.style.stroke);
  rect.setAttribute('stroke-width', node.style.strokeWidth);
  rect.setAttribute('rx', '5');

  const text = createNodeText(node);

  g.appendChild(rect);
  g.appendChild(text);

  return g;
}

// 菱形ノード
function createDiamondNode(node) {
  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  g.classList.add('node');
  g.setAttribute('data-node-id', node.id);

  const cx = node.position.x + node.size.width / 2;
  const cy = node.position.y + node.size.height / 2;
  const w = node.size.width / 2;
  const h = node.size.height / 2;

  const path = document.createElementNS('http://www.w3.org/2000/svg', 'path');
  const d = `M ${cx} ${cy - h} L ${cx + w} ${cy} L ${cx} ${cy + h} L ${cx - w} ${cy} Z`;
  path.setAttribute('d', d);
  path.setAttribute('fill', node.style.fill);
  path.setAttribute('stroke', node.style.stroke);
  path.setAttribute('stroke-width', node.style.strokeWidth);

  const text = createNodeText(node);

  g.appendChild(path);
  g.appendChild(text);

  return g;
}

// ノードテキスト生成
function createNodeText(node) {
  const cx = node.position.x + node.size.width / 2;
  const cy = node.position.y + node.size.height / 2;

  const textGroup = document.createElementNS('http://www.w3.org/2000/svg', 'g');

  // ノードID
  const idText = document.createElementNS('http://www.w3.org/2000/svg', 'text');
  idText.setAttribute('x', cx);
  idText.setAttribute('y', cy - 8);
  idText.setAttribute('text-anchor', 'middle');
  idText.setAttribute('font-size', '12');
  idText.setAttribute('font-weight', '600');
  idText.textContent = node.id;

  // ノードラベル
  const labelText = document.createElementNS('http://www.w3.org/2000/svg', 'text');
  labelText.setAttribute('x', cx);
  labelText.setAttribute('y', cy + 8);
  labelText.setAttribute('text-anchor', 'middle');
  labelText.setAttribute('font-size', '14');
  labelText.textContent = node.label;

  textGroup.appendChild(idText);
  textGroup.appendChild(labelText);

  return textGroup;
}

// エッジ生成
function createEdge(edge, nodes) {
  const fromNode = nodes.find(n => n.id === edge.from);
  const toNode = nodes.find(n => n.id === edge.to);

  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
  g.classList.add('edge');
  g.setAttribute('data-edge-id', edge.id);

  const x1 = fromNode.position.x + fromNode.size.width / 2;
  const y1 = fromNode.position.y + fromNode.size.height;
  const x2 = toNode.position.x + toNode.size.width / 2;
  const y2 = toNode.position.y;

  const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
  line.setAttribute('x1', x1);
  line.setAttribute('y1', y1);
  line.setAttribute('x2', x2);
  line.setAttribute('y2', y2);
  line.setAttribute('stroke', edge.style.stroke);
  line.setAttribute('stroke-width', edge.style.strokeWidth);
  line.setAttribute('marker-end', 'url(#arrowhead)');

  if (edge.style.strokeDasharray !== 'none') {
    line.setAttribute('stroke-dasharray', edge.style.strokeDasharray);
  }

  g.appendChild(line);

  // エッジラベル（あれば）
  if (edge.label) {
    const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
    text.setAttribute('x', (x1 + x2) / 2);
    text.setAttribute('y', (y1 + y2) / 2 - 5);
    text.setAttribute('text-anchor', 'middle');
    text.setAttribute('font-size', '12');
    text.setAttribute('fill', edge.style.stroke);
    text.textContent = edge.label;
    g.appendChild(text);
  }

  return g;
}
```

### レンダリング処理

```javascript
function renderFlowchart() {
  const svg = document.getElementById('flowchart-canvas');
  const nodesLayer = document.getElementById('nodes-layer');
  const edgesLayer = document.getElementById('edges-layer');

  // クリア
  nodesLayer.innerHTML = '';
  edgesLayer.innerHTML = '';

  // エッジを先に描画（ノードの下層）
  flowchartData.edges.forEach(edge => {
    const edgeElement = createEdge(edge, flowchartData.nodes);
    edgesLayer.appendChild(edgeElement);
  });

  // ノードを描画
  flowchartData.nodes.forEach(node => {
    let nodeElement;

    switch (node.type) {
      case 'process':
      case 'start':
      case 'end':
        nodeElement = createRectNode(node);
        break;
      case 'decision':
        nodeElement = createDiamondNode(node);
        break;
      default:
        nodeElement = createRectNode(node);
    }

    // クリックイベント
    nodeElement.addEventListener('click', () => {
      onNodeClick(node.id);
    });

    // ホバーイベント
    nodeElement.addEventListener('mouseenter', () => {
      showNodeTooltip(node);
    });
    nodeElement.addEventListener('mouseleave', () => {
      hideNodeTooltip();
    });

    nodesLayer.appendChild(nodeElement);
  });

  // ステータス色を更新
  updateNodeColors();
}
```

### ステータス色更新

```javascript
function updateNodeColors() {
  flowchartData.nodes.forEach(node => {
    if (node.linkedTasks.length === 0) {
      return; // タスク紐付けなしはデフォルト色
    }

    // 関連タスクのステータスを取得
    const relatedTasks = tasks.filter(t =>
      node.linkedTasks.includes(t.wbs)
    );

    if (relatedTasks.length === 0) {
      return;
    }

    // ステータス優先順位で色を決定
    let dominantStatus = 'Not Started';
    for (const status of STATUS_PRIORITY) {
      if (relatedTasks.some(t => t.status === status)) {
        dominantStatus = status;
        break;
      }
    }

    // ノード要素を取得して色を更新
    const nodeElement = document.querySelector(`[data-node-id="${node.id}"]`);
    if (nodeElement) {
      const shape = nodeElement.querySelector('rect, path');
      const colors = STATUS_COLORS[dominantStatus];

      shape.setAttribute('fill', colors.fill);
      shape.setAttribute('stroke', colors.stroke);

      const texts = nodeElement.querySelectorAll('text');
      texts.forEach(text => {
        text.setAttribute('fill', colors.textColor);
      });
    }
  });
}
```

---

## 実装計画

### Phase 1: 基本構造（v2.9.0-alpha）

**期間**: 2-3日
**工数**: 8-12時間

- [ ] フローチャートデータ構造定義
- [ ] LocalStorageへの保存/読み込み
- [ ] 別ウィンドウ表示機能
- [ ] SVGキャンバス基本実装
- [ ] ノード描画（矩形、菱形）
- [ ] エッジ描画（直線矢印）
- [ ] 手動更新ボタン

### Phase 2: タスク連携（v2.9.0-beta）

**期間**: 2-3日
**工数**: 10-14時間

- [ ] LocalStorage通信基盤
- [ ] タスク→フローハイライト
- [ ] ガント→フローハイライト
- [ ] フロー→タスク/ガントハイライト
- [ ] 自動スクロール機能
- [ ] ハイライトアニメーション

### Phase 3: ステータス連携（v2.9.0-rc）

**期間**: 1-2日
**工数**: 4-6時間

- [ ] ステータス色マッピング
- [ ] ノード色自動更新
- [ ] 5秒自動更新機能
- [ ] 複数タスク紐付き時の色決定ロジック

### Phase 4: UI改善（v2.9.0）

**期間**: 1-2日
**工数**: 4-6時間

- [ ] ツールバー実装
- [ ] ノードリストパネル
- [ ] ステータスバー
- [ ] レスポンシブデザイン調整
- [ ] エラーハンドリング

### Phase 5: エクスポート機能（v2.10.0）

**期間**: 2-3日
**工数**: 8-12時間

- [ ] SVGエクスポート
- [ ] PNGエクスポート（Canvas変換）
- [ ] JSON定義エクスポート
- [ ] インポート機能

### Phase 6: 編集機能（v2.11.0）

**期間**: 3-5日
**工数**: 16-24時間

- [ ] ノードドラッグ&ドロップ
- [ ] ノード追加UI
- [ ] ノード削除機能
- [ ] エッジ追加UI
- [ ] エッジ削除機能
- [ ] プロパティ編集ダイアログ

### 総工数見積

- **v2.9.0（最小機能版）**: 26-38時間（3-5日）
- **v2.10.0（エクスポート追加）**: +8-12時間（+1-2日）
- **v2.11.0（編集機能追加）**: +16-24時間（+2-3日）

---

## 技術制約

### 必須要件

1. **外部ライブラリ不使用**
   - Mermaid.js、D3.js、Cytoscape.js等は使用不可
   - 純粋なSVG DOM APIで実装

2. **単一HTMLファイル構成維持**
   - 全コードを`task-manager.html`に統合
   - 外部CSS/JSファイルは不可

3. **ブラウザ互換性**
   - Microsoft Edge 90+を主要ターゲット
   - Chrome/Braveでも動作（File System Access API要）

4. **セキュリティ制約**
   - CSP（Content Security Policy）準拠
   - XSS対策（ユーザー入力のサニタイズ）
   - eval()使用不可

### 推奨事項

1. **パフォーマンス**
   - ノード数100個まで快適動作
   - 描画処理は16ms以内（60fps維持）
   - メモリ使用量50MB以下

2. **アクセシビリティ**
   - キーボード操作対応
   - スクリーンリーダー対応（ARIA属性）
   - 色覚異常対応（色だけに依存しない）

3. **保守性**
   - コメント充実（JSDoc形式）
   - 関数は単一責任原則
   - マジックナンバー排除

---

## 将来拡張

### v2.12.0以降の候補機能

1. **自動レイアウトアルゴリズム**
   - 階層的レイアウト（Sugiyama法）
   - 力指向レイアウト（Fruchterman-Reingold法）
   - 円形レイアウト

2. **高度な編集機能**
   - 複数ノード一括選択
   - コピー&ペースト
   - Undo/Redo
   - スナップ機能（グリッド吸着）

3. **表示機能拡張**
   - ズーム/パン
   - ミニマップ
   - フィルタリング（ステータス別）
   - 検索機能

4. **インポート機能**
   - Mermaid記法からの変換
   - Graphviz DOT形式インポート
   - PlantUML形式インポート

5. **コラボレーション機能**
   - コメント機能
   - 変更履歴
   - バージョン管理

6. **印刷機能**
   - PDF出力
   - ページ分割印刷
   - 印刷プレビュー

---

## 付録

### サンプルフローチャート定義

別ファイル `flowchart-sample.json` を参照

### 参考資料

- SVG仕様: https://www.w3.org/TR/SVG2/
- File System Access API: https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API
- Canvas API: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API
- LocalStorage API: https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage

### 変更履歴

| バージョン | 日付 | 変更内容 | 著者 |
|-----------|------|---------|------|
| 1.0.0 | 2026-03-20 | 初版作成 | ks-source |

---

**ドキュメント作成者**: ks-source
**最終更新**: 2026-03-20
**ステータス**: 設計完了、実装待ち
