# 技術設計・実装方針 - ガントチャート・インタラクティブ編集

---
**feature**: ガントチャート・インタラクティブ編集
**document**: 技術設計
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
---

## 概要

ガントチャート日程バーのドラッグ&リサイズ機能の技術実装詳細。座標変換、イベント処理、ウィンドウ間通信を中心に設計。

---

## アーキテクチャ概要

### コンポーネント構成

```
┌─────────────────────────────────────────────────────────┐
│         task-manager.html (メインウィンドウ)              │
├─────────────────────────────────────────────────────────┤
│  ・タスクデータ管理 (projectData.tasks)                  │
│  ・updateTaskDates() ← 新規追加                         │
│  ・saveToLocalStorage()                                │
│  ・renderTaskList()                                    │
└─────────────────────────────────────────────────────────┘
                        ↑
                        │ window.opener 通信
                        │
┌─────────────────────────────────────────────────────────┐
│    ガントチャートウィンドウ (generateGanttHTML内)          │
├─────────────────────────────────────────────────────────┤
│  【既存機能】                                            │
│  ・renderGantt() - タイムライン描画                      │
│  ・calculateTimelineCells() - 日付セル計算               │
│                                                         │
│  【新規追加機能】                                         │
│  ・attachDragBehavior() - ドラッグイベント設定            │
│  ・getBarZone() - 領域判定                              │
│  ・positionToDate() - 座標→日付変換                     │
│  ・updatePreview() - プレビュー更新                      │
│  ・applyDateChange() - メイン画面へ反映                  │
└─────────────────────────────────────────────────────────┘
```

---

## 主要関数設計

### 1. getBarZone() - インタラクション領域判定

```javascript
/**
 * 日程バー内のクリック位置を判定
 * @param {MouseEvent} event - マウスイベント
 * @param {HTMLElement} barElement - 日程バー要素
 * @returns {string} 'START_RESIZE' | 'END_RESIZE' | 'MOVE'
 */
function getBarZone(event, barElement) {
  const rect = barElement.getBoundingClientRect();
  const x = event.clientX - rect.left;
  const width = rect.width;

  const RESIZE_HANDLE_WIDTH = 8; // 8px

  if (x < RESIZE_HANDLE_WIDTH) {
    return 'START_RESIZE';  // 左端リサイズ
  } else if (x > width - RESIZE_HANDLE_WIDTH) {
    return 'END_RESIZE';    // 右端リサイズ
  } else {
    return 'MOVE';          // 中央ドラッグ移動
  }
}
```

---

### 2. positionToDate() - 座標→日付変換

```javascript
/**
 * タイムライン上のX座標を日付に変換
 * @param {number} posX - X座標（タイムライン基準）
 * @param {Array} timelineCells - 日付セル配列
 * @param {number} cellWidth - セル幅
 * @returns {string} 日付文字列 "YYYY-MM-DD"
 */
function positionToDate(posX, timelineCells, cellWidth) {
  const cellIndex = Math.floor(posX / cellWidth);

  // 範囲外チェック
  if (cellIndex < 0) return timelineCells[0].date;
  if (cellIndex >= timelineCells.length) {
    return timelineCells[timelineCells.length - 1].date;
  }

  return timelineCells[cellIndex].date;
}
```

---

### 3. dateToPosition() - 日付→座標変換

```javascript
/**
 * 日付をタイムライン上のX座標に変換
 * @param {string} dateStr - 日付文字列 "YYYY-MM-DD"
 * @param {Array} timelineCells - 日付セル配列
 * @param {number} cellWidth - セル幅
 * @returns {number} X座標
 */
function dateToPosition(dateStr, timelineCells, cellWidth) {
  const index = timelineCells.findIndex(cell => cell.date === dateStr);

  if (index === -1) {
    console.warn(`日付 ${dateStr} がタイムライン範囲外です`);
    return 0;
  }

  return index * cellWidth;
}
```

---

### 4. attachDragBehavior() - ドラッグイベント設定

```javascript
/**
 * 日程バーにドラッグ&リサイズ機能を追加
 * @param {HTMLElement} barElement - 日程バー要素
 * @param {Object} task - タスクデータ
 */
function attachDragBehavior(barElement, task) {
  let dragState = null;

  // ホバー時のカーソル変更
  barElement.addEventListener('mousemove', (e) => {
    if (dragState) return; // ドラッグ中は無視

    const zone = getBarZone(e, barElement);
    barElement.style.cursor = getCursorStyle(zone);
  });

  // ドラッグ開始
  barElement.addEventListener('mousedown', (e) => {
    e.preventDefault();

    const zone = getBarZone(e, barElement);
    const rect = barElement.getBoundingClientRect();

    dragState = {
      mode: zone,
      wbsNo: task.wbs_no,
      originalStartDate: task.start_date,
      originalEndDate: task.finish_date,
      startX: e.clientX,
      barLeft: rect.left,
      barWidth: rect.width
    };

    // 半透明化
    barElement.classList.add('dragging');

    // プレビュー要素作成
    createPreviewBar(barElement);

    document.body.style.cursor = getCursorStyle(zone);
  });

  // ドラッグ中（ドキュメントレベルで監視）
  document.addEventListener('mousemove', handleMouseMove);

  // ドラッグ終了
  document.addEventListener('mouseup', handleMouseUp);
}
```

