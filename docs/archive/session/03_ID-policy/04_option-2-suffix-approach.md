# 選択肢2: `_del` サフィックス付与

**アプローチ**: 削除時にWBS番号を `{元番号}_del` に変更し、削除されたことを明示的にマークする

**方針**: 削除済みタスクのWBS番号を変更することで、新規タスクが同じ番号を使用可能にする

---

## 📋 概要

タスク削除時に、WBS番号を `TEST.15.1` → `TEST.15.1_del` のように変更します。これにより、削除済みタスクが明示的に可視化され、元の番号（`TEST.15.1`）は新規タスクで再利用可能になります。

---

## 💻 実装コード

### **削除関数の修正**

```javascript
function deleteTask(wbsNo) {
  const task = projectData.tasks.find(t => t.wbs_no === wbsNo);

  if (!task) {
    alert('タスクが見つかりません');
    return;
  }

  // 確認ダイアログ
  if (!confirm(`タスク「${task.task_name}」(${wbsNo})を削除しますか？\n\n※削除されたタスクはエクスポートデータに残りますが、画面には表示されなくなります。`)) {
    return;
  }

  // 【新規】WBS番号にサフィックス付与
  const originalWbsNo = task.wbs_no;
  task.wbs_no = `${originalWbsNo}_del`;

  // 論理削除
  task.deleted = true;
  task.deleted_at = new Date().toISOString();

  // 【新規】他のタスクの参照を更新
  updateTaskReferences(originalWbsNo, task.wbs_no);

  // LocalStorageに保存
  saveToLocalStorage();

  // 通知表示
  showNotification({
    type: 'success',
    title: '削除完了',
    content: `タスク「${task.task_name}」を削除しました`
  });

  // UIを再描画
  renderTaskTable();
  renderDashboard();
  closeModal();
}
```

---

### **参照更新関数の実装**

```javascript
/**
 * WBS番号変更時に、他のタスクの参照フィールドを更新
 * @param {string} oldWbsNo - 変更前のWBS番号
 * @param {string} newWbsNo - 変更後のWBS番号
 */
function updateTaskReferences(oldWbsNo, newWbsNo) {
  projectData.tasks.forEach(task => {
    // predecessorsフィールドの更新
    if (task.predecessors) {
      const predecessorsList = task.predecessors.split(',').map(p => p.trim());
      const updatedList = predecessorsList.map(p =>
        p === oldWbsNo ? newWbsNo : p
      );
      task.predecessors = updatedList.join(', ');
    }

    // flowchart.nodeIdsの更新（フローチャート連携）
    if (task.flowchart && task.flowchart.nodeIds) {
      task.flowchart.nodeIds = task.flowchart.nodeIds.map(id =>
        id === oldWbsNo ? newWbsNo : id
      );
    }

    // 他の参照フィールドがあれば同様に更新
    // 例: dependencies, related_tasks 等
  });
}
```

---

### **検索機能の修正**

```javascript
// WBS番号検索時に_delサフィックスを考慮
function searchTasks(query) {
  const normalizedQuery = query.toLowerCase().replace('_del', '');

  return projectData.tasks.filter(task => {
    const normalizedWbs = task.wbs_no.toLowerCase().replace('_del', '');
    return normalizedWbs.includes(normalizedQuery) ||
           task.task_name.toLowerCase().includes(query.toLowerCase());
  });
}
```

---

## 🎯 動作例

### **ケース1: 削除時のWBS番号変更**

```
削除前:
{
  "wbs_no": "TEST.15.1",
  "task_name": "要件定義",
  "deleted": false
}

削除後:
{
  "wbs_no": "TEST.15.1_del",  // ← 変更
  "task_name": "要件定義",
  "deleted": true,
  "deleted_at": "2026-03-22T10:30:00.000Z"
}
```

---

### **ケース2: 参照の自動更新**

```
削除前:
Task A: {"wbs_no": "TEST.15.1", "deleted": false}
Task B: {"wbs_no": "TEST.16.1", "predecessors": "TEST.15.1"}

Task Aを削除:
Task A: {"wbs_no": "TEST.15.1_del", "deleted": true}
Task B: {"wbs_no": "TEST.16.1", "predecessors": "TEST.15.1_del"}  // ← 自動更新
```

---

### **ケース3: 新規タスクでの番号再利用**

```
削除済みタスク:
- TEST.15.1_del (削除済み)

新規作成時:
→ phaseTasks から TEST.15.1_del を除外（_delサフィックス付き）
→ maxNumber = 15（_delを除く最大値）
→ 生成: TEST.16.1

ユーザーが手動で TEST.15.1 に変更可能
→ WBS番号の重複チェックで TEST.15.1_del は除外される
→ TEST.15.1 を使用可能（再利用）
```

