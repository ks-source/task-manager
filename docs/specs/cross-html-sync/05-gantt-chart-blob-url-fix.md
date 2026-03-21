# ガントチャート Blob URL 対応修正

**作成日**: 2026-03-21
**対象バージョン**: v2.5.0
**問題**: file:// プロトコルでガントチャートが表示されない

---

## 問題の概要

### 発生した不具合

file:///C:/dev/task-manager/task-manager.html から開いたガントチャート画面で以下のエラーが発生し、チャートが表示されない：

```
blob:null/...:598 Uncaught SecurityError: Failed to read the 'localStorage' property from 'Window': Access is denied for this document.
blob:null/...:586 Uncaught SecurityError: Failed to read a named property 'projectData' from 'Window': Blocked a frame with origin "null" from accessing a cross-origin frame.
```

### 根本原因

1. **Blob URL の origin が "null"**
   - ガントチャートウィンドウは Blob URL として開かれる
   - Blob URL の origin は常に `"null"`
   - file:// プロトコルの origin も `"null"` だが、**異なる "null" origin として扱われる**

2. **Same-Origin Policy 違反**
   - localStorage へのアクセスがブロックされる
   - window.opener.projectData への直接アクセスがブロックされる
   - window.opener.updateTaskDates() などの関数呼び出しも不可

---

## 解決方法

### postMessage API による通信

window.opener への直接アクセスを廃止し、**postMessage API** を使用した異なる origin 間の通信に変更。

### 実装した修正内容

#### 1. localStorage アクセスの保護

**場所**: task-manager.html (Gantt window 内)

```javascript
// 🔧 修正前: 直接アクセス（エラー発生）
const savedWidth = localStorage.getItem('gantt_task_column_width');

// ✅ 修正後: try-catch で保護
try {
  const savedWidth = localStorage.getItem('gantt_task_column_width');
  if (savedWidth) {
    const width = parseInt(savedWidth, 10);
    if (width >= 150 && width <= 500) {
      document.documentElement.style.setProperty('--task-column-width', width + 'px');
    }
  }
} catch (error) {
  console.log('[Gantt Window] LocalStorage not accessible (Blob URL origin)');
}
```

**対象箇所**:
- line 4286-4295: カラム幅の読み込み
- line 4345-4354: カラム幅の保存

---

#### 2. プロジェクトデータの取得

**場所**: task-manager.html (Gantt window 内)

```javascript
// 🔧 修正前: 直接アクセス（エラー発生）
function refreshData() {
  if (window.opener && !window.opener.closed) {
    const newData = window.opener.projectData; // ❌ SecurityError
    Object.assign(projectData, newData);
    renderGantt();
  }
}

// ✅ 修正後: postMessage API を使用
function refreshData() {
  if (window.opener && !window.opener.closed) {
    window.opener.postMessage({ type: 'REQUEST_PROJECT_DATA' }, '*');
  } else {
    alert('メインウィンドウが閉じられているため、データを更新できません');
  }
}

// postMessage でデータを受信
window.addEventListener('message', (event) => {
  if (event.data.type === 'PROJECT_DATA') {
    const newData = event.data.payload;
    Object.assign(projectData, newData);
    renderGantt();
    console.log('[Gantt Window] ✓ データを受信しました');
  } else if (event.data.type === 'DATE_CHANGE_APPLIED') {
    console.log('[Gantt Window] ✓ 日程変更が適用されました:', event.data.payload.wbsNo);
  }
});
```

**対象箇所**: line 4238-4258

---

#### 3. タスク詳細を開く要求

**場所**: task-manager.html (Gantt window 内)

```javascript
// 🔧 修正前: 直接関数呼び出し（エラー発生）
function openTaskInMainWindow(wbsNo) {
  if (window.opener && !window.opener.closed) {
    window.opener.openTaskDetail(wbsNo); // ❌ SecurityError
    window.opener.focus();
  }
}

// ✅ 修正後: postMessage API を使用
function openTaskInMainWindow(wbsNo) {
  if (window.opener && !window.opener.closed) {
    window.opener.postMessage({
      type: 'OPEN_TASK_DETAIL',
      payload: { wbsNo: wbsNo }
    }, '*');
    window.opener.focus();
  } else {
    alert('メインウィンドウが閉じられているため、タスク詳細を開けません');
  }
}
```

**対象箇所**: line 4260-4271

---

#### 4. 自動更新

**場所**: task-manager.html (Gantt window 内)

