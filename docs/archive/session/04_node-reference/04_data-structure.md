# データ構造定義

## 📊 TypeScript型定義

### 基本型

```typescript
/**
 * 参照関係の型（Phase 1: 文字列配列）
 */
type ElementReferences = string[];

/**
 * 参照関係の型（Phase 3: オブジェクト配列、将来拡張用）
 */
type ElementReferencesAdvanced = Array<{
  id: string;
  type?: "related" | "prerequisite" | "supplement" | "note";
  note?: string;
}>;

/**
 * 要素タイプ
 */
type ElementType = "node" | "edge" | "cluster" | "unknown";

/**
 * タスクステータス
 */
type TaskStatus = "Not Started" | "In Progress" | "Completed" | "On Hold" | "Cancelled";
```

### 要素メタデータ

```typescript
/**
 * フローチャート要素のメタデータ
 */
interface ElementMetadata {
  /** 要素タイプ（node, edge, cluster等） */
  type: ElementType;

  /** SVG上の表示テキスト */
  displayText: string;

  /** ユーザーが入力したメモ */
  memo: string;

  /** カスタムラベル（SVGラベルの上書き） */
  customLabel: string;

  /** 元のラベル（SVG読込時の値、復元用） */
  originalLabel: string;

  /** ★新規: この要素が参照している他の要素のID配列 */
  references: ElementReferences;
}

/**
 * 要素メタデータのマップ
 * キー: 要素ID（data-id属性の値）
 */
type ElementMetadataMap = { [elementId: string]: ElementMetadata };
```

### メモノード（手動追加ノード）

```typescript
/**
 * メモノード（手動で追加された注釈ノード）
 */
interface ManualNode {
  /** ノードID（例: MEMO_001） */
  id: string;

  /** ノードのラベル（表示テキスト） */
  label: string;

  /** X座標（SVG座標系） */
  x: number;

  /** Y座標（SVG座標系） */
  y: number;

  /** ノードの幅 */
  width: number;

  /** ノードの高さ */
  height: number;

  /** ★新規: このメモノードが参照している要素のID配列 */
  references: ElementReferences;
}

/**
 * メモノードのマップ
 * キー: ノードID
 */
type ManualNodeMap = { [nodeId: string]: ManualNode };
```

### 手動エッジ

```typescript
/**
 * 手動エッジ（手動で追加されたエッジ）
 */
interface ManualEdge {
  /** エッジID（例: E_MANUAL_001） */
  id: string;

  /** 開始ノードID */
  from: string;

  /** 終了ノードID */
  to: string;

  /** エッジのラベル */
  label: string;

  /** 開始点X座標 */
  x1: number;

  /** 開始点Y座標 */
  y1: number;

  /** 終了点X座標 */
  x2: number;

  /** 終了点Y座標 */
  y2: number;

  /** 制御点X座標（ベジェ曲線用） */
  controlX: number;

  /** 制御点Y座標（ベジェ曲線用） */
  controlY: number;

  /** ★新規（任意）: このエッジが参照している要素のID配列 */
  references?: ElementReferences;
}

/**
 * 手動エッジのマップ
 * キー: エッジID
 */
type ManualEdgeMap = { [edgeId: string]: ManualEdge };
```

### タスク連携

```typescript
/**
 * タスクとフローチャートの連携情報
 */
interface TaskFlowchartLink {
  /** タスク名 */
  taskName: string;

  /** フローチャート要素との連携 */
  flowchartLinks: {
    /** 連携しているノードIDの配列 */
    nodes: string[];

    /** 連携しているエッジIDの配列 */
    edges: string[];
  };
}

/**
 * タスク連携のマップ
 * キー: WBS番号（例: "0.1", "0.2"）
 */
type TaskFlowchartLinkMap = { [wbs: string]: TaskFlowchartLink };
```

### SVGメタデータ（v2.0）

