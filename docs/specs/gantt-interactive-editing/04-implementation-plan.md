# 実装計画・工数見積もり - ガントチャート・インタラクティブ編集

---
**feature**: ガントチャート・インタラクティブ編集
**document**: 実装計画・工数見積もり
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

ガントチャート日程バーのドラッグ&リサイズ機能の段階的実装計画。Phase分けによるリスク管理と早期フィードバック取得を重視。

---

## 全体スケジュール

### タイムライン

```
Week 1:   Phase 1（基本ドラッグ移動）
Week 1-2: Phase 2（リサイズ機能）
Week 2:   Phase 3（UX改善）
Week 3:   ユーザー検証・バグ修正
```

### フェーズ構成

| フェーズ | 機能 | 工数 | 依存関係 |
|---------|------|------|---------|
| **Phase 1** | 基本ドラッグ移動 | 6-8h | なし |
| **Phase 2** | リサイズ機能 | 5-7h | Phase 1完了後 |
| **Phase 3** | UX改善 | 4-6h | Phase 1, 2完了後 |
| **合計** | | **15-21h (2-3日間)** | |

---

## Phase 1: 基本ドラッグ移動（6-8h）

### 目的

日程バー中央をドラッグして、**期間を維持したまま移動**できる機能を実装。

### 実装内容

#### 1. メイン画面側の準備（1h）

**新規関数追加:**

```javascript
// task-manager.html の <script> セクションに追加

/**
 * ガントチャート画面から呼び出されるタスク日程更新関数
 * @param {string} wbsNo - タスクのWBS番号
 * @param {string} newStartDate - 新開始日 "YYYY-MM-DD"
 * @param {string} newEndDate - 新終了日 "YYYY-MM-DD"
 */
function updateTaskDates(wbsNo, newStartDate, newEndDate) {
  const task = projectData.tasks.find(t => t.wbs_no === wbsNo);

  if (!task) {
    console.error(`タスク ${wbsNo} が見つかりません`);
    return;
  }

  // 日程更新
  task.start_date = newStartDate;
  task.finish_date = newEndDate;

  // 期間を再計算
  task.duration_bd = calculateBusinessDays(newStartDate, newEndDate);

  // コメント追加（変更履歴）
  const now = new Date().toISOString();
  if (!task.comments) task.comments = [];
  task.comments.push({
    timestamp: now,
    text: `ガントチャートで日程変更: ${newStartDate} ～ ${newEndDate}`
  });

  // 保存 & UI更新
  saveToLocalStorage();
  markDirty();
  renderTaskList();

  console.log(`[updateTaskDates] ${wbsNo}: ${newStartDate} ～ ${newEndDate}`);
}
```

#### 2. ガントチャート画面の修正（5-7h）

**修正対象:** `generateGanttHTML()` 関数内のJavaScript部分（約3200行目～）

**2-1. 座標変換関数（1.5h）**

```javascript
// positionToDate() - X座標を日付に変換
function positionToDate(posX, timelineCells, cellWidth) {
  const cellIndex = Math.floor(posX / cellWidth);
  if (cellIndex < 0) return timelineCells[0].date;
  if (cellIndex >= timelineCells.length) {
    return timelineCells[timelineCells.length - 1].date;
  }
  return timelineCells[cellIndex].date;
}

// dateToPosition() - 日付をX座標に変換
function dateToPosition(dateStr, timelineCells, cellWidth) {
  const index = timelineCells.findIndex(cell => cell.date === dateStr);
  if (index === -1) return 0;
  return index * cellWidth;
}

// shiftDate() - 日付をN日シフト
function shiftDate(dateStr, shiftCells, cells) {
  const currentIndex = cells.findIndex(c => c.date === dateStr);
  if (currentIndex === -1) return dateStr;

  const newIndex = currentIndex + shiftCells;
  if (newIndex < 0) return cells[0].date;
  if (newIndex >= cells.length) return cells[cells.length - 1].date;

  return cells[newIndex].date;
}
```

