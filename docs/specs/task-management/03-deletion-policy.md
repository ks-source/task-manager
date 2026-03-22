# タスク削除ポリシー - WBS番号管理

---

**feature**: タスク管理
**document**: 削除ポリシー・WBS番号再利用管理
**version**: 1.0
**status**: 仕様確定、実装待ち
**created**: 2026-03-22
**updated**: 2026-03-22

---

## 📖 概要

タスク削除時のWBS番号管理ポリシーを定義します。削除済みタスクのWBS番号を再利用するか否か、削除済みタスクのデータをどのように扱うかを明確化し、業界標準に準拠した実装方針を示します。

---

## 🎯 採用方針: **Option 1（削除済みID除外 + 全履歴考慮）**

### 基本原則

**ID不変性（Immutable ID）** - 一度割り当てられたWBS番号は、削除後も変更しない

---

## 📋 仕様詳細

### 1. 論理削除

タスク削除は「物理削除」ではなく「論理削除」で実装します。

#### データ構造
```json
{
  "wbs_no": "TEST.15.1",
  "task_name": "要件定義",
  "deleted": true,
  "deleted_at": "2026-03-22T10:30:00.000Z",
  "status": "完了",
  "predecessors": "TEST.14.1",
  // その他フィールドはそのまま保持
}
```

#### 実装済みフィールド
- `deleted`: boolean - 削除フラグ（`true` = 削除済み）
- `deleted_at`: string | null - 削除日時（ISO 8601形式）

**実装バージョン**: v2.13.0（2026-03-22）

---

### 2. WBS番号の不変性

#### ❌ **変更しないこと**（ID不変性）
- 削除済みタスクのWBS番号を変更しない（例: `TEST.15.1` → `TEST.15.1_del` のような変更は行わない）
- 削除済みタスクをJSON配列から削除しない
- 削除済みタスクの`wbs_no`フィールドを書き換えない

#### ✅ **変更すること**
- `deleted`フラグを`true`に設定
- `deleted_at`に削除日時を記録

---

### 3. WBS番号自動生成時の扱い

#### 現在の実装（v2.13.4）- ⚠️ **問題あり**

```javascript
// task-manager.html:3322-3351
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // 削除済みタスクを除外して履歴を確認（問題）
  const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);

  let maxNumber = 0;
  phaseTasks.forEach(t => {
    const parts = t.wbs_no.split('.');
    if (parts.length > 1) {
      const lastNum = parseFloat(parts[parts.length - 1]);
      if (!isNaN(lastNum) && lastNum > maxNumber) {
        maxNumber = lastNum;
      }
    }
  });

  const newNumber = Math.floor(maxNumber) + 1;
  const generatedWbsNo = `${phase}.${newNumber}.1`;

  document.getElementById('new-wbs-no').value = generatedWbsNo;
}
```

**問題点**:
- `&& !t.deleted` により削除済みタスクを除外している
- 削除済みタスクのWBS番号が再利用されてしまう

**具体例**:
```
TEST.15.1（削除済み）が存在
↓
新規タスク作成時、TEST.15.1が履歴から除外される
↓
TEST.15.1が再生成される（重複）
```

---

#### Option 1の実装（推奨）- ✅ **修正版**

```javascript
// task-manager.html:3330（修正箇所）
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // 削除済みタスクも含めて全履歴を考慮
  const phaseTasks = projectData.tasks.filter(t => t.phase === phase); // ← !t.deletedを削除

  let maxNumber = 0;
  phaseTasks.forEach(t => {
    const parts = t.wbs_no.split('.');
    if (parts.length > 1) {
      const lastNum = parseFloat(parts[parts.length - 1]);
      if (!isNaN(lastNum) && lastNum > maxNumber) {
        maxNumber = lastNum;
      }
    }
  });

  const newNumber = Math.floor(maxNumber) + 1;
  const generatedWbsNo = `${phase}.${newNumber}.1`;

  document.getElementById('new-wbs-no').value = generatedWbsNo;
}
```

