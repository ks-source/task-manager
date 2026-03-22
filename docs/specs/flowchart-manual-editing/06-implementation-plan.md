# 実装計画・工数見積もり

---
**feature**: フローチャート手動編集
**document**: 実装計画・工数見積もり
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

独立要素アーキテクチャに基づくメモノード・エッジ編集機能とAI解析用統合JSON出力の実装計画。

---

## アーキテクチャ原則

### 独立要素モデル（Independent Element Model）

```
┌─────────────────────────────────────────────┐
│  独立要素モデル (Independent Element Model) │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────┐      ┌──────────────┐   │
│  │ メモノード群  │      │ 手動エッジ群  │   │
│  │ manualNodes  │◄────►│ manualEdges  │   │
│  └──────────────┘ 参照  └──────────────┘   │
│         │           のみ         │          │
│         │                       │          │
│         ▼                       ▼          │
│  ドラッグ移動可能          端点編集可能      │
│  （エッジは不動）        （ノードは不動）    │
│                                             │
│  ※ 移動後は手動でエッジを再調整             │
└─────────────────────────────────────────────┘
```

### 設計原則

1. **ノードとエッジは独立**: 自動追従機能は実装しない
2. **メタデータで関係管理**: システムは自動更新せず、手動で記録
3. **段階的実装**: MVPから始め、早期フィードバック取得
4. **拡張性重視**: 将来的な機能追加に対応可能な設計

### スコープ外機能（明示的除外）

前回のノード吸着機能の教訓を踏まえ、以下は実装しない：
- ❌ ノード移動時のエッジ自動追従
- ❌ システム上の依存関係自動管理
- ❌ ノード-エッジ間の自動同期
- ❌ 複雑な座標変換（CTM）を伴う吸着機能

---

## 全体スケジュール

### タイムライン

```
Week 1-2: Phase 1（メモノード基本機能）
Week 2-3: Phase 2（エッジ編集機能）
Week 4-5: Phase 3-AI MVP（統合JSON出力）
Week 6:   ユーザー検証・AI解析テスト
Week 7+:  フィードバックを基に拡張判断
```

### フェーズ構成

| フェーズ | 機能 | 工数 | 依存関係 |
|---------|------|------|---------|
| **Phase 1** | メモノード作成・ドラッグ | 4-5h | なし |
| **Phase 2** | 手動エッジ端点編集 | 8-10h | Phase 1完了後 |
| **Phase 3-AI MVP** | 統合JSON出力 | 12-14h | Phase 1, 2完了後 |
| **Phase 3-AI 拡張** | 詳細情報追加 | +6-8h | MVP検証後（オプション） |
| **合計（MVP）** | | **24-29h (3-4日間)** | |
| **合計（完全版）** | | **30-37h (4-5日間)** | |

---

## Phase 1: メモノード基本機能（4-5h）

### 実装内容

#### 1. データ構造追加（30分）

```javascript
// グローバル変数追加
let manualNodes = {};        // メモノードのマップ
let nodeAddMode = false;     // ノード追加モードON/OFF
```

#### 2. ノード作成機能（1.5-2h）

**実装関数**:
- `toggleNodeAddMode()` - ノード追加モード切替
- `onSvgClickForNodeAdd()` - クリック位置にノード作成
- `createManualNode(x, y, label)` - SVG要素生成とデータ保存
- `drawNodeToSvg(node)` - SVG要素の描画

**SVG構造**:
```svg
<g class="manual-memo-node" data-node-id="manual-node-XXX" transform="translate(x, y)">
  <rect width="180" height="60" fill="#fffacd" stroke="#daa520" rx="4"/>
  <text x="10" y="35" class="manual-node-label">メモ内容</text>
</g>
```

#### 3. ドラッグ移動機能（1.5-2h）

**実装関数**:
- `attachDragBehavior(nodeId)` - ドラッグイベント設定
- mousedown/mousemove/mouseup イベントハンドラ
- `transform` 属性の動的更新