```typescript
/**
 * SVGメタデータ v2.0
 * SVGコメント内にJSON形式で埋め込まれる
 */
interface SVGMetadataV2 {
  /** メタデータバージョン（v1.0 → v2.0にアップグレード） */
  version: "2.0";

  /** エクスポート日時（ISO 8601形式） */
  exportedAt: string;

  /** 元のSVGファイル名 */
  svgFile: string;

  /** フローチャート要素のメタデータ */
  elements: ElementMetadataMap;

  /** メモノード（手動追加） */
  manualNodes: ManualNodeMap;

  /** 手動エッジ */
  manualEdges: ManualEdgeMap;

  /** ★新規: タスクとフローチャートの連携情報 */
  taskFlowchartLinks: TaskFlowchartLinkMap;

  /** 元のラベル情報（復元用） */
  originalLabels: { [elementId: string]: string };
}
```

---

## 📋 JSONサンプル

### 完全なメタデータ例

```json
{
  "version": "2.0",
  "exportedAt": "2026-03-22T12:00:00.000Z",
  "svgFile": "Dify_Workflow.svg",

  "elements": {
    "U_START": {
      "type": "node",
      "displayText": "[USR_01] ユーザー",
      "memo": "初期状態のノード\n問い合わせが発生する起点",
      "customLabel": "[USR_01] ユーザー：問い合わせ発生",
      "originalLabel": "[USR_01] ユーザー",
      "references": ["MEMO_001", "node_002"]
    },
    "node_002": {
      "type": "node",
      "displayText": "[SYS_01] システム",
      "memo": "",
      "customLabel": "",
      "originalLabel": "[SYS_01] システム",
      "references": []
    },
    "node_003": {
      "type": "node",
      "displayText": "[DB_01] データベース",
      "memo": "マスターデータを参照",
      "customLabel": "",
      "originalLabel": "[DB_01] データベース",
      "references": ["node_002"]
    },
    "E_001": {
      "type": "edge",
      "displayText": "問い合わせ送信",
      "memo": "",
      "customLabel": "",
      "originalLabel": "問い合わせ送信",
      "references": []
    }
  },

  "manualNodes": {
    "MEMO_001": {
      "id": "MEMO_001",
      "label": "重要な注意事項",
      "x": 100,
      "y": 200,
      "width": 120,
      "height": 60,
      "references": ["U_START", "node_002"]
    },
    "MEMO_002": {
      "id": "MEMO_002",
      "label": "共通エラーハンドリング",
      "x": 300,
      "y": 150,
      "width": 150,
      "height": 80,
      "references": ["U_START", "node_002", "node_003"]
    }
  },

  "manualEdges": {
    "E_MANUAL_001": {
      "id": "E_MANUAL_001",
      "from": "MEMO_001",
      "to": "U_START",
      "label": "補足説明",
      "x1": 100,
      "y1": 200,
      "x2": 300,
      "y2": 250,
      "controlX": 200,
      "controlY": 225
    },
    "E_MANUAL_002": {
      "id": "E_MANUAL_002",
      "from": "MEMO_001",
      "to": "node_002",
      "label": "",
      "x1": 220,
      "y1": 200,
      "x2": 400,
      "y2": 300,
      "controlX": 310,
      "controlY": 250
    }
  },

  "taskFlowchartLinks": {
    "0.1": {
      "taskName": "要件定義",
      "flowchartLinks": {
        "nodes": ["U_START", "node_002"],
        "edges": ["E_001"]
      }
    },
    "0.2": {
      "taskName": "Difyワークフローアーキテクチャ設計",
      "flowchartLinks": {
        "nodes": ["U_START", "node_002", "node_003"],
        "edges": []
      }
    },
    "1.1": {
      "taskName": "データベース設計",
      "flowchartLinks": {
        "nodes": ["node_003"],
        "edges": []
      }
    }
  },

  "originalLabels": {
    "U_START": "[USR_01] ユーザー",
    "node_002": "[SYS_01] システム",
    "node_003": "[DB_01] データベース",
    "E_001": "問い合わせ送信"
  }
}
```

---

## 🔄 データ変換ロジック

### メモリ上のデータ構造（JavaScript）

