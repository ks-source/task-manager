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
      connectedEdges: [],      // 手動管理（自動更新しない）
      memo: "補足説明",
      createdBy: "user"
    },
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

**最終更新**: 2026-03-21
**ステータス**: 仕様確定、Phase 1実装待ち
**総工数**: 24-29h（MVP）/ 30-37h（完全版）