**注意**: エッジは追従しない（ユーザーに通知）

#### 4. データ永続化（30分）

**実装関数**:
- `buildMemoData()` に `manualNodes` フィールド追加
- `loadFromFile()` で `manualNodes` を復元・SVG再描画
- localStorage自動保存対応

#### 5. UI追加（30分）

```html
<button class="icon-button" onclick="toggleNodeAddMode()" id="node-add-btn">
  📝 メモノード
</button>
```

### 工数内訳

| タスク | 工数 |
|-------|------|
| データ構造設計・実装 | 0.5h |
| ノード作成機能 | 1.5-2h |
| ドラッグ移動機能 | 1.5-2h |
| データ永続化 | 0.5h |
| UI統合 | 0.5h |
| テスト・デバッグ | 0.5-1h |
| **合計** | **4-5h** |

---

## Phase 2: 手動エッジ端点編集（8-10h）

### 実装内容

#### 1. エッジ編集モード（3-4h）

**実装関数**:
- `attachEdgeEditBehavior(edgeId)` - エッジクリックイベント設定
- `enterEdgeEditMode(edgeId)` - 編集モード開始
- `exitEdgeEditMode()` - 編集モード終了
- `createHandle(x, y, type)` - 端点ハンドル作成

**SVG構造**:
```svg
<!-- 編集中のハンドル -->
<circle cx="100" cy="50" r="8" fill="cyan" stroke="#00aaff"
        class="edge-edit-handle edge-start-handle"/>
<circle cx="300" cy="200" r="8" fill="cyan" stroke="#00aaff"
        class="edge-edit-handle edge-end-handle"/>
```

#### 2. 端点ドラッグ機能（3-4h）

**実装関数**:
- `attachHandleDrag(handle, edgeId, endType)` - ハンドルドラッグ設定
- `updateEdgeLine(edgeId)` - エッジライン再描画
- `updateEdgeLabel(edgeId)` - ラベル位置再計算

**動作**:
- ハンドルドラッグ中、エッジライン/ラベルがリアルタイム更新
- ドラッグ終了時、`updatedAt` と `metadata.manuallyAdjusted` を更新

#### 3. エッジ削除機能（1h）

**実装関数**:
- `deleteManualEdge(edgeId)` - エッジ削除
- 編集モード中に Delete キー対応

#### 4. UI改善（1-2h）

- エッジホバー時のカーソル変更
- 編集モード中の視覚フィードバック
- ESCキーで編集モード終了

### 工数内訳

| タスク | 工数 |
|-------|------|
| エッジ編集モード実装 | 3-4h |
| 端点ドラッグ機能 | 3-4h |
| エッジ削除機能 | 1h |
| UI改善 | 1-2h |
| テスト・デバッグ | 1h |
| **合計** | **8-10h** |

---

## Phase 3-AI MVP: 統合JSON出力（12-14h）

### 実装内容

#### 1. Mermaidノード座標抽出（3-4h）

**実装関数**:
- `extractMermaidNodes()` - ノード情報抽出
- `getBBoxInSVGCoordinates()` - 既存関数を流用

**抽出データ**:
- id, bbox, center, label（original/custom/display）, memo

#### 2. Mermaidエッジ情報抽出（1-2h）

**実装関数**:
- `extractMermaidEdges()` - エッジ基本情報抽出（簡易版）

**MVPスコープ**:
- from/to nodeID のみ抽出
- path は簡略化（"M x1 y1 L x2 y2" 形式の近似）

#### 3. Mermaidクラスタ抽出（2-3h）

**実装関数**:
- `extractMermaidClusters()` - クラスタ情報抽出

**抽出データ**:
- id, bbox, label, childNodes配列

#### 4. 統合データ生成（2-3h）