```javascript
// flowchart-editor.html内のグローバル変数

// 要素のメモ（単純なマップ）
let elementMemos = {
  "U_START": "初期状態のノード\n問い合わせが発生する起点",
  "node_003": "マスターデータを参照"
};

// 要素のカスタムラベル（単純なマップ）
let elementCustomLabels = {
  "U_START": "[USR_01] ユーザー：問い合わせ発生"
};

// ★新規: 要素の参照関係（単純なマップ）
let elementReferences = {
  "U_START": ["MEMO_001", "node_002"],
  "node_003": ["node_002"]
};

// メモノード（オブジェクトのマップ）
let manualNodes = {
  "MEMO_001": {
    id: "MEMO_001",
    label: "重要な注意事項",
    x: 100, y: 200,
    width: 120, height: 60,
    references: ["U_START", "node_002"]  // ★新規
  }
};

// タスク連携（配列）
let mockTasks = [
  {
    wbs: "0.1",
    taskName: "要件定義",
    mermaidIds: "U_START, node_002",
    flowchartLinks: {
      nodes: ["U_START", "node_002"],
      edges: ["E_001"]
    }
  },
  {
    wbs: "0.2",
    taskName: "Difyワークフローアーキテクチャ設計",
    mermaidIds: "U_START, node_002, node_003",
    flowchartLinks: {
      nodes: ["U_START", "node_002", "node_003"],
      edges: []
    }
  }
];
```

### メモリ → SVGメタデータ変換

```javascript
/**
 * メモリ上のデータをSVGメタデータ形式に変換
 */
function buildMetadata() {
  const metadata = {
    version: "2.0",
    exportedAt: new Date().toISOString(),
    svgFile: currentSvgFileName || 'unknown.svg',
    elements: {},
    manualNodes: manualNodes,  // そのまま
    manualEdges: manualEdges,  // そのまま
    taskFlowchartLinks: {},    // ★新規
    originalLabels: originalLabels
  };

  // elements を構築
  const allElementIds = new Set([
    ...Object.keys(elementMemos),
    ...Object.keys(elementCustomLabels),
    ...Object.keys(elementReferences)  // ★新規
  ]);

  for (const elementId of allElementIds) {
    const element = document.querySelector(`[data-id="${elementId}"]`);
    if (!element) continue;

    metadata.elements[elementId] = {
      type: element.getAttribute('data-et') || 'unknown',
      displayText: getElementDisplayText(element),
      memo: elementMemos[elementId] || '',
      customLabel: elementCustomLabels[elementId] || '',
      originalLabel: originalLabels[elementId] || '',
      references: elementReferences[elementId] || []  // ★新規
    };
  }

  // taskFlowchartLinks を構築（mockTasks配列をオブジェクトに変換）
  mockTasks.forEach(task => {
    metadata.taskFlowchartLinks[task.wbs] = {
      taskName: task.taskName,
      flowchartLinks: {
        nodes: task.flowchartLinks.nodes,
        edges: task.flowchartLinks.edges
      }
    };
  });

  return metadata;
}
```

### SVGメタデータ → メモリ変換

```javascript
/**
 * SVGメタデータをメモリ上のデータに復元
 */
function restoreMetadata(extractedMetadata) {
  if (!extractedMetadata) return;

  // 元のラベル情報を復元
  if (extractedMetadata.originalLabels) {
    originalLabels = extractedMetadata.originalLabels;
  }

  // メモ・カスタムラベル・参照関係を復元
  if (extractedMetadata.elements) {
    elementMemos = {};
    elementCustomLabels = {};
    elementReferences = {};  // ★新規

    for (const [elementId, elementData] of Object.entries(extractedMetadata.elements)) {
      if (elementData.memo) {
        elementMemos[elementId] = elementData.memo;
      }
      if (elementData.customLabel) {
        elementCustomLabels[elementId] = elementData.customLabel;
        // SVG DOMに即座に反映
        const element = document.querySelector(`[data-id="${elementId}"]`);
        if (element) {
          updateSvgLabel(element, elementData.customLabel);
        }
      }
      if (elementData.references) {  // ★新規
        elementReferences[elementId] = elementData.references;
      }
    }
  }

  // メモノードを復元
  if (extractedMetadata.manualNodes) {
    manualNodes = extractedMetadata.manualNodes;
    // イベントハンドラを再アタッチ
    for (const nodeId of Object.keys(manualNodes)) {
      attachDragBehavior(nodeId);
    }
  }

  // 手動エッジを復元
  if (extractedMetadata.manualEdges) {
    manualEdges = extractedMetadata.manualEdges;
  }

  // タスク連携を復元（オブジェクト → 配列変換）  // ★新規
  if (extractedMetadata.taskFlowchartLinks) {
    mockTasks = [];
    for (const [wbs, linkData] of Object.entries(extractedMetadata.taskFlowchartLinks)) {
      mockTasks.push({
        wbs: wbs,
        taskName: linkData.taskName,
        mermaidIds: linkData.flowchartLinks.nodes.join(', '),
        flowchartLinks: linkData.flowchartLinks,
        status: "Not Started"  // デフォルト値（A側から再取得時に上書き）
      });
    }
  }
}
```

