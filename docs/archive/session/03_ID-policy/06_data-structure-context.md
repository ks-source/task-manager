# データ構造とコンテキスト

**バージョン**: v2.13.4
**データ形式**: JSON
**永続化**: LocalStorage + File System Access API
**最終更新**: 2026-03-22

---

## 📋 概要

Task Manager のデータ構造を理解することで、WBS番号再利用ポリシーがデータ整合性に与える影響を評価できます。特に、参照フィールド（`predecessors`, `flowchart.nodeIds`）への影響が重要です。

---

## 🗂️ プロジェクトデータの全体構造

```json
{
  "meta": {
    "projectName": "新規プロジェクト",
    "version": "2.0",
    "createdAt": "2026-03-22T08:00:00.000Z",
    "lastModified": "2026-03-22T10:30:00.000Z"
  },
  "tasks": [
    {
      "wbs_no": "TEST.1.1",
      "phase": "TEST",
      "task_type": "開発",
      "task_name": "要件定義",
      "primary_owner": "山田太郎",
      "support_owner": "佐藤花子",
      "duration_bd": 5,
      "start_date": "2026-03-25",
      "finish_date": "2026-03-29",
      "status": "未着手",
      "priority": "高",
      "predecessors": "",
      "verification_method": "レビュー会議",
      "pass_criteria": "承認済み",
      "mermaid_ids": "node_1",
      "display_order": 0,
      "deleted": false,
      "deleted_at": null,
      "comments": [],
      "flowchart": {
        "nodeIds": ["node_1"],
        "memos": {"node_1": "詳細メモ"},
        "customLabels": {},
        "relatedManualEdges": []
      }
    }
  ]
}
```

---

## 🔍 タスクオブジェクトの詳細

### **基本フィールド**

| フィールド | 型 | 必須 | 説明 | 例 |
|----------|-----|------|------|-----|
| `wbs_no` | string | ✅ | **WBS番号（主キー）** | "TEST.15.1" |
| `phase` | string | ✅ | フェーズ名 | "TEST", "DESIGN", "RELEASE" |
| `task_type` | string | | タスク種別 | "開発", "レビュー", "テスト" |
| `task_name` | string | ✅ | タスク名 | "要件定義" |
| `primary_owner` | string | | 主担当者 | "山田太郎" |
| `support_owner` | string | | 支援担当者 | "佐藤花子" |
| `duration_bd` | number | | 工数（業務日） | 5 |
| `start_date` | string | | 開始日（ISO 8601） | "2026-03-25" |
| `finish_date` | string | | 終了日（ISO 8601） | "2026-03-29" |
| `status` | string | ✅ | ステータス | "未着手", "進行中", "完了" |
| `priority` | string | | 優先度 | "最高", "高", "中", "低" |
| `display_order` | number | | 表示順序 | 0, 1, 2, ... |

---

### **参照フィールド（重要）**

| フィールド | 型 | 説明 | 例 | WBS番号変更時の影響 |
|----------|-----|------|-----|-------------------|
| **`predecessors`** | string | 前提タスク（カンマ区切り） | "TEST.14.1,TEST.15.0" | ⚠️ 参照が壊れる |
| **`mermaid_ids`** | string | Mermaidノード ID | "node_1,node_2" | ✅ 影響なし |
| **`flowchart.nodeIds`** | array | フローチャートノード ID | ["node_1"] | ✅ 影響なし |

---

### **削除関連フィールド**

| フィールド | 型 | 説明 | 例 |
|----------|-----|------|-----|
| **`deleted`** | boolean | 削除フラグ | true, false |
| **`deleted_at`** | string / null | 削除日時（ISO 8601） | "2026-03-22T10:30:00.000Z" |

---

### **その他フィールド**

| フィールド | 型 | 説明 | 例 |
|----------|-----|------|-----|
| `verification_method` | string | 検証方法 | "レビュー会議" |
| `pass_criteria` | string | 合格基準 | "承認済み" |
| `comments` | array | コメント履歴 | `[{type: "status_change", ...}]` |
| `flowchart` | object | フローチャート連携データ | `{nodeIds: [...], memos: {...}}` |

---

## ⚠️ 参照整合性の課題

### **predecessorsフィールド（前提タスク）**

#### **データ構造**
```json
{
  "wbs_no": "TEST.16.1",
  "task_name": "設計",
  "predecessors": "TEST.15.1,TEST.14.1"
}
```

