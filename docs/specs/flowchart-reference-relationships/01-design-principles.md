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

## 🔑 設計原則の明文化

**外部有識者レビュー（Opus, GPT-4, Gemini）を踏まえた設計原則**

### 1. 参照の意味と責務の分離

本機能における `references` は**図要素間の汎用的な関連先ID一覧**として扱う。

**現時点（v2.0）の定義**:
- `references` = 「この要素が関連する他の要素のID配列」（意味は汎用的）
- タスク連携情報（`taskFlowchartLinks`）は**別ドメイン情報として独立管理**
- エッジの `from`/`to` とは異なる意味を持つ（視覚的な接続 vs 意味的な関連）

**将来の拡張余地**:
- 参照の種類（relation type: `related`/`prerequisite`/`supplement`/`note`）の導入可能性
- 現時点では文字列配列 `["id1", "id2"]` だが、Phase 3以降でオブジェクト配列への拡張を検討

**重要**: タスク連携を `references` に吸収しない理由は、タスクはフローチャート要素と同列の「図要素」ではなく、別ドメインのエンティティであるため。

### 2. IDの安定性問題への対応

Mermaid由来のIDや自動生成IDは**実装都合の識別子**であり、業務上安定した識別子ではない。

**想定されるリスク**:
- Mermaidを再生成したら `node_002` が `node_003` になる
- 表示順や途中ノード追加でID採番がずれる
- 同じ見た目でも別SVGとして読込むとID対応が崩れる

**Phase 1-2の対策（MVP）**:
- 参照は `references: string[]` のまま（ID基準）
- `elements[id].displayText` と `originalLabel` を必ず保存
- SVG読込時に参照先IDが見つからない場合、**ラベルベースの再解決候補をユーザーに提示**
- 自動で勝手に修復せず、ユーザーに選択肢を見せる

**Phase 3以降の拡張案**:
```javascript
// ID以外のヒント情報を持たせる
{
  "targetId": "node_002",
  "targetHint": {
    "label": "アーキテクチャ設計",
    "type": "node"
  }
}
```

### 3. 正本（Source of Truth）の明確化

**データの正本の定義**:
- **タスク本体の正本**: A側（task-manager.html の LocalStorage）
- **SVG内の `taskFlowchartLinks`**: スナップショット（バックアップ）
- **フローチャート要素のメタデータ（`elements`, `manualNodes`）**: SVGが正本

**同期ルール**:
- **競合時**: A側のタスクデータを優先
- **SVG単独読込時**: スナップショットを復元に使用
- **SVG保存時**: 現在のA側データのスナップショットを含める

**実装方針**:
```javascript
// SVG保存時: タイムスタンプを付加
metadata.taskFlowchartLinks = {
  _snapshotAt: new Date().toISOString(),  // スナップショット時刻
  "0.1": { taskName: "要件定義", ... }
};

// SVG読込時: マージロジック
function restoreTaskLinks(extractedMetadata) {
  const svgTasks = extractedMetadata.taskFlowchartLinks || {};
  const localStorageTasks = getTasksFromLocalStorage();

  if (localStorageTasks && Object.keys(localStorageTasks).length > 0) {
    // A側のデータを優先（最新のはず）
    console.log('ℹ️ A側のタスクデータを優先して読み込みます');
    rebuildMockTasks(localStorageTasks);
  } else {
    // A側データがなければSVGのスナップショットを使用
    console.log('ℹ️ SVGのタスクスナップショットから復元します');
    rebuildMockTasks(svgTasks);
  }
}
```

### 4. 削除・改名・複製の意味論

要素のライフサイクル変更時の参照の扱いを明確化する。

**削除時の仕様**:
```javascript
function deleteElement(elementId) {
  const refCount = countReferencesToElement(elementId);
  if (refCount > 0) {
    if (!confirm(`この要素を削除すると ${refCount} 件の参照が失われます。削除しますか？`)) {
      return;
    }
  }
  // 参照は自動削除（孤立参照を残さない）
  cleanupReferencesToElement(elementId);
  // 要素本体の削除
}
```

**複製時の仕様**:
```javascript
function duplicateElement(elementId) {
  const newElement = { ...elements[elementId] };
  newElement.references = [];  // 参照は原則コピーしない
  // 理由: 参照先が変わる可能性があるため
  // 必要なら「参照も複製する」オプションをUI提供
}
```

**改名時の仕様**:
- ID参照であれば影響なし（IDは不変）
- ラベルベースで補助解決している場合は再評価が必要（Phase 3以降）

### 5. エッジ参照の扱い

**MVPでの方針**:
- **Phase 1-2**: 参照先を**ノードとメモノードに限定**
- **Phase 3以降**: エッジ参照を開放するか、デフォルト非表示にする

**理由**:
- エッジはユーザーが識別しづらい（ラベルが空のエッジも多い）
- Mermaid再生成でエッジIDが変わりやすい可能性がある
- UI候補一覧でノイズになりやすい

**実装例**:
```javascript
function getAllReferenceableIds() {
  // Phase 1-2: ノードとメモノードのみ
  return [
    ...Array.from(document.querySelectorAll('[data-et="node"], [data-et="cluster"]'))
      .map(el => el.getAttribute('data-id')),
    ...Object.keys(manualNodes)
    // エッジは含めない
  ];
}
```