---

## ✅ メリット

### **1. トレーサビリティの向上**
- 削除されたタスクが明示的に可視化
- JSONエクスポート時に一目瞭然
- 監査証跡の明確化

```json
{
  "tasks": [
    {"wbs_no": "TEST.14.1", "deleted": false},
    {"wbs_no": "TEST.15.1_del", "deleted": true},  // ← 削除されたことが明示的
    {"wbs_no": "TEST.16.1", "deleted": false}
  ]
}
```

### **2. ID衝突の完全防止**
- 元の番号と削除済み番号が明確に区別される
- WBS番号の重複が発生しない
- データベースの一意性制約に準拠

### **3. 削除済みタスクの検索性向上**
- `_del` で検索すれば削除済みタスクのみ抽出可能
- フィルタリングが容易
- レポート生成時に有用

### **4. 依存関係の追跡**
- どのタスクが削除済みタスクを参照しているか一目瞭然
- predecessorsフィールドで `TEST.15.1_del` が見えれば、削除済みタスクへの依存がわかる

### **5. エンタープライズ向け**
- コンプライアンス要件に対応
- 監査ログの明確化
- 外部システムとの統合時に削除フラグが可視化

---

## ❌ デメリット

### **1. 実装コスト（中～高）**

**必要な実装**:
1. `deleteTask()` 関数の修正（WBS番号変更）
2. `updateTaskReferences()` 関数の新規実装
3. 検索機能の修正（`_del` サフィックス考慮）
4. WBS番号重複チェックの修正
5. フィルタリング機能の修正
6. CSV/JSONエクスポートの修正

**工数**: 4-6時間

---

### **2. データ整合性リスク（高）**

#### **問題1: predecessorsフィールドの参照破壊**

```json
// 削除前
{"wbs_no": "TEST.16.1", "predecessors": "TEST.15.1"}

// TEST.15.1を削除（参照更新に失敗した場合）
{"wbs_no": "TEST.16.1", "predecessors": "TEST.15.1"}  // ← 存在しないタスクを参照
```

#### **問題2: JSONエクスポート・インポートの複雑化**

```json
// エクスポート時
{"wbs_no": "TEST.15.1_del", "deleted": true}

// インポート時の処理
// - _delサフィックスを保持するか？
// - 元の番号に戻すか？
// - 重複チェックはどうするか？
```

#### **問題3: フローチャート連携への影響**

```javascript
// flowchart-editor.htmlでの参照
task.mermaid_ids = "node_15_1";  // WBS番号と連動

// TEST.15.1を削除 → TEST.15.1_del
// node_15_1 はそのまま？ node_15_1_del に変更？
```

---

### **3. 業界標準からの乖離**

主要プロジェクト管理ツールの比較:

| ツール | 削除時のID変更 | 理由 |
|--------|--------------|------|
| **GitHub** | ❌ しない | トレーサビリティ、URL安定性 |
| **Jira** | ❌ しない | API安定性、外部統合 |
| **Asana** | ❌ しない | タスク参照の保護 |
| **Trello** | ❌ しない | カードURL不変 |
| **Monday.com** | ❌ しない | アイテムID不変 |

**業界標準**: **ID不変性が原則**

---

### **4. ユーザー体験の混乱**

#### **問題1: 検索時の混乱**

```
ユーザーが "TEST.15.1" で検索
→ ヒットしない（TEST.15.1_del になっているため）
→ 「あれ？このタスクどこ行った？」
```

#### **問題2: JSONファイルの可読性低下**

```json
{
  "wbs_no": "TEST.15.1_del",  // ← ノイズ
  "wbs_no": "TEST.16.1_del",
  "wbs_no": "TEST.17.1"
}
```

削除が多いと `_del` だらけになる

---

### **5. 参照更新の漏れリスク**

**更新が必要なフィールド**:
- `predecessors`（前提タスク）
- `flowchart.nodeIds`（フローチャート連携）
- 将来追加されるフィールド（例: `dependencies`, `related_tasks`）

**リスク**:
- 1箇所でも更新を忘れると、参照整合性が壊れる
- バグの発見が困難（参照先が存在しないエラー）

---

### **6. JSONインポート時の複雑化**

```javascript
// インポート時の処理
function importTasks(jsonData) {
  jsonData.tasks.forEach(task => {
    // _delサフィックスがあるタスクをどう扱う？
    if (task.wbs_no.endsWith('_del')) {
      // 1. そのままインポート？
      // 2. _delを除去して元に戻す？
      // 3. deletedフラグと整合性チェック？
    }
  });
}
```

---

## 📊 適用シーン

### **✅ 適している場合**

1. **エンタープライズプロジェクト**:
   - 監査必須
   - コンプライアンス要件あり
   - 外部報告が必要

