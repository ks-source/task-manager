# 提案する参照関係の設計

## 🎯 設計コンセプト

**「視覚的な矢印に依存せず、構造的に要素間の関連性を記録する」**

### 基本方針
1. **明示的なメタデータ管理**: 参照関係を `references` フィールドとして記録
2. **全要素に共通**: 正規ノード、メモノード、エッジすべてに適用可能
3. **一方向参照**: A → B を設定しても、B → A は自動設定しない
4. **人間による入力**: 自動吸着の代わりに、UI経由で明示的に設定
5. **SVGメタデータ化**: 保存・復元により永続化

---

## 📊 データ構造の設計

### 1. 参照関係（references）フィールド

すべての要素に `references` フィールドを追加します。

```javascript
// 正規ノードの例
elements["U_START"] = {
  type: "node",
  displayText: "[USR_01] ユーザー",
  memo: "初期状態のノード",
  customLabel: "[USR_01] ユーザー：問い合わせ発生",
  originalLabel: "[USR_01] ユーザー",
  references: ["MEMO_001", "node_002"]  // ★新規追加
};

// メモノードの例
manualNodes["MEMO_001"] = {
  id: "MEMO_001",
  label: "重要な注意事項",
  x: 100,
  y: 200,
  width: 120,
  height: 60,
  references: ["U_START", "node_002"]  // ★新規追加
};

// 手動エッジの例（任意）
manualEdges["E_MANUAL_001"] = {
  id: "E_MANUAL_001",
  from: "MEMO_001",
  to: "U_START",
  label: "",
  x1: 100, y1: 200,
  x2: 300, y2: 250,
  controlX: 200, controlY: 225,
  references: ["U_START"]  // ★新規追加（任意、from/toと重複する場合は不要）
};
```

### 2. タスク連携のメタデータ化

現在メモリ上のみの `mockTasks` をSVGメタデータに含めます。

```javascript
// SVGメタデータに追加
metadata.taskFlowchartLinks = {
  "0.1": {
    taskName: "要件定義",
    flowchartLinks: {
      nodes: ["U_START", "node_002"],
      edges: []
    }
  },
  "0.2": {
    taskName: "アーキテクチャ設計",
    flowchartLinks: {
      nodes: ["node_003", "node_004"],
      edges: ["E_001"]
    }
  }
};
```

### 3. メタデータ全体構造（v2.0）

```javascript
const metadata = {
  version: "2.0",  // ★バージョンアップ（v1.0 → v2.0）
  exportedAt: "2026-03-22T12:00:00.000Z",
  svgFile: "workflow.svg",

  // ★新規追加: タスクのフローチャート連携
  taskFlowchartLinks: {
    "0.1": { taskName: "要件定義", flowchartLinks: { nodes: [...], edges: [...] } },
    "0.2": { taskName: "設計", flowchartLinks: { nodes: [...], edges: [...] } }
  },

  // 既存: フローチャート要素のメタ情報（referencesを追加）
  elements: {
    "U_START": {
      type: "node",
      displayText: "[USR_01] ユーザー",
      memo: "初期状態のノード",
      customLabel: "",
      originalLabel: "[USR_01] ユーザー",
      references: ["MEMO_001", "node_002"]  // ★新規追加
    }
  },

  // 既存: メモノード（referencesを追加）
  manualNodes: {
    "MEMO_001": {
      id: "MEMO_001",
      label: "重要な注意事項",
      x: 100, y: 200,
      width: 120, height: 60,
      references: ["U_START", "node_002"]  // ★新規追加
    }
  },

  // 既存: 手動エッジ（referencesは任意）
  manualEdges: {
    "E_MANUAL_001": { /* ... */ }
  },

  // 既存: 元のラベル
  originalLabels: { /* ... */ }
};
```

---

## 🔄 データフローの改善

### Before（現状）

