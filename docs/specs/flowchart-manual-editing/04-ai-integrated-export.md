# AI解析用統合JSON出力仕様

---
**feature**: フローチャート手動編集
**document**: AI解析用統合JSON出力
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

Mermaid生成SVGフローチャートと手動追加要素（メモノード・エッジ）を統合した、AI解析最適化JSONデータの出力仕様。

### 背景

外部AIにフローチャート全体を包括的に解析してもらうユースケースに対応するため、以下を統合的に出力する：
- Mermaidオリジナルノード・エッジの座標とメタデータ
- 各要素に追加されたメモとカスタムラベル
- 手動追加したメモノードの座標とメタデータ
- 手動追加したエッジの座標とメタデータ

### 目的

1. **AI解析の容易化**: フロー最適化提案、ボトルネック分析、ドキュメント自動生成
2. **データの一元化**: SVGビジュアルとメタデータを1ファイルで提供
3. **外部ツール連携**: PlantUML変換、他形式エクスポート等への活用
4. **長期的な資産化**: AI技術進化に伴う新ユースケースへの対応

---

## ユースケース

### 1. フロー最適化提案

```
ユーザー → AI:
"この統合JSONを解析して、ボトルネックと改善提案をください"

AIの分析例:
- ノードA→B間の距離が長い（座標から算出）
- メモに"要承認"とあるが承認フローが不明確
- Phase2に手動メモノード3個 → 複雑度が高い可能性
```

### 2. 自然言語での質問応答

```
Q: "Phase 1からPhase 3への最短経路は？"
A: [座標とエッジ情報から経路を計算して回答]

Q: "メモが多いノードはどこ？"
A: [メモ内容と座標を統合解析して回答]
```

### 3. 他形式への自動変換

```
統合JSON → AIプロンプト:
"このフローチャートをPlantUML形式に変換してください"

AI出力:
@startuml
start
:Phase 1開始;
note right: プロジェクトキックオフ会議で決定
...
@enduml
```

### 4. ドキュメント自動生成

```
統合JSON → AI:
"このフローをマークダウン形式の仕様書にしてください"

AI出力:
# プロジェクトワークフロー仕様
## Phase 1: 開始 (座標: x=160, y=80)
プロジェクトキックオフ会議で決定
...
```

---

## データ構造

### 統合JSON構造（完全版）

```json
{
  "meta": {
    "exportType": "ai-analysis-integrated",
    "version": "1.0",
    "flowchartId": "project-workflow",
    "svgFile": "workflow.svg",
    "exportedAt": "2026-03-21T12:00:00Z",
    "description": "Comprehensive flowchart data for AI analysis"
  },

  "viewport": {
    "viewBox": "0 0 1200 800",
    "width": 1200,
    "height": 800
  },

  "mermaidNodes": [
    {
      "id": "flowchart-A",
      "type": "node",
      "bbox": { "x": 100, "y": 50, "width": 120, "height": 60 },
      "center": { "x": 160, "y": 80 },
      "label": {
        "original": "開始",
        "custom": "Phase 1開始",
        "display": "Phase 1開始"
      },
      "memo": "プロジェクトキックオフ会議で決定",
      "hasMemo": true,
      "hasCustomLabel": true
    }
  ],

  "mermaidEdges": [
    {
      "id": "edge-A-to-B",
      "from": "flowchart-A",
      "to": "flowchart-B",
      "approximatePath": "M 160 140 L 300 200",
      "label": ""
    }
  ],

  "mermaidClusters": [
    {
      "id": "cluster-SCOPE",
      "type": "cluster",
      "bbox": { "x": 500, "y": 100, "width": 600, "height": 400 },
      "label": "SCOPE",
      "childNodes": ["PHASE1", "PHASE2", "PHASE3"]
    }
  ],

  "manualNodes": [
    {
      "id": "manual-node-1711012345678",
      "type": "memo",
      "bbox": { "x": 500, "y": 300, "width": 180, "height": 60 },
      "center": { "x": 590, "y": 330 },
      "label": "メモ内容",
      "metadata": {
        "connectedEdges": ["manual-edge-XXX"],
        "memo": "補足説明",
        "createdBy": "user"
      },
      "createdAt": "2026-03-21T10:30:45.678Z",
      "updatedAt": "2026-03-21T10:35:12.123Z"
    }
  ],

  "manualEdges": [
    {
      "id": "manual-edge-1710000000000",
      "from": { "nodeId": "flowchart-A", "x": 220, "y": 80 },
      "to": { "nodeId": "manual-node-1711012345678", "x": 500, "y": 330 },
      "label": "データフロー",
      "metadata": { "manuallyAdjusted": false },
      "createdAt": "2026-03-21T10:25:40.000Z",
      "updatedAt": "2026-03-21T10:25:40.000Z"
    }
  ],

  "connections": [
    {
      "from": "flowchart-A",
      "to": "flowchart-B",
      "type": "mermaid-original",
      "label": ""
    },
    {
      "from": "flowchart-A",
      "to": "manual-node-1711012345678",
      "type": "manual",
      "label": "データフロー"
    }
  ],

  "statistics": {
    "totalNodes": 5,
    "mermaidNodes": 3,
    "manualNodes": 2,
    "totalEdges": 4,
    "mermaidEdges": 2,
    "manualEdges": 2,
    "nodesWithMemo": 3,
    "nodesWithCustomLabel": 1,
    "totalClusters": 1
  }
}
```