---

### 5. handleMouseMove() - ドラッグ中の処理

```javascript
let lastMoveTime = 0;
const THROTTLE_MS = 16; // 約60fps

function handleMouseMove(e) {
  if (!dragState) return;

  // スロットリング
  const now = Date.now();
  if (now - lastMoveTime < THROTTLE_MS) return;
  lastMoveTime = now;

  const deltaX = e.clientX - dragState.startX;
  const deltaCells = Math.round(deltaX / cellWidth);

  updatePreview(dragState, deltaCells);
}
```

---

### 6. updatePreview() - プレビュー更新

```javascript
/**
 * ドラッグ中のプレビュー表示を更新
 * @param {Object} dragState - ドラッグ状態
 * @param {number} deltaCells - セル単位の移動量
 */
function updatePreview(dragState, deltaCells) {
  const { mode, originalStartDate, originalEndDate } = dragState;

  let newStartDate, newEndDate;

  // タイムラインセル配列と日付計算関数を取得
  const cells = getCurrentTimelineCells();

  switch (mode) {
    case 'START_RESIZE':
      // 開始日のみ変更
      newStartDate = shiftDate(originalStartDate, deltaCells, cells);
      newEndDate = originalEndDate;

      // 最小期間チェック（1日以上）
      if (new Date(newStartDate) >= new Date(newEndDate)) {
        newStartDate = shiftDate(newEndDate, -1, cells);
      }
      break;

    case 'END_RESIZE':
      // 終了日のみ変更
      newStartDate = originalStartDate;
      newEndDate = shiftDate(originalEndDate, deltaCells, cells);

      // 最小期間チェック
      if (new Date(newEndDate) <= new Date(newStartDate)) {
        newEndDate = shiftDate(newStartDate, 1, cells);
      }
      break;

    case 'MOVE':
      // 期間維持して移動
      newStartDate = shiftDate(originalStartDate, deltaCells, cells);
      newEndDate = shiftDate(originalEndDate, deltaCells, cells);
      break;
  }

  // プレビューバーの位置・幅を更新
  const previewBar = document.querySelector('.task-bar-preview');
  const newLeft = dateToPosition(newStartDate, cells, cellWidth);
  const newRight = dateToPosition(newEndDate, cells, cellWidth);
  const newWidth = newRight - newLeft + cellWidth;

  previewBar.style.left = `${newLeft}px`;
  previewBar.style.width = `${newWidth}px`;

  // 日付ツールチップ更新
  updateDateTooltip(newStartDate, newEndDate);
}
```

---

### 7. handleMouseUp() - ドラッグ終了処理

```javascript
function handleMouseUp(e) {
  if (!dragState) return;

  // プレビューから最終日付を取得
  const previewBar = document.querySelector('.task-bar-preview');
  const finalStartDate = previewBar.dataset.startDate;
  const finalEndDate = previewBar.dataset.endDate;

  // メイン画面へ反映
  applyDateChange(dragState.wbsNo, finalStartDate, finalEndDate);

  // クリーンアップ
  cleanup();
}

function cleanup() {
  // 半透明解除
  const draggingBars = document.querySelectorAll('.task-bar.dragging');
  draggingBars.forEach(bar => bar.classList.remove('dragging'));

  // プレビュー削除
  const previewBar = document.querySelector('.task-bar-preview');
  if (previewBar) previewBar.remove();

  const tooltip = document.querySelector('.date-preview-tooltip');
  if (tooltip) tooltip.remove();

  // カーソルリセット
  document.body.style.cursor = 'default';

  dragState = null;
}
```

---

### 8. applyDateChange() - メイン画面への反映

```javascript
/**
 * 日程変更をメイン画面に反映
 * @param {string} wbsNo - タスクのWBS番号
 * @param {string} newStartDate - 新開始日 "YYYY-MM-DD"
 * @param {string} newEndDate - 新終了日 "YYYY-MM-DD"
 */
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  // メインウィンドウの存在確認
  if (!window.opener || window.opener.closed) {
    alert('メインウィンドウが閉じられています');
    return;
  }

  // メイン画面の関数を呼び出し
  window.opener.updateTaskDates(wbsNo, newStartDate, newEndDate);

  // メイン画面を前面に表示
  window.opener.focus();

  // ガントチャートを再描画
  setTimeout(() => {
    renderGantt();
  }, 200); // メイン画面の更新を待つ

  console.log(`タスク ${wbsNo} の日程を更新: ${newStartDate} ～ ${newEndDate}`);
}
```

---

## メイン画面側の実装

### updateTaskDates() - タスク日程更新（新規追加）

```javascript
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

---

## 日付計算ヘルパー関数

### shiftDate() - 日付シフト

```javascript
/**
 * 日付をN日シフト（タイムラインセル配列を使用）
 * @param {string} dateStr - 元の日付 "YYYY-MM-DD"
 * @param {number} shiftCells - シフトするセル数
 * @param {Array} cells - タイムラインセル配列
 * @returns {string} 新しい日付 "YYYY-MM-DD"
 */
