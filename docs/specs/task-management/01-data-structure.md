# タスクデータ構造 - 完全版仕様

---

**feature**: タスク管理
**document**: データ構造定義
**version**: 2.0
**status**: 実装済み（v2.13.4時点）
**created**: 2026-03-22
**updated**: 2026-03-22

---

## 📖 概要

Task Managerのタスクデータ構造を定義します。プロジェクトデータはJSON形式で管理され、LocalStorageに保存されます。この文書は、全フィールドの詳細定義、データバージョン管理、JSON Schema等を含む完全版仕様です。

---

## 🗂️ プロジェクトデータの全体構造

### トップレベル構造

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

## 📦 metaオブジェクト

### フィールド定義

| フィールド | 型 | 必須 | 説明 | 例 |
|----------|-----|------|------|-----|
| `projectName` | string | ✅ | プロジェクト名 | "新規プロジェクト" |
| `version` | string | ✅ | データバージョン | "2.0" |
| `createdAt` | string | ✅ | 作成日時（ISO 8601） | "2026-03-22T08:00:00.000Z" |
| `lastModified` | string | ✅ | 最終更新日時（ISO 8601） | "2026-03-22T10:30:00.000Z" |

### JSON Schema

```json
{
  "meta": {
    "type": "object",
    "required": ["projectName", "version", "createdAt", "lastModified"],
    "properties": {
      "projectName": { "type": "string", "minLength": 1 },
      "version": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
      "createdAt": { "type": "string", "format": "date-time" },
      "lastModified": { "type": "string", "format": "date-time" }
    }
  }
}
```

---

## 📋 tasksオブジェクト（配列）

### 基本フィールド

| フィールド | 型 | 必須 | 説明 | 例 | 実装バージョン |
|----------|-----|------|------|-----|---------------|
| `wbs_no` | string | ✅ | **WBS番号（主キー）** | "TEST.15.1" | v1.0.0 |
| `phase` | string | ✅ | フェーズ名 | "TEST", "DESIGN" | v1.0.0 |
| `task_type` | string | | タスク種別 | "開発", "レビュー" | v1.0.0 |
| `task_name` | string | ✅ | タスク名 | "要件定義" | v1.0.0 |
| `primary_owner` | string | | 主担当者 | "山田太郎" | v1.0.0 |
| `support_owner` | string | | 支援担当者 | "佐藤花子" | v1.0.0 |
| `duration_bd` | number | | 工数（業務日） | 5 | v1.0.0 |
| `start_date` | string | | 開始日（YYYY-MM-DD） | "2026-03-25" | v1.0.0 |
| `finish_date` | string | | 終了日（YYYY-MM-DD） | "2026-03-29" | v1.0.0 |
| `status` | string | ✅ | ステータス | "未着手", "進行中" | v1.0.0 |
| `priority` | string | | 優先度 | "最高", "高", "中", "低" | v1.0.0 |
| `display_order` | number | | 表示順序 | 0, 1, 2, ... | v1.0.0 |

---

### 参照フィールド

| フィールド | 型 | 説明 | 例 | WBS番号変更時の影響 | 実装バージョン |
|----------|-----|------|-----|--------------------|--------------
| `predecessors` | string | 前提タスク（カンマ区切り） | "TEST.14.1,TEST.15.0" | ⚠️ 参照が壊れる | v1.0.0 |
| `mermaid_ids` | string | Mermaidノード ID（カンマ区切り） | "node_1,node_2" | ✅ 影響なし | v2.0.0 |

#### predecessorsフィールドの詳細

**フォーマット**: カンマ区切りのWBS番号リスト

```json
{
  "wbs_no": "TEST.16.1",
  "predecessors": "TEST.14.1,TEST.15.1"
}
```