**実装関数**:
- `extractIntegratedData()` - 全データ統合
- `extractManualNodes()` - manualNodes変換
- `extractManualEdges()` - manualEdges変換
- `extractConnections()` - 接続関係グラフ生成
- `calculateStatistics()` - 統計情報計算

#### 5. エクスポート機能（1-2h）

**実装関数**:
- `exportIntegratedJson()` - File System Access API使用

**UI**:
- エクスポートドロップダウンメニュー更新

#### 6. テスト・デバッグ（2h）

- 各種フローチャートでのテスト
- 大規模データでの動作確認
- AI（Claude等）での解析テスト

### 工数内訳

| タスク | 工数 |
|-------|------|
| Mermaidノード座標抽出 | 3-4h |
| Mermaidエッジ情報抽出（簡易版） | 1-2h |
| Mermaidクラスタ抽出 | 2-3h |
| 統合データ生成関数 | 2-3h |
| エクスポート機能実装 | 1-2h |
| UI統合 | 1h |
| テスト・デバッグ | 2h |
| **合計** | **12-14h** |

---

## Phase 3-AI 拡張: 詳細情報（+6-8h、オプション）

### 実装内容（MVP検証後に判断）

#### 1. Mermaidエッジpath詳細抽出（3-4h）

- SVG `<path>` 要素の `d` 属性解析
- ベジェ曲線制御点の抽出

#### 2. シェイプタイプ判別（2-3h）

- rect/circle/diamond の自動識別
- ヒューリスティックによる推定

#### 3. スタイル情報詳細化（2-3h）

- fill, stroke, stroke-width 等の抽出
- CSS computed style の取得

---

## データ構造設計

### manualNodes

```javascript
manualNodes = {
  "manual-node-1711012345678": {
    id: "manual-node-1711012345678",
    type: "memo",
    x: 500,
    y: 300,
    width: 180,
    height: 60,
    label: "メモ内容",
    style: {
      fill: "#fffacd",
      stroke: "#daa520",
      strokeWidth: 2
    },
    metadata: {
      memo: "補足説明",
      createdBy: "user"
    },
    references: [],  // ★ Phase 2: 参照関係管理（/docs/specs/flowchart-reference-relationships/ を参照）
    createdAt: "2026-03-21T10:30:45.678Z",
    updatedAt: "2026-03-21T10:35:12.123Z"
  }
};
```

### manualEdges（拡張版）

```javascript
manualEdges = {
  "manual-edge-1710000000000": {
    id: "manual-edge-1710000000000",
    startX: 100,
    startY: 50,
    startNodeId: "flowchart-A",
    endX: 300,
    endY: 200,
    endNodeId: "manual-node-1711012345678",
    label: "データフロー",
    style: {
      color: "#666",
      width: 2,
      dashArray: "5,5"
    },
    metadata: {
      conceptualStart: "PHASE1",
      conceptualEnd: "manual-node-1711012345678",
      memo: "Phase 1からメモノードへのデータ受け渡し",
      manuallyAdjusted: false      // ★Phase 2で追加
    },
    createdAt: "2026-03-21T10:25:40.000Z",
    updatedAt: "2026-03-21T10:25:40.000Z"  // ★Phase 2で更新
  }
};
```

---

## リスク管理

### 技術的リスク

| リスク | 影響度 | 対策 |
|-------|-------|------|
| Mermaid構造変更 | 中 | バージョン記録、信頼性レベル明示 |
| 大規模データ処理 | 低 | ファイルサイズ表示、gzip推奨 |
| ブラウザ互換性 | 低 | File System Access API の事前確認 |

### スケジュールリスク

| リスク | 影響度 | 対策 |
|-------|-------|------|
| 座標抽出の複雑性 | 中 | 既存関数（getBBoxInSVGCoordinates）流用 |
| エッジpath解析の難易度 | 高 | MVPでは簡略化、拡張版で対応 |
| UI/UX調整の時間超過 | 低 | 最小限のUIから開始 |

