# データ構造定義 - クロスHTML連携

---
**feature**: クロスHTML連携
**document**: データ構造定義
**version**: 1.0
**status**: design
**created**: 2026-03-21
**updated**: 2026-03-21
---

## LocalStorageキー一覧

| キー名 | 型 | サイズ目安 | 更新頻度 |
|--------|------|----------|---------|
| `task-manager-data` | JSON Object | 100KB-1MB | タスク編集時 |
| `flowchart-attributes` | JSON Object | 10KB-100KB | ノード関連付け時 |
| `task-manager-update-trigger` | String | 13bytes | タスク編集時 |
| `flowchart-update-trigger` | String | 13bytes | ノード編集時 |

---

## task-manager-data（メインデータ）

### JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["tasks", "meta", "timestamp"],
  "properties": {
    "tasks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["wbs_no", "task_name"],
        "properties": {
          "wbs_no": { "type": "string" },
          "task_name": { "type": "string" },
          "start_date": { "type": "string", "pattern": "^\\d{4}/\\d{1,2}/\\d{1,2}$" },
          "finish_date": { "type": "string", "pattern": "^\\d{4}/\\d{1,2}/\\d{1,2}$" },
          "status": { "type": "string", "enum": ["未着手", "進行中", "完了", "保留", "中止"] },
          "priority": { "type": "string", "enum": ["低", "中", "高"] },
          "duration_bd": { "type": "number" },
          "flowchart": {
            "type": "object",
            "properties": {
              "nodeId": { "type": "string" },
              "nodeType": { "type": "string", "enum": ["process", "decision", "start", "end", "data"] },
              "position": {
                "type": "object",
                "properties": {
                  "x": { "type": "number" },
                  "y": { "type": "number" }
                }
              },
              "timestamp": { "type": "number" }
            }
          }
        }
      }
    },
    "meta": {
      "type": "object",
      "properties": {
        "project_name": { "type": "string" },
        "version": { "type": "string" }
      }
    },
    "timestamp": { "type": "number" }
  }
}
```

### 実例

```json
{
  "tasks": [
    {
      "wbs_no": "WBS1.1.0",
      "task_name": "UI設計",
      "start_date": "2026/3/2",
      "finish_date": "2026/3/8",
      "status": "進行中",
      "priority": "高",
      "duration_bd": 7,
      "flowchart": {
        "nodeId": "node-process-1",
        "nodeType": "process",
        "position": { "x": 150, "y": 200 },
        "connectedTo": ["node-decision-2"],
        "timestamp": 1234567890
      }
    }
  ],
  "meta": {
    "project_name": "サンプルプロジェクト",
    "version": "1.0"
  },
  "timestamp": 1234567890
}
```

---

## flowchart-attributes（フローチャート属性）

### JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "patternProperties": {
    "^WBS[0-9.]+$": {
      "type": "object",
      "required": ["nodeId", "nodeType", "timestamp"],
      "properties": {
        "nodeId": { "type": "string" },
        "nodeType": { "type": "string", "enum": ["process", "decision", "start", "end", "data"] },
        "position": {
          "type": "object",
          "properties": {
            "x": { "type": "number" },
            "y": { "type": "number" }
          }
        },
        "shape": { "type": "string" },
        "color": { "type": "string" },
        "connectedTo": {
          "type": "array",
          "items": { "type": "string" }
        },
        "metadata": { "type": "object" },
        "timestamp": { "type": "number" }
      }
    }
  }
}
```

### 実例

```json
{
  "WBS1.1.0": {
    "nodeId": "node-process-1",
    "nodeType": "process",
    "position": { "x": 150, "y": 200 },
    "shape": "rectangle",
    "color": "#3498db",
    "connectedTo": ["node-decision-2"],
    "metadata": {
      "layer": 1,
      "group": "UI設計フェーズ"
    },
    "timestamp": 1234567890
  },
  "WBS1.2.0": {
    "nodeId": "node-decision-2",
    "nodeType": "decision",
    "position": { "x": 300, "y": 200 },
    "shape": "diamond",
    "color": "#e74c3c",
    "connectedTo": ["node-process-3", "node-end-1"],
    "timestamp": 1234567891
  }
}
```

---

## NodeType定義

| nodeType | 日本語名 | SVG形状 | 用途 |
|----------|---------|---------|------|
| `process` | 処理 | 長方形 | 通常のタスク・作業 |
| `decision` | 判断 | ひし形 | 条件分岐・意思決定 |
| `start` | 開始 | 角丸長方形 | フロー開始点 |
| `end` | 終了 | 角丸長方形 | フロー終了点 |
| `data` | データ | 平行四辺形 | データ入出力 |

---

## 更新トリガー

### task-manager-update-trigger

```javascript
// 値: タイムスタンプ文字列
"1711043210123"

// 設定タイミング
localStorage.setItem('task-manager-update-trigger', Date.now().toString());
```

### flowchart-update-trigger

```javascript
// 値: タイムスタンプ文字列
"1711043210456"

// 設定タイミング
localStorage.setItem('flowchart-update-trigger', Date.now().toString());
```

---

## バージョン管理（将来拡張）

### データバージョン

```json
{
  "version": "1.0",
  "tasks": [...],
  "meta": {...}
}
```

### マイグレーション戦略

```javascript
function migrateData(data) {
  const currentVersion = data.version || "0.9";

  if (currentVersion === "0.9") {
    // v0.9 → v1.0 マイグレーション
    data.tasks.forEach(task => {
      if (!task.flowchart) {
        task.flowchart = null; // デフォルト値追加
      }
    });
    data.version = "1.0";
  }

  return data;
}
```

---

## 関連ドキュメント

- [README.md](./README.md)
- [01-localstorage-design.md](./01-localstorage-design.md)
- [03-flowchart-integration.md](./03-flowchart-integration.md)

---

**最終更新**: 2026-03-21
**ステータス**: 設計完了