2. **長期運用プロジェクト**（2年以上）:
   - トレーサビリティが最重要
   - 削除理由の可視化が必須
   - 外部システムとの統合

3. **複数チーム間の連携**:
   - 削除されたタスクを明示する必要
   - ドキュメント・レポートでの参照性
   - 外部ステークホルダーへの説明責任

---

### **❌ 適していない場合**

1. **個人プロジェクト**:
   - 複雑さが不要
   - 実装コストが見合わない

2. **短期プロジェクト**（3ヶ月～1年）:
   - シンプルさ重視
   - 過剰な仕組み

3. **小規模チーム**（1-5人）:
   - 内部利用のみ
   - オーバーエンジニアリング

---

## ⚠️ 実装時の注意点

### **1. 参照更新の完全性**

```javascript
// すべての参照フィールドをチェック
const REFERENCE_FIELDS = [
  'predecessors',
  'flowchart.nodeIds',
  'dependencies',  // 将来追加の可能性
  'related_tasks'  // 将来追加の可能性
];

function updateAllReferences(oldWbsNo, newWbsNo) {
  REFERENCE_FIELDS.forEach(field => {
    updateFieldReferences(field, oldWbsNo, newWbsNo);
  });
}
```

---

### **2. トランザクション的な処理**

```javascript
function deleteTask(wbsNo) {
  try {
    // 1. WBS番号変更
    const originalWbsNo = task.wbs_no;
    task.wbs_no = `${originalWbsNo}_del`;

    // 2. 参照更新
    updateTaskReferences(originalWbsNo, task.wbs_no);

    // 3. 削除フラグ設定
    task.deleted = true;
    task.deleted_at = new Date().toISOString();

    // 4. 保存
    saveToLocalStorage();
  } catch (error) {
    // ロールバック
    task.wbs_no = originalWbsNo;
    throw error;
  }
}
```

---

### **3. 後方互換性の保持**

```javascript
// 既存データ（_delサフィックスなし）との互換性
function loadData(jsonData) {
  jsonData.tasks.forEach(task => {
    // 削除済みだがサフィックスがない場合の処理
    if (task.deleted && !task.wbs_no.endsWith('_del')) {
      // マイグレーション: サフィックスを追加
      task.wbs_no = `${task.wbs_no}_del`;
    }
  });
}
```

---

## 🔄 代替案: ハイブリッドアプローチ

### **案: データ層とUI層で分離**

```javascript
// データ層: WBS番号は変更しない（選択肢1）
{
  "wbs_no": "TEST.15.1",
  "deleted": true
}

// UI層: 表示時のみサフィックスを付与
function displayWbsNo(task) {
  return task.deleted ? `${task.wbs_no}_del` : task.wbs_no;
}
```

**メリット**:
- データ整合性維持（選択肢1）
- 視覚的な明示（選択肢2）
- 参照更新不要

**デメリット**:
- JSONエクスポート時に `_del` が含まれない
- トレーサビリティが中途半端

---

## 📝 まとめ

### **総合評価**

| 評価軸 | スコア | 理由 |
|--------|--------|------|
| **実装コスト** | ⭐⭐ (2/5) | 4-6時間の工数 |
| **保守性** | ⭐⭐ (2/5) | 参照更新ロジックが複雑 |
| **データ整合性** | ⭐⭐⭐ (3/5) | 参照破壊のリスクあり |
| **業界標準準拠** | ⭐ (1/5) | 主要ツールはID不変 |
| **ユーザビリティ** | ⭐⭐⭐⭐ (4/5) | 削除が明示的で分かりやすい |
| **拡張性** | ⭐⭐ (2/5) | 新しい参照フィールド追加時の対応が必要 |

**総合スコア**: 14/30 (47%)

---

### **推奨しない理由**

1. ❌ **実装コストが高い** - 4-6時間 vs 5分（選択肢1）
2. ❌ **データ整合性リスク** - 参照破壊の可能性
3. ❌ **業界標準から乖離** - 主要ツールはID不変が原則
4. ❌ **保守性の低下** - 参照更新ロジックが複雑
5. ❌ **JSONエクスポート・インポートの複雑化**

---

### **適用すべきケース**

このアプローチは、以下の**すべて**を満たす場合のみ検討すべき:
- [ ] エンタープライズプロジェクト（監査必須）
- [ ] 長期運用（2年以上）
- [ ] 外部システムとの統合あり
- [ ] 削除されたタスクの明示が法的要件
- [ ] 実装コスト（4-6時間）が許容範囲

**小規模チーム向けの本プロジェクトでは、選択肢1を推奨**

---

**次のステップ**: [05_industry-standards.md](./05_industry-standards.md) で業界標準との比較を確認