### フィールド説明

#### meta

| フィールド | 型 | 説明 |
|-----------|---|------|
| `exportType` | string | 出力タイプ識別子（固定値: "ai-analysis-integrated"） |
| `version` | string | データ構造バージョン |
| `flowchartId` | string | フローチャートID |
| `svgFile` | string | 元のSVGファイル名 |
| `exportedAt` | string | エクスポート日時（ISO 8601） |
| `description` | string | データの説明 |

#### viewport

| フィールド | 型 | 説明 |
|-----------|---|------|
| `viewBox` | string | SVG viewBox属性値 |
| `width` | number | SVG幅 |
| `height` | number | SVG高さ |

#### mermaidNodes

| フィールド | 型 | 説明 |
|-----------|---|------|
| `id` | string | ノードID（data-id属性値） |
| `type` | string | 要素タイプ（"node", "cluster"） |
| `bbox` | object | 境界ボックス（x, y, width, height） |
| `center` | object | 中心座標（x, y） |
| `label.original` | string | 元のラベル |
| `label.custom` | string | カスタムラベル（なければ空文字） |
| `label.display` | string | 表示ラベル（customが優先） |
| `memo` | string | メモ内容 |
| `hasMemo` | boolean | メモの有無 |
| `hasCustomLabel` | boolean | カスタムラベルの有無 |

#### mermaidEdges

| フィールド | 型 | 説明 |
|-----------|---|------|
| `id` | string | エッジID（推定） |
| `from` | string | 開始ノードID |
| `to` | string | 終了ノードID |
| `approximatePath` | string | 近似パス（MVPでは簡略化） |
| `label` | string | エッジラベル |

#### mermaidClusters

| フィールド | 型 | 説明 |
|-----------|---|------|
| `id` | string | クラスタID |
| `type` | string | "cluster" 固定 |
| `bbox` | object | 境界ボックス |
| `label` | string | クラスタラベル |
| `childNodes` | array | 子ノードIDの配列 |

#### manualNodes

| フィールド | 型 | 説明 |
|-----------|---|------|
| `id` | string | ノードID |
| `type` | string | ノードタイプ（"memo"等） |
| `bbox` | object | 境界ボックス |
| `center` | object | 中心座標 |
| `label` | string | ラベル/メモ内容 |
| `metadata` | object | メタデータ（connectedEdges等） |
| `createdAt` | string | 作成日時 |
| `updatedAt` | string | 更新日時 |

#### manualEdges

| フィールド | 型 | 説明 |
|-----------|---|------|
| `id` | string | エッジID |
| `from.nodeId` | string\|null | 開始ノードID |
| `from.x` | number | 開始点X座標 |
| `from.y` | number | 開始点Y座標 |
| `to.nodeId` | string\|null | 終了ノードID |
| `to.x` | number | 終了点X座標 |
| `to.y` | number | 終了点Y座標 |
| `label` | string | エッジラベル |
| `metadata` | object | メタデータ |
| `createdAt` | string | 作成日時 |
| `updatedAt` | string | 更新日時 |

#### connections

接続関係の簡易表現（グラフ解析用）

| フィールド | 型 | 説明 |
|-----------|---|------|
| `from` | string | 開始ノードID |
| `to` | string | 終了ノードID |
| `type` | string | "mermaid-original" または "manual" |
| `label` | string | エッジラベル |

---

## 実装方針

### MVP版（Phase 3-AI MVP: 12-14h）

#### 実装スコープ