---

## 🎨 Phase 4: ステータス色分け機能との統合

### 概要

参照関係管理機能と並行して、**ステータス色分け機能**（Phase 4）も v2.0 メタデータに統合される。

### データ構造

```javascript
// v2.0メタデータに追加されるフィールド
elementStatuses = {
  'flowchart-A': 'completed',       // 正規ノード
  'flowchart-B': 'in-progress',     // 正規ノード
  'subgraph_1': 'on-hold',          // サブグラフ
  'memo_node_1': 'postponed'        // メモノード
};
```

### ステータス定義

7種類のステータス（task-manager.htmlと完全同一の色定義）:
- `not-started` - 未着手（薄グレー）
- `in-progress` - 進行中（薄イエロー）
- `completed` - 完了（薄グリーン）
- `on-hold` - 保留（薄オレンジ）
- `postponed` - 延期（薄ピンク）
- `cancelled` - 中止（グレー）
- `unknown` - 不明（グレー）

### タスクステータスとの独立性の原則

**重要**: フローチャートステータスとタスクステータスは**完全に独立**している。

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

理由:
1. フローチャートとタスクは1:Nの関係（1つのノードに複数タスク）
2. フローチャートステータスは「工程全体」の状態を表す
3. タスクステータスは「個別作業」の状態を表す
4. データ同期による複雑性を回避
```

### 参照関係との関係

- `references` と `elementStatuses` は**独立したフィールド**
- 同一要素が両方の情報を持つことが可能
- **例**: F_01 が "completed" ステータスを持ち、かつ F_02 を参照する

```javascript
// 同一要素が両方の情報を持つ例
{
  "F_01": {
    "type": "node",
    "displayText": "要件定義",
    "memo": "プロジェクトキックオフで決定",
    "references": ["F_02", "MEMO_001"]  // 参照関係
  }
}

elementStatuses = {
  "F_01": "completed"  // ステータス（別フィールド）
};
```

---

## 📊 v2.0メタデータの全体像

### 完全なデータ構造図

```
┌─────────────────────────────────────────────────────────────┐
│ SVG Metadata v2.0                                           │
├─────────────────────────────────────────────────────────────┤
│ version: "2.0"                                              │
│ schemaVersion: "2.0"                                        │
│ exportedAt: "2026-03-22T12:00:00Z"                         │
│ svgFile: "workflow.svg"                                     │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 既存機能（v1.0から継続）                                 │ │
│ ├─────────────────────────────────────────────────────────┤ │
│ │ elements: { ... }          # ノード・エッジのメモ・ラベル │ │
│ │ originalLabels: { ... }    # 元のラベル情報              │ │
│ │ manualNodes: { ... }       # メモノード                  │ │
│ │ manualEdges: { ... }       # 手動エッジ                  │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ v2.0 新機能                                              │ │
│ ├─────────────────────────────────────────────────────────┤ │
│ │ elementStatuses: { ... }   # ステータス色分け（Phase 4） │ │
│ │ references: { ... }        # 参照関係管理                │ │
│ │ taskFlowchartLinks: { ... }# タスク連携スナップショット   │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### フィールド一覧

| フィールド | v1.0 | v2.0 | 説明 | 担当機能 |
|-----------|------|------|------|---------|
| `version` | ✅ | ✅ | メタデータバージョン | 共通 |
| `schemaVersion` | ❌ | ✅ | スキーマバージョン | 共通 |
| `exportedAt` | ✅ | ✅ | エクスポート日時 | 共通 |
| `svgFile` | ✅ | ✅ | 元のSVGファイル名 | 共通 |
| `elements` | ✅ | ✅ | 要素メタデータ | 既存 |
| `originalLabels` | ✅ | ✅ | 元のラベル | 既存 |
| `manualNodes` | ✅ | ✅ | メモノード | 既存 |
| `manualEdges` | ✅ | ✅ | 手動エッジ | 既存 |
| **`elementStatuses`** | **❌** | **✅** | **ステータス情報** | **Phase 4** |
| **`references`** | **❌** | **✅** | **参照関係** | **参照管理** |
| **`taskFlowchartLinks`** | **❌** | **✅** | **タスク連携** | **参照管理** |

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

## 📝 更新履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-22 | v1.2 | Phase 4（ステータス色分け機能）の統合<br>- Phase 4統合セクション追加<br>- v2.0メタデータの全体像図を追加<br>- フィールド一覧表を更新（elementStatuses追加）<br>- タスクステータスとの独立性を明記 |
| 2026-03-22 | v1.1 | 外部有識者レビュー（Opus, GPT-4, Gemini）を反映<br>- 設計原則の明文化セクション追加<br>- 参照の意味と責務の分離を明記<br>- IDの安定性問題への対応を追加<br>- 正本（Source of Truth）の明確化<br>- 削除・改名・複製の意味論を追加<br>- エッジ参照の扱いを明記 |
| 2026-03-22 | v1.0 | 初版作成（外部レビュー前） |

---

**最終更新日**: 2026-03-22
**現在バージョン**: v1.2
**提案レベル**: Phase 4統合完了、実装準備完了