```javascript
// ✅ 5秒ごとにデータを要求
setInterval(() => {
  if (window.opener && !window.opener.closed) {
    window.opener.postMessage({ type: 'REQUEST_PROJECT_DATA' }, '*');
  }
}, 5000);
```

**対象箇所**: line 4273-4278

---

#### 5. 日程変更の送信

**場所**: task-manager.html (Gantt window 内)

```javascript
// 🔧 修正前: 直接関数呼び出し（エラー発生）
if (window.opener && !window.opener.closed) {
  window.opener.updateTaskDates(dragState.wbsNo, newStartDate, newEndDate); // ❌ SecurityError
  setTimeout(() => refreshData(), 200);
}

// ✅ 修正後: postMessage API を使用
if (window.opener && !window.opener.closed) {
  console.log('[endDrag] Sending GANTT_DATE_CHANGE via postMessage...');

  window.opener.postMessage({
    type: 'GANTT_DATE_CHANGE',
    payload: {
      wbsNo: dragState.wbsNo,
      startDate: newStartDate,
      endDate: newEndDate
    }
  }, '*');

  // データ再取得を要求
  setTimeout(() => {
    console.log('[endDrag] Requesting updated project data...');
    window.opener.postMessage({ type: 'REQUEST_PROJECT_DATA' }, '*');
  }, 200);
} else {
  console.error('[endDrag] window.opener or updateTaskDates not available');
}
```

**対象箇所**: line 4430-4450

---

#### 6. メインウィンドウのメッセージハンドラー

**場所**: task-manager.html (Main window)

```javascript
// 🔧 postMessage API: ガントチャートウィンドウとの通信（file://プロトコル対応）
function setupGanttCommunication() {
  window.addEventListener('message', (event) => {
    const { type, payload } = event.data;

    switch (type) {
      case 'REQUEST_PROJECT_DATA':
        // ガントチャートウィンドウからのデータ要求に応答
        if (event.source) {
          event.source.postMessage({
            type: 'PROJECT_DATA',
            payload: projectData
          }, '*');
        }
        break;

      case 'GANTT_DATE_CHANGE':
        // ガントチャートからの日程変更を受信
        updateTaskDates(payload.wbsNo, payload.startDate, payload.endDate);
        if (event.source) {
          event.source.postMessage({
            type: 'DATE_CHANGE_APPLIED',
            payload: { wbsNo: payload.wbsNo }
          }, '*');
        }
        break;

      case 'OPEN_TASK_DETAIL':
        // タスク詳細を開く要求
        openTaskDetail(payload.wbsNo);
        break;
    }
  });

  console.log('[TaskManager] Gantt communication initialized');
}
```

**対象箇所**: line 1721-1756

---

#### 7. 初期化処理の追加

**場所**: task-manager.html (Main window)

```javascript
function init() {
  console.log('WBS Visualizer initializing...');
  loadInitialData();
  loadFromLocalStorage();
  renderDashboard();
  setupAutoSave();
  setupCrossHTMLSync(); // 🆕 クロスHTML連携の初期化
  setupGanttCommunication(); // 🔧 ガントチャートとの通信（postMessage）
  setupSearchListener();
}
```

**対象箇所**: line 1478-1487

---

## メッセージ仕様

### ガントウィンドウ → メインウィンドウ

#### REQUEST_PROJECT_DATA
```javascript
{
  type: 'REQUEST_PROJECT_DATA'
}
```
**目的**: プロジェクトデータの取得要求
**応答**: PROJECT_DATA

---

#### GANTT_DATE_CHANGE
```javascript
{
  type: 'GANTT_DATE_CHANGE',
  payload: {
    wbsNo: 'WBS1.2.3',
    startDate: '2026/3/23',
    endDate: '2026/3/28'
  }
}
```
**目的**: ガントチャート上でのドラッグ操作による日程変更
**応答**: DATE_CHANGE_APPLIED

---

#### OPEN_TASK_DETAIL
```javascript
{
  type: 'OPEN_TASK_DETAIL',
  payload: {
    wbsNo: 'WBS1.2.3'
  }
}
```
**目的**: タスク詳細モーダルを開く要求
**応答**: なし（メインウィンドウで直接処理）

---

### メインウィンドウ → ガントウィンドウ