**2-2. ドラッグイベント処理（2-3h）**

```javascript
let dragState = null;

function attachDragBehavior(barElement, task) {
  barElement.style.cursor = 'move';

  barElement.addEventListener('mousedown', (e) => {
    e.preventDefault();

    const rect = barElement.getBoundingClientRect();

    dragState = {
      mode: 'MOVE',
      wbsNo: task.wbs_no,
      originalStartDate: task.start_date,
      originalEndDate: task.finish_date,
      startX: e.clientX,
      barLeft: rect.left,
      barWidth: rect.width
    };

    barElement.classList.add('dragging');
    createPreviewBar(barElement);

    document.body.style.cursor = 'move';
  });
}

document.addEventListener('mousemove', (e) => {
  if (!dragState) return;

  const deltaX = e.clientX - dragState.startX;
  const cells = getCurrentTimelineCells();
  const cellWidth = getCellWidth();
  const deltaCells = Math.round(deltaX / cellWidth);

  const newStartDate = shiftDate(
    dragState.originalStartDate, deltaCells, cells
  );
  const newEndDate = shiftDate(
    dragState.originalEndDate, deltaCells, cells
  );

  updatePreview(newStartDate, newEndDate);
});

document.addEventListener('mouseup', (e) => {
  if (!dragState) return;

  const previewBar = document.querySelector('.task-bar-preview');
  const finalStartDate = previewBar.dataset.startDate;
  const finalEndDate = previewBar.dataset.endDate;

  applyDateChange(dragState.wbsNo, finalStartDate, finalEndDate);

  cleanup();
});
```

**2-3. プレビュー表示（1.5h）**

```javascript
function createPreviewBar(originalBar) {
  const preview = originalBar.cloneNode(true);
  preview.classList.remove('task-bar');
  preview.classList.add('task-bar-preview');
  preview.style.border = '2px dashed #2E7D32';
  preview.style.opacity = '0.8';

  originalBar.parentElement.appendChild(preview);
}

function updatePreview(newStartDate, newEndDate) {
  const preview = document.querySelector('.task-bar-preview');
  const cells = getCurrentTimelineCells();
  const cellWidth = getCellWidth();

  const newLeft = dateToPosition(newStartDate, cells, cellWidth);
  const newRight = dateToPosition(newEndDate, cells, cellWidth);
  const newWidth = newRight - newLeft + cellWidth;

  preview.style.left = `${newLeft}px`;
  preview.style.width = `${newWidth}px`;
  preview.dataset.startDate = newStartDate;
  preview.dataset.endDate = newEndDate;

  updateDateTooltip(newStartDate, newEndDate);
}

function updateDateTooltip(startDate, endDate) {
  let tooltip = document.querySelector('.date-preview-tooltip');
  if (!tooltip) {
    tooltip = document.createElement('div');
    tooltip.className = 'date-preview-tooltip';
    document.body.appendChild(tooltip);
  }

  const duration = calculateDuration(startDate, endDate);
  tooltip.textContent = `${startDate} ～ ${endDate} (${duration}日間)`;

  const preview = document.querySelector('.task-bar-preview');
  const rect = preview.getBoundingClientRect();
  tooltip.style.left = `${rect.left + rect.width / 2 - 50}px`;
  tooltip.style.top = `${rect.top - 35}px`;
}
```

**2-4. メイン画面への反映（1h）**

```javascript
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  if (!window.opener || window.opener.closed) {
    alert('メインウィンドウが閉じられています');
    return;
  }

  window.opener.updateTaskDates(wbsNo, newStartDate, newEndDate);
  window.opener.focus();

  setTimeout(() => {
    renderGantt();
  }, 200);

  console.log(`タスク ${wbsNo} の日程を更新: ${newStartDate} ～ ${newEndDate}`);
}

function cleanup() {
  const draggingBars = document.querySelectorAll('.task-bar.dragging');
  draggingBars.forEach(bar => bar.classList.remove('dragging'));

  const previewBar = document.querySelector('.task-bar-preview');
  if (previewBar) previewBar.remove();

  const tooltip = document.querySelector('.date-preview-tooltip');
  if (tooltip) tooltip.remove();

  document.body.style.cursor = 'default';

  dragState = null;
}
```

