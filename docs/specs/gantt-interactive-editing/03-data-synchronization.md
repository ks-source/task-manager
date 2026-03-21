# データ同期・整合性管理 - ガントチャート・インタラクティブ編集

---
**feature**: ガントチャート・インタラクティブ編集
**document**: データ同期・整合性管理
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

ガントチャート画面とメイン画面間のデータ同期メカニズム、整合性チェック、競合解決方針を定義。

---

## ウィンドウ間通信アーキテクチャ

### 通信フロー全体像

```
┌────────────────────────────────────────────────────────┐
│          メイン画面 (task-manager.html)                 │
│                                                        │
│  projectData.tasks = [                                │
│    { wbs_no: "WBS1.1.0",                              │
│      start_date: "2026-03-02",                        │
│      finish_date: "2026-03-08",                       │
│      duration_bd: 7 },                                │
│    ...                                                │
│  ]                                                    │
│                                                        │
│  updateTaskDates(wbs, start, end) { ★新規関数         │
│    // タスク更新                                       │
│    // LocalStorage保存                                │
│    // UI再描画                                         │
│  }                                                    │
└───────────────────┬────────────────────────────────────┘
                    │
                    │ window.open()
                    ↓
┌────────────────────────────────────────────────────────┐
│       ガントチャート画面 (独立ウィンドウ)                │
│                                                        │
│  window.opener → メイン画面への参照                     │
│                                                        │
│  データ取得（5秒ごと自動更新）:                          │
│    window.opener.projectData.tasks                    │
│                                                        │
│  データ送信（mouseup時）:                               │
│    window.opener.updateTaskDates(wbs, start, end)     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## データフロー詳細

### 1. 初期表示時

```
[ステップ1] ユーザーがメイン画面で「ガントチャート」ボタンクリック
              ↓
[ステップ2] openGanttChart() 実行
              ↓
[ステップ3] window.open() で新ウィンドウ作成
              ↓
[ステップ4] generateGanttHTML() でHTML生成
              ↓
[ステップ5] ガントチャート画面で renderGantt() 実行
              ↓
[ステップ6] window.opener.projectData.tasks からデータ取得
              ↓
[ステップ7] タイムライン描画完了
```

---

### 2. 日程編集時

```
[ステップ1] ユーザーが日程バーをドラッグ
              ↓
[ステップ2] mousedown → dragState 初期化
              ↓
[ステップ3] mousemove → プレビュー更新（ガントチャート画面内）
              ↓
[ステップ4] mouseup → 最終日付確定
              ↓
[ステップ5] applyDateChange() 実行
              ↓
         ┌──┴───────────────────────────┐
         │                              │
[ステップ6A] 整合性チェック             │
         │  ・最小期間チェック          │
         │  ・先行タスク依存チェック     │
         │  ・タイムライン範囲チェック   │
         │                              │
         │  NG → alert表示、元に戻る    │
         │  OK → 次へ                   │
         └──┬───────────────────────────┘
            │
[ステップ7] window.opener.updateTaskDates(wbs, start, end)
              ↓
[ステップ8] メイン画面でタスクデータ更新
              ↓
[ステップ9] saveToLocalStorage()
              ↓
[ステップ10] markDirty()
              ↓
[ステップ11] renderTaskList()
              ↓
[ステップ12] ガントチャート画面で renderGantt() 再実行
              ↓
[完了] 両画面が同期された状態
```

---

### 3. 自動更新時（5秒ごと）

```
[タイマー起動] setInterval(refreshGanttData, 5000)
              ↓
[チェック1] window.opener が存在するか？
              ↓ YES
[チェック2] window.opener.closed === false？
              ↓ YES
[データ取得] window.opener.projectData.tasks
              ↓
[再描画] renderGantt()
              ↓
[完了] ガントチャート更新完了

※ メイン画面が閉じられた場合は自動更新停止
```

---

## データ整合性チェック

### チェックレベル

| レベル | タイミング | 内容 |
|--------|----------|------|
| **Lv1: 基本** | mouseup直前 | 最小期間（1日以上）チェック |
| **Lv2: 警告** | mouseup直前 | 先行タスク依存関係チェック |
| **Lv3: 範囲** | mousemove中 | タイムライン表示範囲チェック |

---

### Lv1: 基本チェック（エラー扱い）

```javascript
/**
 * 最小期間チェック
 * @returns {boolean} true: OK, false: NG（保存中止）
 */
function validateBasicConstraints(startDate, endDate) {
  const start = new Date(startDate);
  const end = new Date(endDate);

  // 1日以上の期間が必要
  if (end <= start) {
    alert('⚠ エラー: タスクは最低1日の期間が必要です');
    return false;
  }

  // 日付フォーマットチェック
  if (isNaN(start.getTime()) || isNaN(end.getTime())) {
    alert('⚠ エラー: 無効な日付形式です');
    return false;
  }

  return true;
}
```

---

### Lv2: 警告チェック（ユーザー確認）

```javascript
/**
 * 先行タスク依存関係チェック
 * @returns {boolean} true: 続行、false: キャンセル
 */
