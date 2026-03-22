# 現在の実装状況

**ファイル**: task-manager.html
**バージョン**: v2.13.4
**対象機能**: WBS番号自動生成、タスク削除（論理削除）
**最終更新**: 2026-03-22

---

## 📋 概要

現在の実装では、WBS番号の自動生成時に削除済みタスクを除外していますが、**削除済みタスクを含む全履歴を考慮していない**ため、削除された番号が再利用される可能性があります。

---

## 🔍 関連するコード

### **1. WBS番号自動生成関数** (3322-3351行)

```javascript
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // Find all tasks in the same phase
  const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);

  // Extract numeric parts and find max
  let maxNumber = 0;
  phaseTasks.forEach(t => {
    // Try to extract number from WBS_No (e.g., "PH1.2.3" -> 2.3)
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

**重要な行**:
- **3330行**: `const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);`
  - 削除済みタスク (`!t.deleted`) を除外
  - **問題**: この除外されたタスクリストからmaxNumberを計算している

---

### **2. タスク削除関数** (3242-3285行)

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

  // 論理削除
  task.deleted = true;
  task.deleted_at = new Date().toISOString();

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

**重要なポイント**:
- **論理削除**: `deleted: true` フラグのみ設定
- **WBS番号は変更しない**: `task.wbs_no` はそのまま
- **データは保持**: JSONエクスポートに含まれる

---

### **3. タスク表示のフィルタリング** (1991-2027行)

```javascript
function getFilteredTasks() {
  if (!projectData || !projectData.tasks) return [];

  const searchQuery = document.getElementById('search-input').value.toLowerCase();
  const phaseFilter = document.getElementById('phase-filter').value;
  const statusFilter = document.getElementById('status-filter').value;
  const priorityFilter = document.getElementById('priority-filter').value;

  return projectData.tasks
    .filter(task => {
      // Skip deleted tasks
      if (task.deleted) return false;

      // Search filter (task name or WBS number)
      const matchesSearch = !searchQuery ||
        task.task_name.toLowerCase().includes(searchQuery) ||
        task.wbs_no.toLowerCase().includes(searchQuery);

      // Phase filter
      const matchesPhase = !phaseFilter || task.phase === phaseFilter;

      // Status filter
      const matchesStatus = !statusFilter || task.status === statusFilter;

      // Priority filter
      const taskPriority = task.priority || '中';
      const matchesPriority = !priorityFilter || taskPriority === priorityFilter;

      return matchesSearch && matchesPhase && matchesStatus && matchesPriority;
    })
    .sort((a, b) => {
      // Sort by display_order
      const orderA = a.display_order !== undefined ? a.display_order : 999999;
      const orderB = b.display_order !== undefined ? b.display_order : 999999;
      return orderA - orderB;
    });
}
```

**重要な行**:
- **2002行**: `if (task.deleted) return false;`
  - 削除済みタスクをUI表示から除外

---

## 🎯 現在の動作

### **正常ケース（削除なし）**

```
既存タスク:
- TEST.1.1 (未削除)
- TEST.2.1 (未削除)
- TEST.3.1 (未削除)

新規作成時:
1. phaseTasksに3件すべて含まれる
2. maxNumber = 3 を算出
3. 生成: TEST.4.1 ✅
```

---

### **問題ケース（削除あり）**

```
既存タスク:
- TEST.1.1 (未削除)
- TEST.2.1 (削除済み) ← deleted: true
- TEST.3.1 (未削除)

新規作成時:
1. phaseTasks = [TEST.1.1, TEST.3.1] (削除済み除外)
2. maxNumber = 3 を算出
3. 生成: TEST.4.1 ✅

---

全削除後のケース:
既存タスク:
- TEST.1.1 (削除済み)
- TEST.2.1 (削除済み)
- TEST.3.1 (削除済み)

新規作成時:
1. phaseTasks = [] (空配列)
2. maxNumber = 0
3. 生成: TEST.1.1 ❌ (再利用発生！)
```

---

### **ユーザー報告のケース**

```
既存タスク:
- TEST.15.1 (作成)
- TEST.15.1 (削除済み)

新規作成時:
1. phaseTasks = [] (削除済み除外)
2. maxNumber = 0
3. 生成: TEST.1.1 が提案される
4. ユーザー期待: TEST.16.1 または TEST.15.2
```

---

## ⚠️ 問題点の整理

### **1. 全削除時の番号リセット**
- **状況**: あるフェーズのすべてのタスクを削除
- **現状**: TEST.1.1 から再開
- **期待**: TEST.{削除前の最大+1}.1

### **2. 番号の再利用リスク**
- **状況**: TEST.15.1 を削除後、新規作成
- **現状**: TEST.1.1 が提案される（条件による）
- **期待**: TEST.15.1 は使われない

### **3. 歯抜け番号の発生**
- **状況**: TEST.1, TEST.2, TEST.3 → TEST.2削除
- **現状**: TEST.1, TEST.3 のみ表示（歯抜け）
- **影響**: ユーザーが「TEST.2はどこ？」と疑問

---

## 🔧 現在の実装の良い点

### ✅ **削除済みタスクの除外は実装済み**
- UI表示から正しく除外（1991-2027行）
- CSV出力からも除外（2695-2715行）
- JSONエクスポートには含まれる（データ保持）

### ✅ **論理削除が正常動作**
- `deleted: true` フラグ設定
- `deleted_at` タイムスタンプ記録
- データ損失なし

### ✅ **WBS番号の不変性**
- 削除時にWBS番号を変更しない
- JSONエクスポート・インポートの整合性維持

---

## 🐛 問題の根本原因

**3330行のフィルタリングロジック**:
```javascript
const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);
```

このフィルタリングされた配列から `maxNumber` を算出しているため、削除済みタスクの番号が考慮されません。

**修正すべき箇所**:
- maxNumber算出時に、削除済みタスクも含めた全履歴を考慮する必要がある

---

## 📊 データフロー図

```
ユーザー操作: 新規タスク作成
    ↓
generateWBSNumber() 呼び出し
    ↓
projectData.tasks から同じフェーズのタスクをフィルタ
    ├─ 削除済みタスク: 除外 (!t.deleted)
    └─ 未削除タスク: 含める
    ↓
未削除タスクのみから maxNumber を算出
    ↓
maxNumber + 1 で新しい番号を生成
    ↓
生成された番号を入力フィールドに設定
```

**問題**: 削除済みタスクの番号が maxNumber 計算に含まれない

---

## 🎯 期待される動作

```
既存タスク（削除含む全履歴）:
- TEST.1.1 (未削除)
- TEST.2.1 (削除済み)
- TEST.3.1 (未削除)
- TEST.4.1 (削除済み)
- TEST.5.1 (未削除)

新規作成時:
1. 全タスク（削除含む）から maxNumber = 5 を算出
2. 生成: TEST.6.1 ✅

期待される連番:
TEST.1.1 → TEST.2.1 → TEST.3.1 → ... → TEST.6.1
（削除されたタスクは欠番として残る）
```

---

## 📝 まとめ

### **現状**
- 削除済みタスクをUI表示から除外 ✅
- 論理削除が正常動作 ✅
- WBS番号自動生成が動作 ✅
- **ただし**、削除済みタスクの番号が再利用される可能性あり ❌

### **修正方針**
- maxNumber算出時に、削除済みタスクを含む全履歴を考慮
- 実装コスト: 1-2行の変更
- リスク: 低

---

**次のステップ**: [02_problem-statement.md](./02_problem-statement.md) で具体的な問題シナリオを確認