### 成功基準

- ✅ 日程バー中央をドラッグして移動できる
- ✅ ドラッグ中にプレビューが表示される
- ✅ 移動後の日程がメイン画面に反映される
- ✅ ガントチャートが自動更新される

### 工数内訳

| タスク | 工数 |
|-------|------|
| メイン画面側 updateTaskDates() 実装 | 1h |
| 座標変換関数（3つ） | 1.5h |
| ドラッグイベント処理 | 2-3h |
| プレビュー表示 | 1.5h |
| テスト・デバッグ | 1h |
| **合計** | **6-8h** |

---

## Phase 2: リサイズ機能（5-7h）

### 目的

日程バーの左端・右端をドラッグして、**開始日・終了日を個別に変更**できる機能を実装。

### 実装内容

#### 1. 領域判定機能（1.5h）

```javascript
const RESIZE_HANDLE_WIDTH = 8; // 8px

function getBarZone(event, barElement) {
  const rect = barElement.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const width = rect.width;

  if (x < RESIZE_HANDLE_WIDTH) {
    return 'START_RESIZE';
  } else if (x > width - RESIZE_HANDLE_WIDTH) {
    return 'END_RESIZE';
  } else {
    return 'MOVE';
  }
}

function getCursorStyle(zone) {
  switch(zone) {
    case 'START_RESIZE':
    case 'END_RESIZE':
      return 'col-resize';
    case 'MOVE':
      return 'move';
    default:
      return 'default';
  }
}
```

#### 2. カーソル動的変更（1h）

```javascript
function attachDragBehavior(barElement, task) {
  // ホバー時のカーソル変更
  barElement.addEventListener('mousemove', (e) => {
    if (dragState) return;

    const zone = getBarZone(e, barElement);
    barElement.style.cursor = getCursorStyle(zone);
  });

  // mousedown時に mode を設定
  barElement.addEventListener('mousedown', (e) => {
    const zone = getBarZone(e, barElement);

    dragState = {
      mode: zone, // 'START_RESIZE' | 'END_RESIZE' | 'MOVE'
      // ...
    };

    document.body.style.cursor = getCursorStyle(zone);
  });
}
```

#### 3. リサイズロジック（2-3h）

```javascript
function updatePreviewByMode(dragState, deltaX) {
  const cells = getCurrentTimelineCells();
  const cellWidth = getCellWidth();
  const deltaCells = Math.round(deltaX / cellWidth);

  let newStartDate, newEndDate;

  switch (dragState.mode) {
    case 'START_RESIZE':
      // 開始日のみ変更
      newStartDate = shiftDate(
        dragState.originalStartDate, deltaCells, cells
      );
      newEndDate = dragState.originalEndDate;

      // 最小期間チェック（1日以上）
      if (new Date(newStartDate) >= new Date(newEndDate)) {
        newStartDate = shiftDate(newEndDate, -1, cells);
      }
      break;

    case 'END_RESIZE':
      // 終了日のみ変更
      newStartDate = dragState.originalStartDate;
      newEndDate = shiftDate(
        dragState.originalEndDate, deltaCells, cells
      );

      // 最小期間チェック
      if (new Date(newEndDate) <= new Date(newStartDate)) {
        newEndDate = shiftDate(newStartDate, 1, cells);
      }
      break;

    case 'MOVE':
      // 期間維持して移動（Phase 1で実装済み）
      newStartDate = shiftDate(
        dragState.originalStartDate, deltaCells, cells
      );
      newEndDate = shiftDate(
        dragState.originalEndDate, deltaCells, cells
      );
      break;
  }

  updatePreview(newStartDate, newEndDate);
}
```