function validateDependencyConstraints(task, newStartDate, newEndDate) {
  const warnings = [];

  // 先行タスクチェック
  if (task.predecessors && task.predecessors.trim() !== '') {
    const predWbs = task.predecessors.split(',')[0].trim();
    const predTask = window.opener.projectData.tasks.find(
      t => t.wbs_no === predWbs
    );

    if (predTask && predTask.finish_date) {
      const predEnd = new Date(predTask.finish_date);
      const newStart = new Date(newStartDate);

      if (newStart < predEnd) {
        warnings.push(
          `先行タスク ${predWbs} の終了日（${predTask.finish_date}）より前に開始します`
        );
      }
    }
  }

  // 後続タスクチェック
  const successors = findSuccessorTasks(task.wbs_no);
  successors.forEach(succTask => {
    if (succTask.start_date) {
      const succStart = new Date(succTask.start_date);
      const newEnd = new Date(newEndDate);

      if (newEnd > succStart) {
        warnings.push(
          `後続タスク ${succTask.wbs_no} の開始日（${succTask.start_date}）より後に終了します`
        );
      }
    }
  });

  // 警告がある場合は確認ダイアログ
  if (warnings.length > 0) {
    const message = [
      '⚠ 以下の依存関係に矛盾があります:\n',
      ...warnings.map(w => `  ・${w}`),
      '\nこのまま保存しますか？'
    ].join('\n');

    return confirm(message);
  }

  return true;
}

/**
 * 指定タスクを先行タスクとする後続タスクを検索
 */
function findSuccessorTasks(wbsNo) {
  return window.opener.projectData.tasks.filter(task => {
    if (!task.predecessors) return false;
    const preds = task.predecessors.split(',').map(p => p.trim());
    return preds.includes(wbsNo);
  });
}
```

---

### Lv3: 範囲チェック（リアルタイム制限）

```javascript
/**
 * タイムライン表示範囲内に収まるかチェック
 * mousemove中に呼び出し、範囲外へのドラッグを制限
 */
function constrainToTimeline(dateStr) {
  const timelineCells = getCurrentTimelineCells();
  const minDate = timelineCells[0].date;
  const maxDate = timelineCells[timelineCells.length - 1].date;

  const date = new Date(dateStr);
  const min = new Date(minDate);
  const max = new Date(maxDate);

  if (date < min) return minDate;
  if (date > max) return maxDate;

  return dateStr;
}
```

---

## 競合解決方針

### シナリオ1: メイン画面で同時編集

```
[状況]
ガントチャート画面でドラッグ中に、
メイン画面でも同じタスクを編集

[方針]
後勝ち（Last Write Wins）

[理由]
・編集頻度が低いため競合リスクは低い
・楽観的ロック実装コストが高い
・ユーザーは通常1画面で操作

[動作]
1. ガントチャート画面で mouseup
2. メイン画面のデータを上書き
3. メイン画面の変更は失われる

[今後の拡張案]
変更検出機能（タイムスタンプ比較）を追加し、
競合時に警告表示
```

---

### シナリオ2: メイン画面が閉じられた場合

```
[状況]
ガントチャート画面表示中にメイン画面を閉じる

[方針]
ガントチャート画面は読み取り専用モードに移行

[動作]
1. 5秒自動更新で window.opener.closed を検出
2. 編集機能を無効化（ドラッグ不可）
3. エラーメッセージ表示:
   「⚠ メインウィンドウが閉じられました。編集機能は無効です。」

[実装]
function checkMainWindowStatus() {
  if (!window.opener || window.opener.closed) {
    disableEditMode();
    showWarningBanner('メインウィンドウが閉じられました');
    clearInterval(autoRefreshTimer);
  }
}
```

---

## データ永続化

### LocalStorage保存タイミング

| トリガー | 関数 | 備考 |
|---------|------|------|
| ガントチャートで日程変更 | `updateTaskDates()` 内で `saveToLocalStorage()` | 即座に保存 |
| メイン画面でタスク編集 | `saveTaskDetail()` 内で `saveToLocalStorage()` | 既存動作 |
| 5秒自動更新 | なし | 読み取りのみ |

### ダーティフラグ管理

```javascript
// メイン画面（task-manager.html）
let isDirty = false;