**含める**:
- ✅ Mermaidノード（id, bbox, center, label, memo）
- ✅ manualNodes全情報
- ✅ manualEdges全情報
- ✅ 基本的な接続関係（from-to）
- ✅ 統計情報
- ✅ viewport情報

**除外**（将来拡張で対応）:
- ❌ Mermaidエッジの詳細path（"approximatePath"は簡略化）
- ❌ シェイプタイプ自動判別（すべて"node"）
- ❌ 詳細スタイル情報（色、太さ等）

### 関数設計

#### 1. extractIntegratedData()

```javascript
/**
 * AI解析用の統合JSONデータを抽出
 * @returns {Object} 統合データオブジェクト
 */
function extractIntegratedData() {
  const svg = getSvgElement();
  const viewBox = svg.getAttribute('viewBox') || '0 0 1200 800';
  const [vbX, vbY, vbWidth, vbHeight] = viewBox.split(' ').map(Number);

  return {
    meta: {
      exportType: "ai-analysis-integrated",
      version: "1.0",
      flowchartId: currentSvgFileName?.replace('.svg', '') || 'untitled',
      svgFile: currentSvgFileName,
      exportedAt: new Date().toISOString(),
      description: "Comprehensive flowchart data for AI analysis"
    },

    viewport: {
      viewBox: viewBox,
      width: vbWidth,
      height: vbHeight
    },

    mermaidNodes: extractMermaidNodes(),
    mermaidEdges: extractMermaidEdges(),
    mermaidClusters: extractMermaidClusters(),
    manualNodes: extractManualNodes(),
    manualEdges: extractManualEdges(),
    connections: extractConnections(),
    statistics: calculateStatistics()
  };
}
```

#### 2. extractMermaidNodes()

```javascript
/**
 * Mermaidノードの座標とメタデータを抽出
 * @returns {Array} ノード配列
 */
function extractMermaidNodes() {
  const nodes = [];
  const svg = getSvgElement();
  const nodeElements = svg.querySelectorAll('[data-et="node"]:not([data-id^="manual-"])');

  nodeElements.forEach(element => {
    const id = element.getAttribute('data-id') || element.id;
    const bbox = getBBoxInSVGCoordinates(element);
    const centerX = bbox.x + bbox.width / 2;
    const centerY = bbox.y + bbox.height / 2;

    const originalLabel = originalLabels[id] || '';
    const customLabel = elementCustomLabels[id] || '';
    const displayLabel = customLabel || originalLabel;
    const memo = elementMemos[id] || '';

    nodes.push({
      id: id,
      type: "node",
      bbox: {
        x: Math.round(bbox.x * 10) / 10,
        y: Math.round(bbox.y * 10) / 10,
        width: Math.round(bbox.width * 10) / 10,
        height: Math.round(bbox.height * 10) / 10
      },
      center: {
        x: Math.round(centerX * 10) / 10,
        y: Math.round(centerY * 10) / 10
      },
      label: {
        original: originalLabel,
        custom: customLabel,
        display: displayLabel
      },
      memo: memo,
      hasMemo: !!memo,
      hasCustomLabel: !!customLabel
    });
  });

  return nodes;
}
```

#### 3. extractManualEdges()

```javascript
/**
 * 手動エッジを統合JSON形式に変換
 * @returns {Array} エッジ配列
 */
function extractManualEdges() {
  const edges = [];

  for (const edge of Object.values(manualEdges)) {
    edges.push({
      id: edge.id,
      from: {
        nodeId: edge.startNodeId,
        x: Math.round(edge.startX * 10) / 10,
        y: Math.round(edge.startY * 10) / 10
      },
      to: {
        nodeId: edge.endNodeId,
        x: Math.round(edge.endX * 10) / 10,
        y: Math.round(edge.endY * 10) / 10
      },
      label: edge.label || '',
      metadata: edge.metadata || {},
      createdAt: edge.createdAt,
      updatedAt: edge.updatedAt || edge.createdAt
    });
  }

  return edges;
}
```

#### 4. exportIntegratedJson()

```javascript
/**
 * 統合JSONをファイルとしてエクスポート
 */
async function exportIntegratedJson() {
  try {
    if (!('showSaveFilePicker' in window)) {
      alert('このブラウザはFile System Access APIに対応していません。');
      return;
    }

    const data = extractIntegratedData();
    const jsonStr = JSON.stringify(data, null, 2);

    const suggestedName = `${data.meta.flowchartId}_ai_integrated.json`;

    const handle = await window.showSaveFilePicker({
      suggestedName: suggestedName,
      types: [{
        description: 'JSON Files',
        accept: { 'application/json': ['.json'] }
      }]
    });

    const writable = await handle.createWritable();
    await writable.write(jsonStr);
    await writable.close();

    const sizeKB = Math.round(jsonStr.length / 1024);
    showNotification(`AI解析用JSONをエクスポートしました (${sizeKB}KB)`);
    console.log('AI解析用JSONエクスポート完了:', handle.name);

  } catch (err) {
    if (err.name !== 'AbortError') {
      console.error('エクスポートエラー:', err);
      alert('エクスポートに失敗しました: ' + err.message);
    }
  }
}
```