#### 4. 最小期間制約（1h）

```javascript
function validateMinimumDuration(startDate, endDate) {
  const start = new Date(startDate);
  const end = new Date(endDate);

  if (end <= start) {
    alert('⚠ タスクは最低1日の期間が必要です');
    return false;
  }

  return true;
}

// mouseup時に呼び出し
function handleMouseUp(e) {
  if (!dragState) return;

  const previewBar = document.querySelector('.task-bar-preview');
  const finalStartDate = previewBar.dataset.startDate;
  const finalEndDate = previewBar.dataset.endDate;

  if (!validateMinimumDuration(finalStartDate, finalEndDate)) {
    cleanup(); // 元に戻す
    return;
  }

  applyDateChange(dragState.wbsNo, finalStartDate, finalEndDate);
  cleanup();
}
```

### 成功基準

- ✅ 左端ドラッグで開始日を変更できる
- ✅ 右端ドラッグで終了日を変更できる
- ✅ ホバー時にカーソルが適切に変化する
- ✅ 最低1日の期間制約が機能する

### 工数内訳

| タスク | 工数 |
|-------|------|
| 領域判定機能 | 1.5h |
| カーソル動的変更 | 1h |
| リサイズロジック | 2-3h |
| 最小期間制約 | 1h |
| テスト・デバッグ | 1h |
| **合計** | **5-7h** |

---

## Phase 3: UX改善（4-6h）

### 目的

視覚的フィードバック、エラーハンドリング、整合性チェックを追加し、実用レベルに引き上げる。

### 実装内容

#### 1. 先行タスク依存チェック（2-3h）

```javascript
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

// mouseup時に呼び出し
function handleMouseUp(e) {
  // ...

  const task = window.opener.projectData.tasks.find(
    t => t.wbs_no === dragState.wbsNo
  );

  if (!validateDependencyConstraints(task, finalStartDate, finalEndDate)) {
    cleanup();
    return;
  }

  applyDateChange(dragState.wbsNo, finalStartDate, finalEndDate);
  cleanup();
}
```

#### 2. 通知UI（1h）

```javascript
function showNotification(message, type = 'success') {
  const toast = document.createElement('div');
  toast.className = `notification notification-${type}`;
  toast.textContent = message;

  document.body.appendChild(toast);

  setTimeout(() => {
    toast.classList.add('fade-out');
    setTimeout(() => toast.remove(), 300);
  }, 3000);
}

// applyDateChange内で呼び出し
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  // ...

  window.opener.updateTaskDates(wbsNo, newStartDate, newEndDate);

  showNotification(`タスク ${wbsNo} の日程を更新しました`, 'success');

  // ...
}
```

#### 3. CSS追加（1h）

```css
/* 日程バー */
.task-bar {
  transition: opacity 0.2s;
}

.task-bar.dragging {
  opacity: 0.3;
  pointer-events: none;
}

/* プレビューバー */
.task-bar-preview {
  border: 2px dashed #2E7D32;
  opacity: 0.8;
  pointer-events: none;
  z-index: 100;
}

/* 日付ツールチップ */
.date-preview-tooltip {
  position: absolute;
  background: #333;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 11px;
  z-index: 101;
}

/* 通知 */
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

@keyframes slideIn {
  from { transform: translateX(400px); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}
```

#### 4. エラーハンドリング強化（1-2h）