```
┌─────────────────────────────────────────────────┐
│ B側（flowchart-editor.html）                     │
├─────────────────────────────────────────────────┤
│ メモリ上のみ:                                   │
│ - mockTasks（タスク連携）                       │
│ - 参照関係は記録されない                        │
│                                                 │
│ SVG保存時:                                      │
│ - elementMemos, elementCustomLabels             │
│ - manualNodes, manualEdges                      │
│ ❌ mockTasks は保存されない                     │
│ ❌ 参照関係も保存されない                       │
└─────────────────────────────────────────────────┘
        │ SVG再読込
        ↓
┌─────────────────────────────────────────────────┐
│ ✅ メモ・ラベルは復元される                     │
│ ❌ タスク連携は失われる（A側から再取得が必要） │
│ ❌ 参照関係は復元されない                       │
└─────────────────────────────────────────────────┘
```

### After（提案）

```
┌─────────────────────────────────────────────────┐
│ B側（flowchart-editor.html）                     │
├─────────────────────────────────────────────────┤
│ メモリ上:                                       │
│ - mockTasks（タスク連携）                       │
│ - elements[id].references（参照関係）           │
│ - manualNodes[id].references                    │
│                                                 │
│ SVG保存時（v2.0メタデータ）:                    │
│ ✅ taskFlowchartLinks                           │
│ ✅ elements[id].references                      │
│ ✅ manualNodes[id].references                   │
│ ✅ 既存のメモ・ラベル・手動要素                 │
└─────────────────────────────────────────────────┘
        │ SVG再読込
        ↓
┌─────────────────────────────────────────────────┐
│ ✅ すべてのメタデータが復元される               │
│ ✅ タスク連携も復元される                       │
│ ✅ 参照関係も復元される                         │
│ → A側のデータがなくても単独で動作可能          │
└─────────────────────────────────────────────────┘
```

---

## 💡 設計の利点

### 1. 永続化による情報の保全
- **問題**: SVG再読込時にタスク連携・参照関係が失われる
- **解決**: SVGメタデータとして保存することで永続化

### 2. 単独動作の実現
- **問題**: A側（task-manager.html）のデータに依存
- **解決**: B側のSVGだけで完結、A側なしでも動作可能

### 3. AI解析の準備
- **問題**: 視覚的な矢印だけでは意味的な関係性を伝えられない
- **解決**: 構造化されたメタデータとして参照関係を記録

### 4. 人間による明示的な管理
- **問題**: ノード吸着の自動化は技術的に困難
- **解決**: UI経由で人間が明示的に設定、意図が明確

### 5. 拡張性の確保
- **将来**: 参照の種類（"関連", "補足", "前提"）を追加可能
- **将来**: 逆参照の自動表示、グラフビジュアライザー等

---

## 🎨 参照関係の表現方法

### パターン1: 文字列配列（推奨）

```javascript
references: ["U_START", "node_002", "MEMO_003"]
```

**利点**:
- シンプルで理解しやすい
- JSONサイズが小さい
- 実装が容易

**欠点**:
- 参照の意味（種類）を表現できない
- 追加情報（コメント等）を付与できない

### パターン2: オブジェクト配列（将来拡張用）

```javascript
references: [
  { id: "U_START", type: "prerequisite", note: "前提条件" },
  { id: "node_002", type: "related", note: "関連ノード" },
  { id: "MEMO_003", type: "supplement", note: "補足情報" }
]
```

**利点**:
- 参照の意味を明示できる
- 追加情報を付与できる
- 将来の拡張性が高い

**欠点**:
- 実装が複雑
- JSONサイズが大きくなる
- UI設計が複雑化

**推奨**: Phase 1では文字列配列、Phase 3でオブジェクト配列に拡張

---

## 🔗 参照関係の意味論

### 一方向参照の方針

**Q: A → B を設定した時、B → A も自動設定するか？**

**A: 自動設定しない**

**理由**:
1. **意図の明確化**: ユーザーが明示的に設定した関係のみを記録
2. **参照の方向性**: "A が B を参照する" と "B が A を参照する" は異なる意味
3. **データの整合性**: 自動双方向化はデータの肥大化と管理複雑化を招く

**例**:
```
MEMO_001: "U_STARTノードの注意事項"
  → references: ["U_START"]  // MEMO_001がU_STARTを参照

U_START:
  → references: []  // 自動的にMEMO_001を含めない
```

### 循環参照の扱い

**Q: A → B → A のような循環参照を許可するか？**

**A: 許可する**