### 品質リスク

| リスク | 影響度 | 対策 |
|-------|-------|------|
| データ整合性 | 中 | バリデーション関数実装 |
| エクスポートファイル破損 | 低 | try-catch、エラーハンドリング |
| AI解析での不正確性 | 低 | データ信頼性レベル明示 |

---

## 成功基準

### Phase 1

- ✅ メモノードを任意の位置に作成できる
- ✅ ドラッグで移動できる
- ✅ JSONに保存・復元できる

### Phase 2

- ✅ エッジをクリックして編集モードに入れる
- ✅ 端点ハンドルをドラッグして座標変更できる
- ✅ エッジを削除できる

### Phase 3-AI MVP

- ✅ Mermaidノード座標を正確に抽出できる
- ✅ manualNodesとmanualEdgesを含む統合JSONを出力できる
- ✅ 出力したJSONをAI（Claude等）で解析し、有用な回答を得られる

### Phase 4: ステータス色分け機能

- ✅ 正規ノード・サブグラフ・メモノードにステータスを設定できる
- ✅ task-manager.htmlと同一の7種類のステータス色が適用される
- ✅ 右クリックメニューでステータス変更できる
- ✅ ステータス情報がSVGメタデータに保存・復元される

---

## Phase 4: ステータス色分け機能（6-9h）

### 概要

フローチャート要素（正規ノード・サブグラフ・メモノード）に対して、視覚的なステータス（完了・進行中・未着手等）を設定し、task-manager.htmlと同一の色で色分け表示する機能。

**重要な設計原則**:
- フローチャートステータスとタスクステータスは**完全に独立**
- フローチャートステータス = 視覚的な状態管理（SVGに保存）
- タスクステータス = 実タスク進捗管理（JSONに保存）
- 両者のデータ連携は行わない

---

### Phase 4-1: 基本実装（3-4h）

#### 実装内容

1. **ステータス色定義**（30分）
   - `STATUS_COLORS`定数をtask-manager.htmlからコピー
   - 7種類のステータス（未着手・進行中・完了・保留・延期・中止・不明）
   - 各ステータスの背景色・テキスト色・ボーダー色を定義

2. **データ構造追加**（30分）
   - グローバル変数`elementStatuses = {}`を追加
   - `buildMemoData()`に`elementStatuses`フィールドを追加
   - `exportSystemSvg()`のメタデータに`elementStatuses`を追加
   - バージョンを`"1.0"` → `"2.0"`に更新

3. **右クリックメニューUI**（1.5-2h）
   - コンテキストメニューHTML構造
   - CSS（白背景、影、ホバー効果）
   - メニュー表示ロジック（要素タイプ判定）
   - ステータス選択イベント

4. **正規ノードへのステータス適用**（1-1.5h）
   - `applyStatusToNode(nodeId, status)`関数
   - SVG要素の`fill`、`stroke`、`text`色を変更
   - `data-status`属性を設定

#### 成功基準

- ✅ 正規ノードを右クリックでステータスメニューが表示される
- ✅ ステータス選択でノードの色が変わる
- ✅ SVG保存・読み込みでステータスが復元される

---

### Phase 4-2: 拡張機能（2-3h）

#### 実装内容

1. **サブグラフへのステータス適用**（1h）
   - `applyStatusToSubgraph(clusterId, status)`関数
   - 背景色を30%透明化（視認性確保）
   - ボーダー色・ストローク幅の調整

2. **メモノードへのステータス適用**（30分）
   - `applyStatusToMemoNode(memoId, status)`関数
   - `.memo-content`の背景色・ボーダー色を変更

3. **ステータスクリア機能**（30分）
   - 右クリックメニューに「ステータスをクリア」項目を追加
   - `clearStatus(elementId)`関数
   - `elementStatuses`からキーを削除
   - デフォルト色に戻す

4. **ヘッダー凡例削除**（15分）
   - デモ用凡例（完了・進行中・未着手）を削除
   - ヘッダーをシンプル化