#### **選択肢1（ID不変）の場合**
```json
// TEST.15.1を削除
{
  "wbs_no": "TEST.15.1",
  "task_name": "要件定義",
  "deleted": true,
  "deleted_at": "2026-03-22T10:30:00.000Z"
}

// TEST.16.1の参照は壊れない（TEST.15.1は存在する）
{
  "wbs_no": "TEST.16.1",
  "task_name": "設計",
  "predecessors": "TEST.15.1,TEST.14.1"  // ✅ 参照有効
}
```

**処理時**:
```javascript
// predecessorsの解決
const predecessorIds = task.predecessors.split(',').map(id => id.trim());
predecessorIds.forEach(id => {
  const predecessorTask = projectData.tasks.find(t => t.wbs_no === id);
  if (predecessorTask) {
    if (predecessorTask.deleted) {
      console.warn(`Predecessor ${id} is deleted`);
      // UI表示: "TEST.15.1（削除済み）"
    } else {
      // 通常処理
    }
  }
});
```

---

#### **選択肢2（ID変更）の場合**

```json
// TEST.15.1を削除 → WBS番号変更
{
  "wbs_no": "TEST.15.1_del",  // ← 変更
  "task_name": "要件定義",
  "deleted": true,
  "deleted_at": "2026-03-22T10:30:00.000Z"
}

// TEST.16.1の参照を更新する必要あり
{
  "wbs_no": "TEST.16.1",
  "task_name": "設計",
  "predecessors": "TEST.15.1_del,TEST.14.1"  // ← 更新必須
}
```

**リスク**:
- ✅ 更新ロジックが正常動作すれば問題なし
- ❌ 更新漏れ → `TEST.15.1` が見つからない（存在しないタスク）
- ❌ 複雑な参照更新ロジックが必要

---

## 📊 データ例: 削除前後の変化

### **選択肢1（ID不変）の場合**

#### **削除前のJSON**
```json
{
  "tasks": [
    {
      "wbs_no": "TEST.14.1",
      "task_name": "企画",
      "deleted": false
    },
    {
      "wbs_no": "TEST.15.1",
      "task_name": "要件定義",
      "deleted": false
    },
    {
      "wbs_no": "TEST.16.1",
      "task_name": "設計",
      "predecessors": "TEST.15.1",
      "deleted": false
    }
  ]
}
```

#### **TEST.15.1削除後のJSON**
```json
{
  "tasks": [
    {
      "wbs_no": "TEST.14.1",
      "task_name": "企画",
      "deleted": false
    },
    {
      "wbs_no": "TEST.15.1",
      "task_name": "要件定義",
      "deleted": true,  // ← フラグのみ変更
      "deleted_at": "2026-03-22T10:30:00.000Z"
    },
    {
      "wbs_no": "TEST.16.1",
      "task_name": "設計",
      "predecessors": "TEST.15.1",  // ← 変更なし（参照有効）
      "deleted": false
    }
  ]
}
```

**メリット**:
- ✅ WBS番号不変
- ✅ predecessors参照が壊れない
- ✅ JSONエクスポート・インポートが単純

---

### **選択肢2（ID変更）の場合**

#### **TEST.15.1削除後のJSON**
```json
{
  "tasks": [
    {
      "wbs_no": "TEST.14.1",
      "task_name": "企画",
      "deleted": false
    },
    {
      "wbs_no": "TEST.15.1_del",  // ← WBS番号変更
      "task_name": "要件定義",
      "deleted": true,
      "deleted_at": "2026-03-22T10:30:00.000Z"
    },
    {
      "wbs_no": "TEST.16.1",
      "task_name": "設計",
      "predecessors": "TEST.15.1_del",  // ← 参照更新必須
      "deleted": false
    }
  ]
}
```

**デメリット**:
- ❌ predecessors更新ロジックが必要
- ❌ 更新漏れリスク
- ❌ JSONインポート時の処理が複雑

---

## 🔗 flowchartフィールドの参照

### **データ構造**
```json
{
  "flowchart": {
    "nodeIds": ["node_1", "node_2"],
    "memos": {
      "node_1": "詳細メモ",
      "node_2": "別のメモ"
    },
    "customLabels": {
      "node_1": "カスタムラベル"
    },
    "relatedManualEdges": ["edge_1"]
  }
}
```

**WBS番号変更の影響**:
- ✅ `flowchart.nodeIds` はMermaidノードIDを保持（WBS番号とは別）
- ✅ WBS番号が変更されても影響なし

---

## 📈 スケーラビリティへの影響

### **タスク数の増加**

| タスク数 | 選択肢1（ID不変） | 選択肢2（ID変更） |
|---------|----------------|-----------------|
| **100件** | パフォーマンス問題なし | 参照更新に0.5秒程度 |
| **1000件** | パフォーマンス問題なし | 参照更新に5秒程度（体感遅延） |
| **10000件** | LocalStorageサイズ制限に注意 | 参照更新に50秒程度（操作不可） |