**処理例**:
```javascript
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

**ID不変性との関係**:
- WBS番号が不変であれば、predecessors参照は壊れない
- 削除済みタスクへの参照も有効（論理削除のため）

詳細: [03-deletion-policy.md](./03-deletion-policy.md)

---

### 削除関連フィールド

| フィールド | 型 | 必須 | 説明 | 例 | 実装バージョン |
|----------|-----|------|------|-----|--------------
| `deleted` | boolean | ✅ | 削除フラグ | true, false | v2.13.0 |
| `deleted_at` | string / null | | 削除日時（ISO 8601） | "2026-03-22T10:30:00.000Z" | v2.13.0 |

**論理削除の実装**:
```json
{
  "wbs_no": "TEST.15.1",
  "task_name": "要件定義",
  "deleted": true,
  "deleted_at": "2026-03-22T10:30:00.000Z",
  // その他フィールドはそのまま保持
}
```

詳細: [03-deletion-policy.md](./03-deletion-policy.md)

---

### 検証関連フィールド

| フィールド | 型 | 説明 | 例 | 実装バージョン |
|----------|-----|------|-----|--------------
| `verification_method` | string | 検証方法 | "レビュー会議" | v1.0.0 |
| `pass_criteria` | string | 合格基準 | "承認済み" | v1.0.0 |

---

### コメントフィールド

| フィールド | 型 | 説明 | 例 | 実装バージョン |
|----------|-----|------|-----|--------------
| `comments` | array | コメント履歴 | `[{type: "status_change", ...}]` | v1.5.0 |

**構造例**:
```json
{
  "comments": [
    {
      "type": "status_change",
      "from": "未着手",
      "to": "進行中",
      "timestamp": "2026-03-22T09:00:00.000Z",
      "user": "山田太郎"
    },
    {
      "type": "comment",
      "text": "進捗確認: 50%完了",
      "timestamp": "2026-03-22T10:00:00.000Z",
      "user": "佐藤花子"
    }
  ]
}
```

---

### フローチャート連携フィールド

| フィールド | 型 | 説明 | 実装バージョン |
|----------|-----|------|--------------
| `flowchart` | object | フローチャート連携データ | v2.15.0 |

#### flowchartオブジェクトの詳細構造

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

| サブフィールド | 型 | 説明 | 例 |
|-------------|-----|------|-----|
| `nodeIds` | array | フローチャートノード IDリスト | ["node_1", "node_2"] |
| `memos` | object | ノードIDをキーとしたメモ | {"node_1": "詳細メモ"} |
| `customLabels` | object | ノードIDをキーとしたカスタムラベル | {"node_1": "ラベル"} |
| `relatedManualEdges` | array | 関連する手動エッジIDリスト | ["edge_1"] |

**WBS番号変更の影響**:
- ✅ `flowchart.nodeIds` はMermaidノードIDを保持（WBS番号とは別）
- ✅ WBS番号が変更されても影響なし

詳細: [../data-integration/](../data-integration/)

---

## 📊 ステータス・優先度の定義

### statusフィールド

| 値 | 説明 | UI表示色（例） |
|----|------|---------------|
| `未着手` | タスク未開始 | グレー |
| `進行中` | タスク実行中 | 青 |
| `完了` | タスク完了 | 緑 |
| `保留` | 一時停止 | オレンジ |
| `中止` | タスク中止 | 赤 |

**将来の拡張**（Phase 2計画）:
- `キャンセル` - 計画中止
- `アーカイブ` - 完了後のアーカイブ

詳細: [03-deletion-policy.md](./03-deletion-policy.md#phase-2-削除機能の拡張)

---

### priorityフィールド

| 値 | 説明 | 表示順序 |
|----|------|---------|
| `最高` | 最優先 | 1 |
| `高` | 高優先度 | 2 |
| `中` | 中優先度 | 3 |
| `低` | 低優先度 | 4 |

---

## 📐 JSON Schema（完全版）

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["meta", "tasks"],
  "properties": {
    "meta": {
      "type": "object",
      "required": ["projectName", "version", "createdAt", "lastModified"],
      "properties": {
        "projectName": { "type": "string", "minLength": 1 },
        "version": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "createdAt": { "type": "string", "format": "date-time" },
        "lastModified": { "type": "string", "format": "date-time" }
      }
    },
    "tasks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["wbs_no", "phase", "task_name", "status", "deleted"],
        "properties": {
          "wbs_no": {
            "type": "string",
            "pattern": "^[A-Z]+\\.[0-9]+\\.[0-9]+$",
            "description": "WBS番号: {Phase}.{Major}.{Minor}"
          },
          "phase": { "type": "string", "minLength": 1 },
          "task_type": { "type": "string" },
          "task_name": { "type": "string", "minLength": 1 },
          "primary_owner": { "type": "string" },
          "support_owner": { "type": "string" },
          "duration_bd": { "type": "number", "minimum": 0 },
          "start_date": {
            "type": "string",
            "pattern": "^\\d{4}-\\d{2}-\\d{2}$"
          },
          "finish_date": {
            "type": "string",
            "pattern": "^\\d{4}-\\d{2}-\\d{2}$"
          },
          "status": {
            "type": "string",
            "enum": ["未着手", "進行中", "完了", "保留", "中止"]
          },
          "priority": {
            "type": "string",
            "enum": ["最高", "高", "中", "低"]
          },
          "predecessors": { "type": "string" },
          "verification_method": { "type": "string" },
          "pass_criteria": { "type": "string" },
          "mermaid_ids": { "type": "string" },
          "display_order": { "type": "number", "minimum": 0 },
          "deleted": { "type": "boolean" },
          "deleted_at": {
            "type": ["string", "null"],
            "format": "date-time"
          },
          "comments": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "type": { "type": "string" },
                "text": { "type": "string" },
                "timestamp": { "type": "string", "format": "date-time" },
                "user": { "type": "string" }
              }
            }
          },
          "flowchart": {
            "type": "object",
            "properties": {
              "nodeIds": {
                "type": "array",
                "items": { "type": "string" }
              },
              "memos": {
                "type": "object",
                "patternProperties": {
                  "^node_.*$": { "type": "string" }
                }
              },
              "customLabels": {
                "type": "object",
                "patternProperties": {
                  "^node_.*$": { "type": "string" }
                }
              },
              "relatedManualEdges": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
  }
}
```