**理由**:
1. **参照は依存関係ではない**: 相互に関連する要素は実際に存在する
2. **グラフ理論的に妥当**: DAG（有向非巡回グラフ）である必要はない
3. **検出のみ実施**: 循環参照を検出して警告表示、エラーにはしない

**実装**:
```javascript
function detectCircularReferences(elementId, visited = new Set()) {
  if (visited.has(elementId)) {
    console.warn('⚠️ 循環参照を検出:', Array.from(visited), '→', elementId);
    return true;
  }

  visited.add(elementId);
  const refs = getReferences(elementId);

  for (const refId of refs) {
    if (detectCircularReferences(refId, new Set(visited))) {
      return true;
    }
  }

  return false;
}
```

---

## 🧩 具体的な利用シーン

### シーン1: メモノードの補足説明

```
正規ノード: U_START - [USR_01] ユーザー
メモノード: MEMO_001 - "初期状態に関する重要な注意事項"
参照関係: MEMO_001.references = ["U_START"]
```

**意味**: MEMO_001は U_START に関する補足説明である

### シーン2: 複数ノードへの共通メモ

```
正規ノード: U_START, node_002, node_003
メモノード: MEMO_002 - "共通エラーハンドリング"
参照関係: MEMO_002.references = ["U_START", "node_002", "node_003"]
```

**意味**: MEMO_002は3つのノードに共通する情報である

### シーン3: タスクとフローチャートの連携

```
タスク: 0.2 - "Difyワークフローアーキテクチャ設計"
フローチャート要素: U_START, node_002
参照関係: taskFlowchartLinks["0.2"].flowchartLinks.nodes = ["U_START", "node_002"]
```

**意味**: タスク0.2の成果物にU_STARTとnode_002が含まれる

### シーン4: 正規ノード間の関連性

```
正規ノード: node_002 - [SYS_01] システム
正規ノード: node_003 - [SYS_02] データベース
参照関係: node_002.references = ["node_003"]
```

**意味**: node_002の実装にはnode_003の理解が前提となる

---

## 🎯 設計原則のまとめ

### 1. KISS原則（Keep It Simple, Stupid）
- 文字列配列で参照関係を表現（最小限の複雑性）
- 一方向参照のみ（双方向は手動で設定）

### 2. 明示性の原則
- 自動推測ではなく、人間が明示的に設定
- 意図が明確、誤解が生じない

### 3. 拡張性の原則
- 将来のオブジェクト配列化に対応可能
- 参照の種類（type）を追加可能

### 4. 永続性の原則
- SVGメタデータとして保存
- リロード時に完全復元

### 5. 単一責任の原則
- `references`: 要素間の関連性のみを表現
- 視覚的な矢印（manualEdges）とは独立

---

## 📝 TypeScript型定義（参考）

```typescript
// 参照関係の型（Phase 1: 文字列配列）
type ElementReferences = string[];

// 参照関係の型（Phase 3: オブジェクト配列）
type ElementReferencesAdvanced = Array<{
  id: string;
  type?: "related" | "prerequisite" | "supplement" | "note";
  note?: string;
}>;

// 要素メタデータ
interface ElementMetadata {
  type: string;
  displayText: string;
  memo: string;
  customLabel: string;
  originalLabel: string;
  references: ElementReferences;  // ★新規追加
}

// メモノード
interface ManualNode {
  id: string;
  label: string;
  x: number;
  y: number;
  width: number;
  height: number;
  references: ElementReferences;  // ★新規追加
}

// タスク連携
interface TaskFlowchartLink {
  taskName: string;
  flowchartLinks: {
    nodes: string[];
    edges: string[];
  };
}

// SVGメタデータ v2.0
interface SVGMetadataV2 {
  version: "2.0";
  exportedAt: string;
  svgFile: string;
  elements: { [id: string]: ElementMetadata };
  manualNodes: { [id: string]: ManualNode };
  manualEdges: { [id: string]: any };
  taskFlowchartLinks: { [wbs: string]: TaskFlowchartLink };  // ★新規追加
  originalLabels: { [id: string]: string };
}
```

---

**作成日**: 2026-03-22
**バージョン**: 1.0
**提案レベル**: 設計確定前（外部レビュー待ち）
