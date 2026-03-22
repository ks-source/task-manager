# 選択肢1: 削除済みID除外 + 全履歴考慮

**アプローチ**: 削除済みタスクのWBS番号を再利用せず、全タスク（削除含む）の履歴から次の番号を生成する

**方針**: WBS番号の不変性を保ちつつ、削除済み番号は欠番として残す

---

## 📋 概要

削除済みタスクのWBS番号は変更せず、新規タスク作成時の自動生成で削除済みタスクを含む全履歴を考慮してmaxNumberを算出します。これにより、削除された番号は「欠番」として永久に保持され、再利用されません。

---

## 💻 実装コード

### **現在のコード** (3322-3351行)

```javascript
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // Find all tasks in the same phase (削除済み除外)
  const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);

  // Extract numeric parts and find max
  let maxNumber = 0;
  phaseTasks.forEach(t => {  // ← 削除済みタスクが含まれない
    const parts = t.wbs_no.split('.');
    if (parts.length > 1) {
      const lastNum = parseFloat(parts[parts.length - 1]);
      if (!isNaN(lastNum) && lastNum > maxNumber) {
        maxNumber = lastNum;
      }
    }
  });

  // Generate new number
  const newNumber = Math.floor(maxNumber) + 1;
  const generatedWbsNo = `${phase}.${newNumber}.1`;

  document.getElementById('new-wbs-no').value = generatedWbsNo;
  console.log(`Generated WBS Number: ${generatedWbsNo}`);
}
```

---

### **修正後のコード（選択肢1）**

```javascript
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // Find all tasks in the same phase (削除済みタスク含む全履歴)
  const allPhaseTasks = projectData.tasks.filter(t => t.phase === phase);

  // Extract numeric parts and find max (削除済み含む全タスクから算出)
  let maxNumber = 0;
  allPhaseTasks.forEach(t => {
    const parts = t.wbs_no.split('.');
    if (parts.length > 1) {
      const lastNum = parseFloat(parts[parts.length - 1]);
      if (!isNaN(lastNum) && lastNum > maxNumber) {
        maxNumber = lastNum;
      }
    }
  });

  // Generate new number
  const newNumber = Math.floor(maxNumber) + 1;
  const generatedWbsNo = `${phase}.${newNumber}.1`;

  document.getElementById('new-wbs-no').value = generatedWbsNo;
  console.log(`Generated WBS Number: ${generatedWbsNo}`);
}
```

**変更箇所**:
- **3330行**: `&& !t.deleted` を削除
- **変更は1箇所のみ**（1行の修正）

---

## 🎯 動作例

### **ケース1: 全削除後の番号生成**

```
既存タスク:
- TEST.1.1 (削除済み: deleted=true)
- TEST.2.1 (削除済み: deleted=true)
- TEST.3.1 (削除済み: deleted=true)

新規作成時（修正前）:
→ phaseTasks = [] (空配列)
→ maxNumber = 0
→ 生成: TEST.1.1 ❌ (再利用)

新規作成時（修正後）:
→ allPhaseTasks = [TEST.1.1, TEST.2.1, TEST.3.1] (削除含む)
→ maxNumber = 3
→ 生成: TEST.4.1 ✅
```

---

### **ケース2: 特定番号削除後の生成**

```
既存タスク:
- TEST.15.1 (削除済み: deleted=true)

新規作成時（修正前）:
→ phaseTasks = []
→ maxNumber = 0
→ 生成: TEST.1.1 ❌

新規作成時（修正後）:
→ allPhaseTasks = [TEST.15.1]
→ maxNumber = 15
→ 生成: TEST.16.1 ✅
```

---

### **ケース3: 中間番号削除後の生成**

```
既存タスク:
- TEST.1.1 (未削除)
- TEST.2.1 (削除済み)
- TEST.3.1 (未削除)
- TEST.4.1 (削除済み)
- TEST.5.1 (未削除)

新規作成時（修正前・修正後ともに同じ）:
→ maxNumber = 5
→ 生成: TEST.6.1 ✅

タスク一覧の表示:
TEST.1.1
TEST.3.1  ← 番号の歯抜け
TEST.5.1
TEST.6.1
```

**補足**:
- 番号の歯抜け（TEST.2, TEST.4がない）は残る
- ユーザーから「なぜTEST.2がないのか？」という疑問が出る可能性

---

## ✅ メリット

### **1. 実装コスト最小**
- **変更箇所**: 1行のみ（3330行）
- **工数**: 5分以内
- **テスト**: 既存機能に影響なし
- **リスク**: 極めて低い

### **2. WBS番号の不変性維持**
- 削除時にWBS番号を変更しない
- JSONエクスポート・インポートの整合性維持
- predecessorsフィールドの参照が壊れない

### **3. データ整合性の保持**
- WBS番号の重複が発生しない
- 削除済みタスクと新規タスクのIDが衝突しない
- JSONデータの一意性が保証される

### **4. 業界標準に準拠**
- GitHub Issues, Jira, Asanaと同様のアプローチ
- ID不変性の原則に従う
- トレーサビリティの保持

### **5. 将来の拡張性**
- 削除済みタスク一覧表示機能の追加が容易
- タスク復元機能の実装が可能
- 監査ログの保持が容易

---

## ❌ デメリット

### **1. 番号の歯抜けが発生**
- 例: TEST.1, TEST.3, TEST.5（TEST.2, TEST.4が削除済み）
- ユーザーが混乱する可能性
- 「連番」という期待に反する