5. **マイグレーション処理**（30分）
   - `migrateMetadata(data)`関数
   - v1.0 → v2.0の自動マイグレーション
   - `elementStatuses`を空オブジェクトで初期化

#### 成功基準

- ✅ サブグラフとメモノードにもステータスを設定できる
- ✅ ステータスクリアで元の色に戻る
- ✅ v1.0ファイルをv2.0実装で開いても正常動作する

---

### Phase 4-3: UX改善（1-2h）

#### 実装内容

1. **ステータスアイコンバッジ**（45分）
   - ノード右上に小さいステータスバッジを表示
   - SVG `<circle>`要素でステータス色を表示
   - ホバー時にステータス名を表示

2. **フィルタリング機能拡張**（45分）
   - 既存の「完了済みを隠す」を拡張
   - 「ステータスでフィルタ」ドロップダウン
   - 複数ステータスの同時非表示

3. **一括ステータス設定**（オプション、30分）
   - 複数要素を選択してステータス一括設定
   - Ctrl+クリックで複数選択
   - 右クリックメニューで一括変更

#### 成功基準

- ✅ ステータスバッジで一目でステータスが分かる
- ✅ フィルタリングで特定ステータスを非表示にできる

---

### 実装順序

```
Phase 4-1: 基本実装（3-4h）
  ↓
ユーザー検証（正規ノードのみ）
  ↓
Phase 4-2: 拡張機能（2-3h）
  ↓
ユーザー検証（全要素対応）
  ↓
Phase 4-3: UX改善（1-2h）
  ↓
総合テスト・リリース
```

---

### 技術的考慮事項

#### 1. タスクステータスとの独立性

```
┌─────────────────────────┐         ┌─────────────────────────┐
│ flowchart-editor.html   │         │ task-manager.html       │
│                         │         │                         │
│ elementStatuses {       │         │ projectData.tasks [{    │
│   'F_01': 'completed'   │         │   wbs_no: "WBS1.1.0",   │
│ }                       │         │   status: "進行中"      │
│                         │ 🚫 連携なし │ }]                      │
│ 用途: 視覚的な状態管理    │         │ 用途: 実タスク進捗管理   │
│ 保存: SVGファイル         │         │ 保存: JSONファイル       │
└─────────────────────────┘         └─────────────────────────┘
```

**理由**:
1. フローチャートとタスクは1:Nの関係（1つのノードに複数タスク）
2. フローチャートステータスは「工程全体」の状態を表す
3. タスクステータスは「個別作業」の状態を表す
4. データ同期による複雑性を回避

#### 2. ステータス色の統一

- task-manager.htmlの`STATUS_COLORS`定義をそのままコピー
- 視覚的一貫性を確保
- 7種類すべてサポート

#### 3. 既存機能との整合性

- カスタムラベル・メモと同じデータ構造パターン
- 右クリックメニューの既存UIパターン踏襲
- `buildMemoData()`への追加のみ（既存コード影響最小化）

#### 4. バージョン管理

- メタデータバージョンを`"1.0"` → `"2.0"`に更新
- v1.0ファイルの自動マイグレーション
- 後方互換性の確保

---

### データフロー

```javascript
// 1. ユーザー操作
要素を右クリック
  ↓
メニューでステータス選択（例: "完了"）
  ↓

// 2. 内部処理
elementStatuses['F_01'] = 'completed';
  ↓
applyStatusToNode('F_01', 'completed');
  ↓
SVG要素のfill/stroke/text色を変更
  ↓

// 3. 保存
Ctrl+S または自動保存
  ↓
buildMemoData()でelementStatusesを含める
  ↓
exportSystemSvg()でSVGコメントに埋め込み
  ↓

// 4. 復元
SVGファイルを読み込み
  ↓
メタデータからelementStatusesを抽出
  ↓
restoreStatusesFromMetadata()でスタイル適用
```