**修正内容**:
- **1行のみ変更**: `filter(t => t.phase === phase && !t.deleted)` → `filter(t => t.phase === phase)`
- `&& !t.deleted` 条件を削除

**結果**:
```
TEST.15.1（削除済み）が存在
↓
新規タスク作成時、TEST.15.1が履歴に含まれる
↓
次のWBS番号はTEST.16.1が生成される（重複なし）
```

---

### 4. predecessorsフィールドへの影響

#### データ構造
```json
{
  "wbs_no": "TEST.16.1",
  "task_name": "設計",
  "predecessors": "TEST.15.1,TEST.14.1"
}
```

#### Option 1の場合（ID不変）
```json
// TEST.15.1を削除
{
  "wbs_no": "TEST.15.1",
  "task_name": "要件定義",
  "deleted": true,
  "deleted_at": "2026-03-22T10:30:00.000Z"
}

// TEST.16.1の参照は壊れない
{
  "wbs_no": "TEST.16.1",
  "task_name": "設計",
  "predecessors": "TEST.15.1,TEST.14.1"  // ✅ 参照有効（TEST.15.1は存在する）
}
```

**処理時の扱い**:
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

**メリット**:
- ✅ predecessors参照が壊れない
- ✅ 削除済みタスクを特別扱いで表示可能
- ✅ 参照更新ロジック不要

---

### 5. UI表示での扱い

#### タスク一覧（メイン画面）

```javascript
// task-manager.html:1935 renderTaskTable()
function renderTaskTable() {
  const filteredTasks = projectData.tasks.filter(t => !t.deleted); // 削除済みは非表示
  // ...
}
```

**実装済み（v2.13.0）**: 削除済みタスクは自動的に非表示

---

#### JSONエクスポート

```json
{
  "meta": {...},
  "tasks": [
    {"wbs_no": "TEST.1.1", "deleted": false, ...},
    {"wbs_no": "TEST.2.1", "deleted": true, "deleted_at": "...", ...},
    {"wbs_no": "TEST.3.1", "deleted": false, ...}
  ]
}
```

**実装済み（v2.13.0）**: 削除済みタスクもJSONに含まれる

---

## 🏆 採用理由（外部AI評価結果）

### 評価したAI
1. **Claude Opus 4.5** - 総合評価レポート（21.6KB）
2. **GPT-4** - 再評価と補完提言（14.4KB）
3. **Gemini** - 批判的レビュー（14.4KB）

### 全AI一致の推奨事項

| 評価項目 | Option 1（ID不変） | Option 2（ID変更） | 推奨 |
|---------|-------------------|-------------------|------|
| **ID不変性** | ✅ 保持 | ❌ 変更 | ✅ Option 1 |
| **実装コスト** | ✅ 最小（5分） | ❌ 高（4-6h） | ✅ Option 1 |
| **predecessors参照** | ✅ 壊れない | ⚠️ 更新ロジック必要 | ✅ Option 1 |
| **業界標準準拠** | ✅ GitHub/Jira | ❌ 非標準 | ✅ Option 1 |
| **リスク評価** | ✅ 低（2/10） | ⚠️ 高（7/10） | ✅ Option 1 |

詳細: [archive/session/03_ID-policy/feedback/](../../../archive/session/03_ID-policy/feedback/)

---

### 業界標準との比較

| システム | ID再利用 | 削除後の扱い | 理由 |
|---------|---------|------------|------|
| **GitHub Issues** | ❌ しない | #1234（クローズのみ） | URL安定性、外部リンク保護 |
| **Jira** | ❌ しない | PROJ-1234（アーカイブ） | API安定性、監査証跡 |
| **Asana** | ❌ しない | 内部ID不変 | 依存関係保護 |
| **Monday.com** | ❌ しない | アイテムID不変 | ワークフロー保護 |
| **Trello** | ✅ する | カード削除後無効化 | シンプルさ重視（個人向け） |

**統計**: ID再利用なし 86%（6/7ツール）

**結論**: ID不変性が業界標準