function updateTaskDates(wbsNo, newStartDate, newEndDate) {
  // ... タスク更新処理 ...

  saveToLocalStorage();
  markDirty(); // ダーティフラグON

  // ブラウザ閉じる時の警告
  window.onbeforeunload = () => {
    if (isDirty) {
      return '未保存の変更があります';
    }
  };
}
```

---

## エラーハンドリング

### エラー種別と対応

| エラー種別 | 発生タイミング | 対応 |
|-----------|--------------|------|
| **通信エラー** | `window.opener` 参照時 | エラーメッセージ表示、編集モード無効化 |
| **データ不整合** | 日付計算時 | 警告表示、元の値に戻す |
| **制約違反** | mouseup時 | アラート表示、保存中止 |

### 実装例

```javascript
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  try {
    // 1. メインウィンドウ存在チェック
    if (!window.opener || window.opener.closed) {
      throw new Error('メインウィンドウが閉じられています');
    }

    // 2. 基本制約チェック
    if (!validateBasicConstraints(newStartDate, newEndDate)) {
      return; // アラート表示済み、処理中止
    }

    // 3. タスク存在チェック
    const task = window.opener.projectData.tasks.find(
      t => t.wbs_no === wbsNo
    );
    if (!task) {
      throw new Error(`タスク ${wbsNo} が見つかりません`);
    }

    // 4. 依存関係チェック（警告）
    if (!validateDependencyConstraints(task, newStartDate, newEndDate)) {
      return; // ユーザーがキャンセル
    }

    // 5. メイン画面へ反映
    window.opener.updateTaskDates(wbsNo, newStartDate, newEndDate);

    // 6. 成功通知
    showNotification(`タスク ${wbsNo} の日程を更新しました`);

  } catch (error) {
    console.error('[applyDateChange] エラー:', error);
    alert(`日程変更に失敗しました: ${error.message}`);

    // エラー時は元の状態に戻す
    cleanup();
  }
}
```

---

## 通知UI

### トースト通知

```javascript
/**
 * 成功/エラー通知を表示
 * @param {string} message - メッセージ
 * @param {string} type - 'success' | 'error' | 'warning'
 */
function showNotification(message, type = 'success') {
  const toast = document.createElement('div');
  toast.className = `notification notification-${type}`;
  toast.textContent = message;

  document.body.appendChild(toast);

  // 3秒後に自動削除
  setTimeout(() => {
    toast.classList.add('fade-out');
    setTimeout(() => toast.remove(), 300);
  }, 3000);
}
```

### CSS

```css
.notification {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 12px 20px;
  border-radius: 4px;
  font-size: 14px;
  z-index: 9999;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  animation: slideIn 0.3s ease-out;
}

.notification-success {
  background: #4CAF50;
  color: white;
}

.notification-error {
  background: #f44336;
  color: white;
}

.notification-warning {
  background: #ff9800;
  color: white;
}

.notification.fade-out {
  animation: fadeOut 0.3s ease-out;
}

@keyframes slideIn {
  from { transform: translateX(400px); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}

@keyframes fadeOut {
  from { opacity: 1; }
  to { opacity: 0; }
}
```

---

## テストシナリオ

### 同期テスト

| テストケース | 期待動作 |
|------------|---------|
| ガントで編集 → メイン画面確認 | タスクリストの日付が更新されている |
| メイン画面で編集 → ガント自動更新 | 5秒以内にガントチャートに反映 |
| 両画面で同時編集 | 後勝ち（最後の編集が優先） |

### 整合性テスト

| テストケース | 期待動作 |
|------------|---------|
| 終了日 < 開始日 に変更 | エラー表示、保存中止 |
| 先行タスクより前に開始 | 警告表示、ユーザー確認 |
| タイムライン範囲外へドラッグ | 範囲端で停止 |

### エラーハンドリングテスト

| テストケース | 期待動作 |
|------------|---------|
| メイン画面を閉じてから編集 | エラーメッセージ、編集無効化 |
| 無効なタスクIDを指定 | エラーメッセージ表示 |
| ネットワーク切断状態で編集 | LocalStorageへの保存は成功 |

---

## パフォーマンス考慮

### 自動更新の最適化

```javascript
// 差分検出による無駄な再描画の抑制
let lastDataHash = '';

function refreshGanttData() {
  if (!window.opener || window.opener.closed) return;

  const tasks = window.opener.projectData.tasks;
  const currentHash = generateHash(tasks);

  // データが変更されていない場合はスキップ
  if (currentHash === lastDataHash) {
    console.log('[refreshGanttData] データ変更なし、スキップ');
    return;
  }

  lastDataHash = currentHash;
  renderGantt(); // 再描画
}

function generateHash(data) {
  return JSON.stringify(data.map(t => ({
    wbs: t.wbs_no,
    start: t.start_date,
    end: t.finish_date
  })));
}
```

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要
- [01-ui-ux-design.md](./01-ui-ux-design.md) - UI/UX設計
- [02-technical-design.md](./02-technical-design.md) - 技術実装詳細
- [04-implementation-plan.md](./04-implementation-plan.md) - 実装計画

---

**最終更新**: 2026-03-21
**ステータス**: 仕様確定、実装待ち