---

## 🔍 データ検証ロジック

### 参照の一貫性チェック

```javascript
/**
 * 参照の一貫性をチェック（存在しないIDへの参照を検出）
 */
function validateReferences(metadata) {
  const allIds = new Set([
    ...Object.keys(metadata.elements),
    ...Object.keys(metadata.manualNodes),
    ...Object.keys(metadata.manualEdges)
  ]);

  const errors = [];

  // elements の参照をチェック
  for (const [elementId, elementData] of Object.entries(metadata.elements)) {
    if (!elementData.references) continue;

    for (const refId of elementData.references) {
      if (!allIds.has(refId)) {
        errors.push({
          elementId: elementId,
          invalidRef: refId,
          type: 'element'
        });
      }
    }
  }

  // manualNodes の参照をチェック
  for (const [nodeId, nodeData] of Object.entries(metadata.manualNodes)) {
    if (!nodeData.references) continue;

    for (const refId of nodeData.references) {
      if (!allIds.has(refId)) {
        errors.push({
          elementId: nodeId,
          invalidRef: refId,
          type: 'manualNode'
        });
      }
    }
  }

  return errors;
}

/**
 * 検証エラーをクリーンアップ（無効な参照を削除）
 */
function cleanupInvalidReferences(metadata, errors) {
  errors.forEach(error => {
    if (error.type === 'element') {
      const refs = metadata.elements[error.elementId].references;
      metadata.elements[error.elementId].references =
        refs.filter(id => id !== error.invalidRef);

      console.warn(`⚠️ 無効な参照を削除: ${error.elementId} → ${error.invalidRef}`);
    } else if (error.type === 'manualNode') {
      const refs = metadata.manualNodes[error.elementId].references;
      metadata.manualNodes[error.elementId].references =
        refs.filter(id => id !== error.invalidRef);

      console.warn(`⚠️ 無効な参照を削除: ${error.elementId} → ${error.invalidRef}`);
    }
  });
}
```

### 循環参照の検出

```javascript
/**
 * 循環参照を検出（警告のみ、エラーにはしない）
 */
function detectCircularReferences(elementId, metadata, visited = new Set(), path = []) {
  if (visited.has(elementId)) {
    console.warn('⚠️ 循環参照を検出:', [...path, elementId].join(' → '));
    return true;
  }

  visited.add(elementId);
  path.push(elementId);

  const refs = getReferences(elementId, metadata);

  for (const refId of refs) {
    if (detectCircularReferences(refId, metadata, new Set(visited), [...path])) {
      return true;
    }
  }

  return false;
}

/**
 * 要素の参照リストを取得
 */
function getReferences(elementId, metadata) {
  if (metadata.elements[elementId]) {
    return metadata.elements[elementId].references || [];
  }
  if (metadata.manualNodes[elementId]) {
    return metadata.manualNodes[elementId].references || [];
  }
  return [];
}
```

---

## 📏 データサイズ見積もり

### 典型的なプロジェクトでのメタデータサイズ

```
フローチャート規模: 50ノード、70エッジ
メモノード: 10個
タスク: 20個

JSON サイズ見積もり:
- elements (50ノード + 70エッジ): 約 30KB
  （1要素あたり約250バイト）
- manualNodes (10個): 約 2KB
- manualEdges (15本): 約 3KB
- taskFlowchartLinks (20タスク): 約 5KB
- originalLabels: 約 10KB

合計: 約 50KB（圧縮前）

SVGファイル全体: 約 200-300KB
```

**結論**: SVGファイルサイズへの影響は軽微（+15-20%程度）

---

**作成日**: 2026-03-22
**バージョン**: 1.0
**メタデータバージョン**: v2.0