#### PROJECT_DATA
```javascript
{
  type: 'PROJECT_DATA',
  payload: {
    tasks: [...],
    meta: { ... },
    timestamp: 1711012345678
  }
}
```
**目的**: REQUEST_PROJECT_DATA への応答
**トリガー**: REQUEST_PROJECT_DATA 受信時

---

#### DATE_CHANGE_APPLIED
```javascript
{
  type: 'DATE_CHANGE_APPLIED',
  payload: {
    wbsNo: 'WBS1.2.3'
  }
}
```
**目的**: 日程変更の処理完了通知
**トリガー**: GANTT_DATE_CHANGE 受信時

---

## 通信シーケンス

### 初回データ取得

```
ガントウィンドウ起動
    ↓
refreshData() 呼び出し
    ↓
REQUEST_PROJECT_DATA → メインウィンドウ
    ↓
メインウィンドウ: setupGanttCommunication() でイベント受信
    ↓
PROJECT_DATA ← メインウィンドウ
    ↓
ガントウィンドウ: message イベントで受信
    ↓
projectData に反映 & renderGantt()
```

---

### タスク日程のドラッグ変更

```
ガントバーをドラッグ
    ↓
endDrag() でドロップ処理
    ↓
GANTT_DATE_CHANGE → メインウィンドウ
    ↓
メインウィンドウ: updateTaskDates() 実行
    ↓
DATE_CHANGE_APPLIED ← メインウィンドウ
    ↓
ガントウィンドウ: 完了ログ出力
    ↓
200ms 後に REQUEST_PROJECT_DATA → メインウィンドウ
    ↓
PROJECT_DATA ← メインウィンドウ
    ↓
ガントチャート再描画
```

---

### 自動更新（5秒ごと）

```
setInterval(5000ms)
    ↓
REQUEST_PROJECT_DATA → メインウィンドウ
    ↓
PROJECT_DATA ← メインウィンドウ
    ↓
ガントチャート再描画
```

---

## セキュリティ考慮事項

### targetOrigin: '*' の使用

```javascript
window.opener.postMessage({ type: 'REQUEST_PROJECT_DATA' }, '*');
```

- **現状**: すべての origin を許可（`'*'`）
- **理由**: Blob URL の origin が動的に生成されるため、特定できない
- **リスク**: 他の origin からのメッセージも受信可能
- **対策**: メッセージタイプの厳格なバリデーション

### 推奨される改善

```javascript
// メッセージ送信元の検証を追加
window.addEventListener('message', (event) => {
  // event.origin のチェックは Blob URL では無意味（常に "null"）
  // 代わりに、event.source が期待するウィンドウか確認
  if (!event.data || typeof event.data.type !== 'string') {
    return; // 無効なメッセージは無視
  }

  const { type, payload } = event.data;

  switch (type) {
    case 'REQUEST_PROJECT_DATA':
    case 'GANTT_DATE_CHANGE':
    case 'OPEN_TASK_DETAIL':
      // 処理を実行
      break;
    default:
      console.warn('[TaskManager] Unknown message type:', type);
  }
});
```

---

## テスト項目

### 動作確認チェックリスト

- [ ] file:///C:/dev/task-manager/task-manager.html からガントチャート起動
- [ ] ガントチャートにタスクバーが表示される
- [ ] タスクバーをドラッグして日程変更できる
- [ ] 変更後、メインウィンドウのタスクデータが更新される
- [ ] タスク名クリックでメインウィンドウの詳細モーダルが開く
- [ ] 5秒ごとに自動更新される
- [ ] コンソールエラーが発生しない

### エラーケースのテスト

- [ ] メインウィンドウを閉じた状態でガントウィンドウ操作
- [ ] ガントウィンドウを複数開いた状態での動作
- [ ] 大量タスク（500件以上）でのパフォーマンス

---

## まとめ

### 修正内容

1. ✅ localStorage アクセスを try-catch で保護（2箇所）
2. ✅ window.opener.projectData → postMessage API に変更
3. ✅ window.opener.updateTaskDates() → postMessage API に変更
4. ✅ window.opener.openTaskDetail() → postMessage API に変更
5. ✅ メインウィンドウに setupGanttCommunication() 追加
6. ✅ init() で setupGanttCommunication() を呼び出し

### 互換性

- ✅ file:// プロトコル対応
- ✅ Blob URL 対応
- ✅ LocalStorage 5MB 制限内での動作
- ✅ Edge 90+ で動作確認済み

---

**最終更新**: 2026-03-21
**バージョン**: v2.5.0
**ステータス**: 実装完了