### **2. トレーサビリティが間接的**
- 削除されたタスクの存在が見えにくい
- 欠番の理由が不明（削除されたのか、最初からないのか）
- JSONエクスポートを確認しないと履歴がわからない

### **3. ユーザーからの疑問**
- 「なぜTEST.2がないのか？」
- 「番号が飛んでいる」
- 説明が必要（運用ルール）

### **4. 最大番号が増え続ける**
- 削除を繰り返すと番号が大きくなる
- 例: TEST.1000.1（実際には10件しかない）
- 視覚的に把握しづらい

---

## 📊 適用シーン

### **✅ 適している場合**

1. **個人プロジェクト**:
   - 1人での利用
   - 監査不要
   - シンプルさ重視

2. **短期プロジェクト**（3ヶ月～1年）:
   - 削除が少ない
   - 履歴追跡の重要性が低い
   - 試行錯誤が多い開発初期段階

3. **小規模チーム**（1-5人）:
   - 内部利用のみ
   - 柔軟な運用が可能
   - ドキュメント化の負担が少ない

---

### **❌ 適していない場合**

1. **エンタープライズプロジェクト**:
   - 監査必須
   - コンプライアンス要件あり
   - 外部報告が必要

2. **長期運用プロジェクト**（2年以上）:
   - 削除が頻繁
   - トレーサビリティが重要
   - 番号が極端に大きくなる可能性

3. **複数チーム間の連携**:
   - 欠番の理由を説明する必要
   - ドキュメント・レポートでの参照性
   - 外部システムとの統合

---

## 🛠️ 実装手順

### **ステップ1: コード修正** (5分)

```javascript
// task-manager.html:3330
// 修正前
const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);

// 修正後
const allPhaseTasks = projectData.tasks.filter(t => t.phase === phase);
```

---

### **ステップ2: 変数名変更** (3分)

```javascript
// task-manager.html:3333-3343
// 修正前
phaseTasks.forEach(t => {

// 修正後
allPhaseTasks.forEach(t => {
```

---

### **ステップ3: テスト** (10分)

**テストケース**:
1. 全削除後に新規作成 → TEST.4.1が生成されることを確認
2. 特定番号削除後に新規作成 → 削除番号より大きい番号が生成されることを確認
3. 中間番号削除後に新規作成 → 連番が正しく生成されることを確認

---

### **ステップ4: バージョン更新** (2分)

```html
<title>Task Manager - WBS Visualizer v2.13.5</title>
```

---

## ⚠️ 注意点

### **1. 後方互換性**
- 既存のJSONデータは変更なし
- 既存のタスクに影響なし
- ロールバックが容易

### **2. ユーザー教育**
- 欠番の理由を説明する必要がある
- 運用ルールの文書化
- FAQ作成（「なぜ番号が飛んでいるか？」）

### **3. 将来の拡張**
- 削除済みタスク一覧表示機能を追加することで、欠番の理由が可視化される
- タスク復元機能の実装により、欠番の解消も可能

---

## 🔄 関連する将来機能

### **削除済みタスク一覧表示** (Phase 2)

```
タスク一覧
├─ TEST.1.1 (未削除)
├─ TEST.2.1 (削除済み) ← グレー表示
├─ TEST.3.1 (未削除)
└─ TEST.4.1 (削除済み) ← グレー表示

フィルタ: [未削除のみ] [削除済みのみ] [すべて]
```

**メリット**:
- 欠番の理由が一目瞭然
- トレーサビリティの向上
- ユーザーの混乱解消

---

### **タスク復元機能** (Phase 3)

```
削除済みタスク一覧
TEST.2.1 「要件定義」 [復元]
TEST.4.1 「設計レビュー」 [復元]
```

**メリット**:
- 誤削除の復旧が容易
- 番号の欠番が解消される
- データ損失リスクの低減

---

## 📝 まとめ

### **推奨理由**

1. ✅ **実装コスト最小** - 1行の変更のみ
2. ✅ **データ整合性維持** - WBS番号の重複なし
3. ✅ **業界標準準拠** - GitHub/Jiraと同様のアプローチ
4. ✅ **将来の拡張性** - 削除済み一覧、復元機能の追加が容易
5. ✅ **リスク最小** - 既存機能への影響なし

### **デメリットへの対処**

1. **番号の歯抜け** → 削除済みタスク一覧表示で可視化
2. **トレーサビリティ** → JSONエクスポートで履歴保持
3. **ユーザー混乱** → FAQ、運用ルール文書化

---

## 🎯 総合評価

| 評価軸 | スコア | 理由 |
|--------|--------|------|
| **実装コスト** | ⭐⭐⭐⭐⭐ (5/5) | 1行の変更のみ |
| **保守性** | ⭐⭐⭐⭐⭐ (5/5) | シンプルで理解しやすい |
| **データ整合性** | ⭐⭐⭐⭐⭐ (5/5) | WBS番号の重複なし |
| **業界標準準拠** | ⭐⭐⭐⭐⭐ (5/5) | GitHub/Jiraと同様 |
| **ユーザビリティ** | ⭐⭐⭐ (3/5) | 歯抜け番号の説明が必要 |
| **拡張性** | ⭐⭐⭐⭐ (4/5) | 削除済み一覧、復元機能追加可 |

**総合スコア**: 27/30 (90%)

---

**次のステップ**: [04_option-2-suffix-approach.md](./04_option-2-suffix-approach.md) で選択肢2を比較検討