function shiftDate(dateStr, shiftCells, cells) {
  const currentIndex = cells.findIndex(c => c.date === dateStr);

  if (currentIndex === -1) {
    console.warn(`日付 ${dateStr} がタイムライン範囲外`);
    return dateStr;
  }

  const newIndex = currentIndex + shiftCells;

  // 範囲チェック
  if (newIndex < 0) return cells[0].date;
  if (newIndex >= cells.length) return cells[cells.length - 1].date;

  return cells[newIndex].date;
}
```

---

## セル幅計算（表示モード対応）

### getCellWidth() - 現在の表示モードのセル幅を取得

```javascript
/**
 * 現在の表示モード（Day/Week/Month）に応じたセル幅を取得
 * @returns {number} セル幅（px）
 */
function getCellWidth() {
  const viewMode = getCurrentViewMode(); // 'day' | 'week' | 'month'

  switch (viewMode) {
    case 'day':
      return 40; // 1日 = 40px
    case 'week':
      return 80; // 1週間 = 80px
    case 'month':
      return 120; // 1ヶ月 = 120px
    default:
      return 40;
  }
}
```

---

## エラーハンドリング

### 1. 最小期間チェック

```javascript
/**
 * 開始日と終了日が最低1日の期間を持つかチェック
 * @param {string} startDate
 * @param {string} endDate
 * @returns {boolean}
 */
function validateMinimumDuration(startDate, endDate) {
  const start = new Date(startDate);
  const end = new Date(endDate);

  if (end <= start) {
    alert('⚠ タスクは最低1日の期間が必要です');
    return false;
  }

  return true;
}
```

### 2. 先行タスク依存関係チェック

```javascript
/**
 * 先行タスクとの依存関係をチェック
 * @param {Object} task - タスクオブジェクト
 * @param {string} newStartDate - 新開始日
 * @returns {boolean} true: 続行可能、false: キャンセル
 */
function checkPredecessorConstraint(task, newStartDate) {
  if (!task.predecessors || task.predecessors.trim() === '') {
    return true; // 先行タスクなし
  }

  const predecessorWbs = task.predecessors.split(',')[0].trim();
  const predecessorTask = projectData.tasks.find(t => t.wbs_no === predecessorWbs);

  if (!predecessorTask || !predecessorTask.finish_date) {
    return true; // 先行タスクの終了日未設定
  }

  const predEnd = new Date(predecessorTask.finish_date);
  const newStart = new Date(newStartDate);

  if (newStart < predEnd) {
    const result = confirm(
      `⚠ 警告: 先行タスク ${predecessorWbs} の終了日より前に開始します。\n` +
      `先行タスク終了日: ${predecessorTask.finish_date}\n` +
      `新開始日: ${newStartDate}\n\n` +
      `このまま保存しますか？`
    );

    return result;
  }

  return true;
}
```

---

## CSS スタイル

```css
/* 日程バー基本スタイル */
.task-bar {
  position: absolute;
  height: 30px;
  border-radius: 4px;
  transition: opacity 0.2s;
  cursor: pointer;
}

/* ドラッグ中（半透明化） */
.task-bar.dragging {
  opacity: 0.3;
  pointer-events: none;
}

/* プレビューバー */
.task-bar-preview {
  position: absolute;
  height: 30px;
  background: linear-gradient(135deg, #4CAF50 0%, #66BB6A 100%);
  border: 2px dashed #2E7D32;
  border-radius: 4px;
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
  white-space: nowrap;
  z-index: 101;
  transform: translateY(-35px);
}

/* カーソルスタイル */
.resize-cursor {
  cursor: col-resize;
}

.move-cursor {
  cursor: move;
}
```

---

## パフォーマンス最適化

### 1. イベントリスナーの管理

```javascript
// ドラッグ開始時のみドキュメントレベルのリスナーを追加
function startDrag() {
  document.addEventListener('mousemove', handleMouseMove);
  document.addEventListener('mouseup', handleMouseUp);
}

// ドラッグ終了時に削除
function endDrag() {
  document.removeEventListener('mousemove', handleMouseMove);
  document.removeEventListener('mouseup', handleMouseUp);
}
```

### 2. 再描画の最小化

```javascript
// transform を使用（reflow回避）
previewBar.style.transform = `translateX(${newX}px)`;
previewBar.style.width = `${newWidth}px`;

// 代わりに left/width を直接変更すると再描画コスト高
// previewBar.style.left = `${newX}px`; // 避ける
```

---

## テスト項目

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

---

## 関連ドキュメント

- [README.md](./README.md) - 全体概要
- [01-ui-ux-design.md](./01-ui-ux-design.md) - UI/UX設計
- [03-data-synchronization.md](./03-data-synchronization.md) - データ同期仕様
- [04-implementation-plan.md](./04-implementation-plan.md) - 実装計画

---

**最終更新**: 2026-03-21
**ステータス**: 仕様確定、実装待ち