**結論**: 選択肢2は大規模プロジェクトでパフォーマンス問題が発生

---

## 💾 JSONエクスポート・インポート

### **選択肢1（ID不変）のエクスポート**

```json
{
  "meta": {...},
  "tasks": [
    {"wbs_no": "TEST.1.1", "deleted": false},
    {"wbs_no": "TEST.2.1", "deleted": true, "deleted_at": "..."},
    {"wbs_no": "TEST.3.1", "deleted": false}
  ]
}
```

**インポート時の処理**:
```javascript
function importData(jsonData) {
  projectData = jsonData;  // そのまま読み込み
  saveToLocalStorage();
  renderTaskTable();  // 削除済みタスクは自動的にフィルタリング
}
```

**メリット**: シンプル、エラーが発生しにくい

---

### **選択肢2（ID変更）のエクスポート**

```json
{
  "meta": {...},
  "tasks": [
    {"wbs_no": "TEST.1.1", "deleted": false},
    {"wbs_no": "TEST.2.1_del", "deleted": true, "deleted_at": "..."},
    {"wbs_no": "TEST.3.1", "deleted": false, "predecessors": "TEST.2.1_del"}
  ]
}
```

**インポート時の処理**:
```javascript
function importData(jsonData) {
  // _delサフィックスの整合性チェック
  jsonData.tasks.forEach(task => {
    if (task.deleted && !task.wbs_no.endsWith('_del')) {
      console.warn(`Task ${task.wbs_no} is deleted but no _del suffix`);
      // マイグレーション: サフィックスを追加？
    }
  });

  projectData = jsonData;
  saveToLocalStorage();
  renderTaskTable();
}
```

**デメリット**: 複雑、エッジケースの処理が必要

---

## 🔍 検索・フィルタリングへの影響

### **WBS番号での検索**

#### **選択肢1（ID不変）**
```javascript
function searchByWbsNo(query) {
  return projectData.tasks.filter(task =>
    task.wbs_no.includes(query) && !task.deleted
  );
}

// ユーザー入力: "TEST.15"
// 結果: TEST.15.1（削除済み）は除外、TEST.15.2が表示
```

**メリット**: シンプル

---

#### **選択肢2（ID変更）**
```javascript
function searchByWbsNo(query) {
  const normalizedQuery = query.replace('_del', '');

  return projectData.tasks.filter(task => {
    const normalizedWbs = task.wbs_no.replace('_del', '');
    return normalizedWbs.includes(normalizedQuery) && !task.deleted;
  });
}

// ユーザー入力: "TEST.15"
// 結果: TEST.15.1_del（削除済み）もマッチ → 除外処理が必要
```

**デメリット**: 正規化処理が必要、複雑

---

## 📝 まとめ

### **データ整合性の観点から**

| 観点 | 選択肢1（ID不変） | 選択肢2（ID変更） |
|-----|----------------|-----------------|
| **predecessors参照** | ✅ 壊れない | ⚠️ 更新ロジック必要 |
| **JSONエクスポート** | ✅ シンプル | ❌ 複雑 |
| **JSONインポート** | ✅ シンプル | ❌ 検証・マイグレーション必要 |
| **検索機能** | ✅ シンプル | ❌ 正規化処理必要 |
| **パフォーマンス** | ✅ 問題なし | ⚠️ 大規模時に遅延 |

**総合評価**: **選択肢1（ID不変）がデータ整合性の観点から圧倒的に優れている**

---

### **推奨アプローチ**

**選択肢1（削除済みID除外 + 全履歴考慮）** を推奨

**理由**:
1. ✅ predecessors参照が壊れない
2. ✅ JSONエクスポート・インポートが単純
3. ✅ 検索・フィルタリングが単純
4. ✅ パフォーマンス問題なし
5. ✅ 実装コスト最小（1行の変更）

---

**完了**: 7つのドキュメントファイルをすべて作成しました

- [00_PROMPT_TO_AI.md](./00_PROMPT_TO_AI.md) - メインプロンプト
- [01_current-implementation.md](./01_current-implementation.md) - 現在の実装
- [02_problem-statement.md](./02_problem-statement.md) - 問題定義
- [03_option-1-exclude-only.md](./03_option-1-exclude-only.md) - 選択肢1詳細
- [04_option-2-suffix-approach.md](./04_option-2-suffix-approach.md) - 選択肢2詳細
- [05_industry-standards.md](./05_industry-standards.md) - 業界標準比較
- [06_data-structure-context.md](./06_data-structure-context.md) - データ構造（本ファイル）