---

## 💾 LocalStorageキー

### プライマリキー: `task-manager-data`

**値**: JSON文字列（上記のプロジェクトデータ全体）

```javascript
// 保存
localStorage.setItem('task-manager-data', JSON.stringify(projectData));

// 読み込み
const projectData = JSON.parse(localStorage.getItem('task-manager-data'));
```

---

### 更新トリガー: `task-manager-update-trigger`

**値**: タイムスタンプ文字列

```javascript
// 更新通知（他のHTMLへ通知）
localStorage.setItem('task-manager-update-trigger', Date.now().toString());
```

詳細: [../data-integration/future-roadmap.md](../data-integration/future-roadmap.md)

---

## 🔄 データバージョン管理

### 現在のバージョン: `2.0`

| バージョン | リリース日 | 主な変更 |
|----------|-----------|---------|
| **1.0** | 2026-01-01 | 初版リリース |
| **1.5** | 2026-02-15 | comments フィールド追加 |
| **2.0** | 2026-03-01 | flowchart, mermaid_ids, deleted, deleted_at フィールド追加 |

---

### マイグレーション戦略

#### v1.0 → v2.0 マイグレーション

```javascript
function migrateToV2(data) {
  // バージョンチェック
  if (data.meta.version === '2.0') {
    return data; // 既にv2.0
  }

  // v1.0 → v2.0
  if (data.meta.version === '1.0' || data.meta.version === '1.5') {
    data.tasks.forEach(task => {
      // deleted フィールド追加
      if (task.deleted === undefined) {
        task.deleted = false;
        task.deleted_at = null;
      }

      // flowchart フィールド追加
      if (!task.flowchart) {
        task.flowchart = {
          nodeIds: [],
          memos: {},
          customLabels: {},
          relatedManualEdges: []
        };
      }

      // mermaid_ids フィールド追加
      if (!task.mermaid_ids) {
        task.mermaid_ids = '';
      }
    });

    data.meta.version = '2.0';
  }

  return data;
}
```

---

## 📤 JSONエクスポート・インポート

### エクスポート形式

```json
{
  "meta": {
    "projectName": "プロジェクト名",
    "version": "2.0",
    "createdAt": "2026-03-22T08:00:00.000Z",
    "lastModified": "2026-03-22T10:30:00.000Z"
  },
  "tasks": [
    {
      "wbs_no": "TEST.1.1",
      "deleted": false,
      ...
    },
    {
      "wbs_no": "TEST.2.1",
      "deleted": true,
      "deleted_at": "2026-03-22T10:30:00.000Z",
      ...
    }
  ]
}
```

**削除済みタスクの扱い**:
- ✅ 削除済みタスクも含めてエクスポート
- ✅ `deleted: true` フィールドで識別

---

### インポート処理

```javascript
function importData(jsonData) {
  // バージョンマイグレーション
  jsonData = migrateToV2(jsonData);

  // JSON Schema検証（オプション）
  if (!validateProjectData(jsonData)) {
    throw new Error('Invalid project data format');
  }

  // データ保存
  projectData = jsonData;
  saveToLocalStorage();
  renderTaskTable(); // 削除済みタスクは自動的にフィルタリング
}
```

---

## 🔍 検索・フィルタリング

### WBS番号での検索

```javascript
function searchByWbsNo(query) {
  return projectData.tasks.filter(task =>
    task.wbs_no.includes(query) && !task.deleted
  );
}

// ユーザー入力: "TEST.15"
// 結果: TEST.15.1, TEST.15.2等がマッチ（削除済みは除外）
```

---

### フェーズでのフィルタリング

```javascript
function filterByPhase(phase) {
  return projectData.tasks.filter(task =>
    task.phase === phase && !task.deleted
  );
}
```

---

### ステータスでのフィルタリング

```javascript
function filterByStatus(status) {
  return projectData.tasks.filter(task =>
    task.status === status && !task.deleted
  );
}
```

---

## 📊 スケーラビリティ

### タスク数の増加

| タスク数 | LocalStorageサイズ | パフォーマンス |
|---------|------------------|--------------|
| **100件** | ~500KB | 問題なし |
| **1000件** | ~5MB | 問題なし |
| **10000件** | ~50MB | ⚠️ LocalStorage制限に注意（5-10MB） |

**対策**（将来）:
- IndexedDBへの移行
- ページネーション
- 削除済みタスクの定期的なアーカイブ

---

## 🔗 関連ドキュメント

- [02-wbs-numbering-system.md](./02-wbs-numbering-system.md) - WBS番号体系
- [03-deletion-policy.md](./03-deletion-policy.md) - 削除ポリシー
- [../data-integration/](../data-integration/) - フローチャート連携
- [archive/session/03_ID-policy/06_data-structure-context.md](../../../archive/session/03_ID-policy/06_data-structure-context.md) - データ構造詳細分析

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成（v2.0仕様を完全版として統合）、JSON Schema追加 |

---

**最終更新**: 2026-03-22
**ステータス**: 実装済み（v2.13.4時点）
**データバージョン**: 2.0