詳細: [archive/session/03_ID-policy/05_industry-standards.md](../../../archive/session/03_ID-policy/05_industry-standards.md)

---

## 🔧 実装手順

### ⚠️ 実装前の必須修正

**Major/Minor番号抽出バグの修正が必須**（Opus指摘）

現在の `generateWBSNumber()` は Minor番号をMajor番号として使用しているため、Option 1実装前に修正が必要です。

詳細: [02-wbs-numbering-system.md](./02-wbs-numbering-system.md#major-minor-number-extraction-bug)

---

### Option 1実装（1行変更）

#### 修正箇所
- **ファイル**: task-manager.html
- **行番号**: 3330
- **関数**: `generateWBSNumber()`

#### Before（v2.13.4）
```javascript
const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);
```

#### After（v2.14.0予定）
```javascript
const phaseTasks = projectData.tasks.filter(t => t.phase === phase);
```

#### 工数見積もり
- **修正**: 5分
- **検証**: 10分
- **合計**: 15分

---

### 検証シナリオ

#### テストケース1: 削除後の新規タスク作成
```
1. TEST.15.1 を作成
2. TEST.15.1 を削除（deleted: true）
3. 新規タスク作成で「WBS番号生成」をクリック
4. 期待結果: TEST.16.1 が生成される（TEST.15.1 は再利用されない）
```

#### テストケース2: predecessors参照
```
1. TEST.15.1（削除済み）が存在
2. TEST.16.1 の predecessors に "TEST.15.1" を設定
3. タスク一覧を確認
4. 期待結果: エラーなし、参照が壊れない
```

#### テストケース3: JSONエクスポート・インポート
```
1. 削除済みタスクを含むプロジェクトをエクスポート
2. 新しいセッションでインポート
3. 期待結果: 削除済みタスクも正常にインポートされる
```

---

## 🚀 将来の拡張計画

### Phase 2: 削除機能の拡張

#### 2-1. 削除済みタスク表示トグル
```javascript
// UI追加: "削除済みタスクを表示" チェックボックス
let showDeleted = false;

function renderTaskTable() {
  const filteredTasks = showDeleted
    ? projectData.tasks // 全タスク表示
    : projectData.tasks.filter(t => !t.deleted); // 削除済み除外
  // ...
}
```

**工数**: 30-45分

---

#### 2-2. タスク復元機能
```javascript
function restoreTask(wbsNo) {
  const task = projectData.tasks.find(t => t.wbs_no === wbsNo);
  if (task && task.deleted) {
    task.deleted = false;
    task.deleted_at = null;
    saveData();
    renderTaskTable();
    showNotification({title: '復元完了', content: `${wbsNo} を復元しました`});
  }
}
```

**工数**: 1-1.5時間

---

#### 2-3. ステータス管理拡張（cancelled/archived）

**現在のステータス**:
```javascript
status: "未着手" | "進行中" | "完了" | "保留" | "中止"
```

**拡張案**（Gemini提言）:
```javascript
status: "未着手" | "進行中" | "完了" | "保留" | "中止" | "キャンセル" | "アーカイブ"
archived: boolean
cancelled: boolean
```

**メリット**:
- 削除（delete）とキャンセル（cancel）の区別が明確
- アーカイブ機能により、完了タスクを整理可能

**工数**: 2-3時間

詳細: [archive/session/03_ID-policy/feedback/03_gemini_WBS_Reuse_Policy_Critical_Review.md.md](../../../archive/session/03_ID-policy/feedback/03_gemini_WBS_Reuse_Policy_Critical_Review.md.md)

---

### Phase 3: 内部ID分離（Option 3）

#### 概要（GPT提言）

**現在の問題**:
- WBS番号が「主キー」と「表示ID」の二重の役割を持つ
- WBS番号を変更すると、predecessors参照が全て壊れる

**Option 3の提案**:
```json
{
  "task_id": "uuid-1234-5678",           // 内部ID（不変、主キー）
  "wbs_no": "TEST.15.1",                 // 表示ID（変更可能）
  "predecessors": ["uuid-1234-5677"],    // task_idで参照
  "deleted": false
}
```

**メリット**:
- ✅ WBS番号を自由に変更可能
- ✅ predecessors参照が壊れない
- ✅ 削除・復元が容易

**デメリット**:
- ❌ データマイグレーションが必要（全タスクにtask_id追加）
- ❌ predecessors参照の変換が必要
- ❌ 実装コスト高（6-10時間）

**推奨タイミング**: Phase 2完了後、大規模プロジェクト運用時

詳細: [archive/session/03_ID-policy/feedback/02_gpt_WBS番号再利用ポリシー 再評価と補完提言.md](../../../archive/session/03_ID-policy/feedback/02_gpt_WBS番号再利用ポリシー%20再評価と補完提言.md)

---

## 📊 データ整合性への影響

### predecessorsフィールド

| 観点 | Option 1（ID不変） | Option 2（ID変更） |
|-----|-------------------|--------------------|
| **参照破壊** | ✅ なし | ⚠️ あり（更新ロジック必須） |
| **更新コスト** | ✅ 0 | ❌ 高（全参照を走査） |
| **エラーリスク** | ✅ 低 | ⚠️ 高（更新漏れ） |

---

### flowchartフィールド

```json
{
  "flowchart": {
    "nodeIds": ["node_1", "node_2"],
    "memos": {"node_1": "詳細メモ"}
  }
}
```

**WBS番号変更の影響**:
- ✅ `flowchart.nodeIds` はMermaidノードIDを保持（WBS番号とは別）
- ✅ WBS番号が変更されても影響なし

---

### JSONエクスポート・インポート

| 観点 | Option 1（ID不変） | Option 2（ID変更） |
|-----|-------------------|--------------------|
| **シンプルさ** | ✅ シンプル | ❌ 複雑（サフィックス検証必要） |
| **整合性チェック** | ✅ 不要 | ⚠️ 必要（`_del`整合性） |
| **マイグレーション** | ✅ 不要 | ⚠️ 必要（旧データ対応） |

---

## 📝 参考資料

### 詳細分析ドキュメント（archive）
- [00_PROMPT_TO_AI.md](../../../archive/session/03_ID-policy/00_PROMPT_TO_AI.md) - 外部AI評価用プロンプト
- [01_current-implementation.md](../../../archive/session/03_ID-policy/01_current-implementation.md) - 現在の実装
- [02_problem-statement.md](../../../archive/session/03_ID-policy/02_problem-statement.md) - 問題定義
- [03_option-1-exclude-only.md](../../../archive/session/03_ID-policy/03_option-1-exclude-only.md) - Option 1詳細
- [04_option-2-suffix-approach.md](../../../archive/session/03_ID-policy/04_option-2-suffix-approach.md) - Option 2詳細
- [05_industry-standards.md](../../../archive/session/03_ID-policy/05_industry-standards.md) - 業界標準比較
- [06_data-structure-context.md](../../../archive/session/03_ID-policy/06_data-structure-context.md) - データ構造

### 外部AI評価レポート
- [01_opus_WBS番号再利用ポリシー 総合評価レポート.md](../../../archive/session/03_ID-policy/feedback/01_opus_WBS番号再利用ポリシー%20総合評価レポート.md) - Opus評価
- [02_gpt_WBS番号再利用ポリシー 再評価と補完提言.md](../../../archive/session/03_ID-policy/feedback/02_gpt_WBS番号再利用ポリシー%20再評価と補完提言.md) - GPT評価
- [03_gemini_WBS_Reuse_Policy_Critical_Review.md.md](../../../archive/session/03_ID-policy/feedback/03_gemini_WBS_Reuse_Policy_Critical_Review.md.md) - Gemini評価

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成、Option 1仕様確定、外部AI評価結果統合 |

---

**最終更新**: 2026-03-22
**ステータス**: 仕様確定、実装待ち
**次のアクション**: Major/Minor番号バグ修正 → Option 1実装（1行変更）