### UI統合

```html
<!-- エクスポートドロップダウンメニュー -->
<div class="dropdown" id="export-dropdown">
  <button class="icon-button" onclick="toggleDropdown('export-dropdown')">
    📤 出力
  </button>
  <div class="dropdown-menu">
    <button class="dropdown-item" onclick="exportEditedSvg(); closeDropdown('export-dropdown');">
      SVG出力（ビジュアル共有用）
    </button>
    <button class="dropdown-item" onclick="saveToFileAs(); closeDropdown('export-dropdown');">
      JSONメモ保存（セッション復元用）
    </button>
    <button class="dropdown-item" onclick="exportIntegratedJson(); closeDropdown('export-dropdown');">
      AI解析用JSON（総合データ）★新規
    </button>
  </div>
</div>
```

---

## 工数見積もり

### MVP版（Phase 3-AI MVP）

| タスク | 工数 | 優先度 |
|-------|------|--------|
| extractMermaidNodes() 実装 | 3-4h | 高 |
| extractMermaidEdges() 簡易版 | 1-2h | 中 |
| extractMermaidClusters() 実装 | 2-3h | 中 |
| extractManualNodes() 実装 | 1h | 高 |
| extractManualEdges() 実装 | 1h | 高 |
| extractConnections() 実装 | 2h | 中 |
| exportIntegratedJson() 実装 | 1h | 高 |
| UI統合（ドロップダウンメニュー） | 1-2h | 高 |
| テスト・デバッグ | 2h | 高 |
| **合計** | **12-14h** | |

### 拡張版（将来実装）

| タスク | 工数 |
|-------|------|
| Mermaidエッジpath詳細抽出 | 3-4h |
| シェイプタイプ判別 | 2-3h |
| スタイル情報詳細化 | 2-3h |
| **合計** | **+6-8h** |

---

## ROI分析

### 投資

- 実装工数: 12-14h（MVP版）
- 保守コスト: 低（JSON構造明確、拡張容易）

### リターン

#### 短期（0-3ヶ月）
- AI活用による作業効率化（フロー分析時間短縮: 2h/月）
- ドキュメント自動生成による工数削減（3h/月）

#### 中期（3-12ヶ月）
- 他プロジェクトへの横展開
- 外部ツール連携による機能拡張

#### 長期（12ヶ月以上）
- AI技術進化による新ユースケース開拓
- 競合ツールとの差別化要素

### 投資回収期間

```
工数投資: 14h
削減効果: 5h/月（分析2h + ドキュメント3h）
投資回収期間: 約3ヶ月
```

---

## リスクと対策

### リスク1: データサイズ肥大化

**想定**: 大規模フローチャート（100ノード）で 500KB-1MB

**対策**:
- エクスポート時にファイルサイズを表示
- gzip圧縮を推奨するメッセージ表示
- 将来的に軽量版オプション提供

### リスク2: Mermaid内部構造への依存

**対策**:
- データ信頼性レベルをmetaに明記
- バージョン情報を記録
- 近似値であることを明示

### リスク3: UI複雑化

**対策**:
- ドロップダウンメニューで整理
- 各出力形式の用途を明示
- ツールチップでヘルプ表示

---

## 段階的実装計画

```
Phase 1: メモノード実装 (4-5h)
  ↓
Phase 2: エッジ編集 (8-10h)
  ↓
Phase 3-AI MVP: 統合JSON (12-14h) ← 本仕様
  ↓
ユーザーフィードバック・AI解析検証
  ↓
Phase 3-AI 拡張: 詳細情報 (+6-8h)
```

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要
- [01-overview.md](./01-overview.md) - 全体設計
- [05-data-structure.md](./05-data-structure.md) - データ構造定義
- [06-implementation-plan.md](./06-implementation-plan.md) - 実装計画

---

**最終更新**: 2026-03-21
**ステータス**: 仕様確定、実装待ち
**推奨度**: 85/100点（実装を強く推奨）