```javascript
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  try {
    if (!window.opener || window.opener.closed) {
      throw new Error('メインウィンドウが閉じられています');
    }

    if (!validateBasicConstraints(newStartDate, newEndDate)) {
      return;
    }

    const task = window.opener.projectData.tasks.find(
      t => t.wbs_no === wbsNo
    );
    if (!task) {
      throw new Error(`タスク ${wbsNo} が見つかりません`);
    }

    if (!validateDependencyConstraints(task, newStartDate, newEndDate)) {
      return;
    }

    window.opener.updateTaskDates(wbsNo, newStartDate, newEndDate);

    showNotification(`タスク ${wbsNo} の日程を更新しました`, 'success');

  } catch (error) {
    console.error('[applyDateChange] エラー:', error);
    showNotification(`日程変更に失敗: ${error.message}`, 'error');
    cleanup();
  }
}
```

### 成功基準

- ✅ 先行タスクとの矛盾を警告表示できる
- ✅ 成功/エラー通知が表示される
- ✅ プレビュー表示がスムーズ
- ✅ エラー時に適切にロールバックされる

### 工数内訳

| タスク | 工数 |
|-------|------|
| 先行タスク依存チェック | 2-3h |
| 通知UI実装 | 1h |
| CSS追加 | 1h |
| エラーハンドリング強化 | 1-2h |
| 統合テスト | 1h |
| **合計** | **4-6h** |

---

## リスク管理

### 技術的リスク

| リスク | 影響度 | 対策 |
|--------|--------|------|
| 表示モード（Day/Week/Month）での座標計算の複雑化 | 中 | モード別の変換関数を分離実装 |
| ガントチャート画面とメイン画面の同期タイミング | 中 | mouseup時に即座に反映、5秒自動更新は継続 |
| 営業日計算の精度 | 低 | 既存の`calculateBusinessDays`関数を流用 |

### スケジュールリスク

| リスク | 影響度 | 対策 |
|--------|--------|------|
| Phase 1の遅延 | 高 | Phase 1を最優先、Phase 2/3は後回し可 |
| ユーザー検証での仕様変更要求 | 中 | MVPリリース後のフィードバックで判断 |

---

## テスト計画

### 単体テスト

| 関数 | テストケース |
|------|------------|
| `getBarZone()` | 左端8px / 中央 / 右端8px |
| `positionToDate()` | 範囲内 / 範囲外（負） / 範囲外（超過） |
| `dateToPosition()` | 有効な日付 / 無効な日付 |
| `shiftDate()` | プラス / マイナス / 範囲外 |

### 統合テスト

| シナリオ | 確認項目 |
|---------|---------|
| 左端リサイズ | 開始日変更、終了日不変 |
| 右端リサイズ | 終了日変更、開始日不変 |
| 中央ドラッグ | 期間維持、両日付変更 |
| 最小期間違反 | エラー表示、元に戻る |
| 先行タスク矛盾 | 警告表示、確認ダイアログ |
| メイン画面同期 | タスクリストが更新される |

---

## デプロイ計画

### Phase 1完了後

1. ローカル環境でテスト
2. テストユーザーへのデモ
3. フィードバック収集
4. Phase 2へ進むか判断

### Phase 2完了後

1. Phase 1 + 2 の統合テスト
2. エッジケーステスト
3. Phase 3へ進むか判断

### Phase 3完了後（正式リリース）

1. 全機能の統合テスト
2. CHANGELOG.md 更新
3. バージョン番号更新（v2.13.0）
4. Git commit & push
5. GitHub Pagesへの反映確認

---

## 次のアクション

### 即座に実施

1. ✅ 仕様書のレビュー（本ドキュメント）
2. → Phase 1実装開始

### Phase 1完了後

1. Phase 1のユーザー検証
2. Phase 2実装開始

### Phase 2完了後

1. Phase 2のユーザー検証
2. Phase 3実装開始

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要
- [01-ui-ux-design.md](./01-ui-ux-design.md) - UI/UX設計
- [02-technical-design.md](./02-technical-design.md) - 技術実装詳細
- [03-data-synchronization.md](./03-data-synchronization.md) - データ同期仕様

---

**最終更新**: 2026-03-21
**ステータス**: 仕様確定、Phase 1実装待ち
**総工数**: 15-21h（2-3日間）