---

### コード例

#### 右クリックメニューHTML

```html
<div id="status-context-menu" class="status-context-menu" style="display: none;">
  <div class="status-menu-header">ステータス設定</div>
  <div class="status-menu-item" data-status="not-started">
    <span class="status-dot" style="background: #6c757d;"></span>
    <span>未着手</span>
  </div>
  <div class="status-menu-item" data-status="in-progress">
    <span class="status-dot" style="background: #ffc107;"></span>
    <span>進行中</span>
  </div>
  <div class="status-menu-item" data-status="completed">
    <span class="status-dot" style="background: #28a745;"></span>
    <span>完了</span>
  </div>
  <div class="status-menu-item" data-status="on-hold">
    <span class="status-dot" style="background: #fd7e14;"></span>
    <span>保留</span>
  </div>
  <div class="status-menu-item" data-status="postponed">
    <span class="status-dot" style="background: #dc3545;"></span>
    <span>延期</span>
  </div>
  <div class="status-menu-item" data-status="cancelled">
    <span class="status-dot" style="background: #999;"></span>
    <span>中止</span>
  </div>
  <div class="status-menu-item" data-status="unknown">
    <span class="status-dot" style="background: #adb5bd;"></span>
    <span>不明</span>
  </div>
  <div class="status-menu-divider"></div>
  <div class="status-menu-item status-clear">
    <span>✗ ステータスをクリア</span>
  </div>
</div>
```

#### イベントハンドラ

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

// メニュー項目クリック
document.querySelectorAll('.status-menu-item').forEach(item => {
  item.addEventListener('click', () => {
    const status = item.getAttribute('data-status');
    const elementId = getCurrentContextMenuTarget();

    if (status) {
      setElementStatus(elementId, status);
    } else if (item.classList.contains('status-clear')) {
      clearElementStatus(elementId);
    }

    hideStatusContextMenu();
  });
});
```

---

### リスク管理

| リスク | 影響度 | 対策 |
|-------|-------|------|
| 別セッションのスキーマ修正と競合 | 高 | `elementStatuses`をトップレベルフィールドとして独立配置 |
| バージョン番号の不整合 | 中 | 別セッション完了後にバージョン調整 |
| 既存要素への影響 | 低 | ステータス未設定時は既存色を維持 |
| パフォーマンス（大量要素） | 低 | 100要素以下では問題なし |

---

### 工数見積もり

| フェーズ | 内容 | 工数 |
|---------|------|------|
| Phase 4-1 | 基本実装（正規ノード） | 3-4h |
| Phase 4-2 | 拡張（サブグラフ・メモノード・マイグレーション） | 2-3h |
| Phase 4-3 | UX改善（バッジ・フィルタリング） | 1-2h |
| **合計** | | **6-9h** |

---

## 次のアクション

### 即座に実施

1. ✅ 仕様書のレビュー（本ドキュメント）
2. → Phase 1実装開始

### Phase 1完了後

1. Phase 1のユーザー検証
2. Phase 2実装開始

### Phase 2完了後

1. Phase 2のユーザー検証
2. Phase 3-AI MVP実装開始

### Phase 3-AI MVP完了後

1. 実際のAI解析テスト
2. ユーザーフィードバック収集
3. 拡張版実装の要否判断

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要
- [01-overview.md](./01-overview.md) - 全体設計・方針
- [02-manual-edge.md](./02-manual-edge.md) - 手動エッジ仕様
- [04-ai-integrated-export.md](./04-ai-integrated-export.md) - AI統合JSON出力仕様
- [05-data-structure.md](./05-data-structure.md) - データ構造定義

---

**最終更新**: 2026-03-22
**ステータス**: Phase 4仕様追加完了
**総工数**:
- Phase 1-3（MVP）: 24-29h
- Phase 4（ステータス色分け）: 6-9h
- **合計**: 30-38h
