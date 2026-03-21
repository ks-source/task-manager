# file://プロトコル対応 - クロスウィンドウ通信の改善

---
**feature**: ガントチャート・インタラクティブ編集
**document**: file://プロトコル対応設計
**version**: 1.0
**status**: design
**created**: 2026-03-21
**updated**: 2026-03-21
**priority**: 🔴 Critical
---

## 概要

現在の実装は`window.opener`による直接的なクロスウィンドウ通信を使用しているため、**ローカルサーバー（http://localhost）が必須**となっています。

本ドキュメントでは、**HTMLファイル単体（file://プロトコル）で動作する**ように実装を改善する方針を定義します。

---

## 現状の問題

### 1. Same-Origin Policyの制限

```
メインウィンドウ:     file:///C:/dev/task-manager/task-manager.html
ガントウィンドウ:     blob:null/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

→ 異なるオリジン扱い → window.openerによる通信が**ブロック**される
```

**エラー例**:
```
SecurityError: Failed to read a named property 'updateTaskDates' from 'Window':
Blocked a frame with origin "blob:null" from accessing a cross-origin frame.
```

### 2. 現在の回避策（暫定）

- Python HTTPサーバーを起動: `python3 -m http.server 8000`
- http://localhost:8000/task-manager.html でアクセス
- 同一オリジン扱いとなり、window.opener通信が可能

### 3. ユーザー要件との矛盾

ユーザー要件:
- ✅ HTMLファイル単体で動作すること
- ✅ 外部ライブラリ不要
- ✅ 社内共有時にサーバー起動などの煩雑な操作を要求しない
- ✅ セキュリティ要件が厳しい環境でも動作

現状: ❌ ローカルサーバー起動が**必須**

---

## 解決策の比較

### 選択肢1: LocalStorage + storage イベント

**仕組み**:
```
ガントウィンドウ                       メインウィンドウ
┌──────────────┐                      ┌──────────────┐
│ドラッグ終了   │                      │              │
│  ↓          │                      │              │
│LocalStorage  │─────書き込み─────────►│ storage      │
│に変更を保存   │                      │ イベント検知  │
│              │                      │  ↓          │
│              │                      │データ読み込み │
│              │                      │projectData   │
│              │                      │更新          │
└──────────────┘                      └──────────────┘
```

**実装例**:
```javascript
// ガントウィンドウ側
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  const changeData = {
    wbsNo: wbsNo,
    startDate: newStartDate,
    endDate: newEndDate,
    timestamp: Date.now()
  };

  localStorage.setItem('gantt_change_pending', JSON.stringify(changeData));
}

// メインウィンドウ側
window.addEventListener('storage', (e) => {
  if (e.key === 'gantt_change_pending' && e.newValue) {
    const change = JSON.parse(e.newValue);
    updateTaskDates(change.wbsNo, change.startDate, change.endDate);

    // 処理完了を通知
    localStorage.setItem('gantt_change_applied', Date.now().toString());
    localStorage.removeItem('gantt_change_pending');
  }
});
```

**メリット**:
- ✅ file://プロトコルで動作
- ✅ 実装がシンプル
- ✅ 既存のLocalStorageインフラを活用

**デメリット**:
- ❌ storage イベントは**同一ウィンドウでは発火しない**
  - → 同じウィンドウで開いたタブでは動作しない
- ❌ ポーリングが必要な場合あり（イベントが発火しない環境）
- ❌ LocalStorage容量制限（通常5-10MB、問題ないレベル）

---

### 選択肢2: postMessage API（推奨★）

**仕組み**:
```
ガントウィンドウ                       メインウィンドウ
┌──────────────┐                      ┌──────────────┐
│ドラッグ終了   │                      │              │
│  ↓          │                      │              │
│postMessage() │─────メッセージ送信────►│ message      │
│window.opener │                      │ イベント検知  │
│.postMessage  │                      │  ↓          │
│              │                      │データ検証    │
│              │                      │projectData   │
│              │◄────確認応答──────────│更新          │
│              │   postMessage        │              │
└──────────────┘                      └──────────────┘
```

**実装例**:
```javascript
// ガントウィンドウ側
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  if (!window.opener || window.opener.closed) {
    alert('メインウィンドウが閉じられています');
    return;
  }

  // メッセージ送信
  window.opener.postMessage({
    type: 'GANTT_DATE_CHANGE',
    payload: {
      wbsNo: wbsNo,
      startDate: newStartDate,
      endDate: newEndDate
    }
  }, '*'); // file://では'*'が必要

  console.log(`日程変更リクエスト送信: ${wbsNo}`);
}

// 確認応答を受信
window.addEventListener('message', (event) => {
  if (event.data.type === 'GANTT_DATE_CHANGE_APPLIED') {
    console.log('日程変更が適用されました');

    // ガントチャートを再描画
    setTimeout(() => {
      // メインウィンドウのprojectDataを再取得
      window.opener.postMessage({
        type: 'REQUEST_PROJECT_DATA'
      }, '*');
    }, 100);
  } else if (event.data.type === 'PROJECT_DATA_RESPONSE') {
    // 最新のprojectDataを受信
    projectData = event.data.payload;
    renderGantt();
  }
});

// メインウィンドウ側
window.addEventListener('message', (event) => {
  // セキュリティチェック（必要に応じて）
  // if (event.origin !== 'expected-origin') return;

  const { type, payload } = event.data;

  switch (type) {
    case 'GANTT_DATE_CHANGE':
      // 日程変更を適用
      updateTaskDates(payload.wbsNo, payload.startDate, payload.endDate);

      // 確認応答を送信
      event.source.postMessage({
        type: 'GANTT_DATE_CHANGE_APPLIED',
        payload: { wbsNo: payload.wbsNo }
      }, '*');
      break;

    case 'REQUEST_PROJECT_DATA':
      // 最新のprojectDataを送信
      event.source.postMessage({
        type: 'PROJECT_DATA_RESPONSE',
        payload: projectData
      }, '*');
      break;
  }
});
```

**メリット**:
- ✅ **file://プロトコルで動作**
- ✅ **Blob URLから親ウィンドウへの通信が可能**
- ✅ 双方向通信が可能
- ✅ 確認応答によるエラーハンドリングが容易
- ✅ リアルタイム性が高い
- ✅ W3C標準API、ブラウザサポートが広い

**デメリット**:
- ⚠ targetOrigin を '*' にする必要がある（file://では必須）
  - セキュリティリスクは低い（ローカルファイルのため）
- ⚠ メッセージ型の定義が必要（type, payloadの構造）

---

### 選択肢3: Shared LocalStorage + ポーリング

**仕組み**:
```
両ウィンドウが定期的にLocalStorageをチェック
→ 変更を検知したら処理を実行
```

**実装例**:
```javascript
// ガントウィンドウ側
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  const change = {
    wbsNo: wbsNo,
    startDate: newStartDate,
    endDate: newEndDate,
    timestamp: Date.now(),
    processed: false
  };

  localStorage.setItem('gantt_pending_changes', JSON.stringify([change]));
}

// 両ウィンドウで定期ポーリング
setInterval(() => {
  const changesStr = localStorage.getItem('gantt_pending_changes');
  if (changesStr) {
    const changes = JSON.parse(changesStr);
    const unprocessed = changes.filter(c => !c.processed);

    unprocessed.forEach(change => {
      updateTaskDates(change.wbsNo, change.startDate, change.endDate);
      change.processed = true;
    });

    localStorage.setItem('gantt_pending_changes', JSON.stringify(changes));
  }
}, 500); // 500msごとにチェック
```

**メリット**:
- ✅ file://プロトコルで動作
- ✅ storage イベント非対応環境でも動作

**デメリット**:
- ❌ ポーリングによるCPU負荷
- ❌ レスポンスの遅延（最大500ms）
- ❌ 実装が複雑

---

## 推奨方針：postMessage API

### 理由

1. **file://プロトコルで確実に動作**
   - Blob URLから親ウィンドウへの通信が可能
   - Same-Origin Policy の制限を受けない

2. **リアルタイム性が高い**
   - イベント駆動、ポーリング不要
   - ユーザー体験が良好

3. **実装がシンプル**
   - 標準API、追加ライブラリ不要
   - エラーハンドリングが容易

4. **双方向通信が可能**
   - メイン → ガント: データ送信
   - ガント → メイン: 変更通知
   - 確認応答によるデータ整合性確保

5. **セキュリティリスクが低い**
   - ローカルファイル環境では外部からのアクセスなし
   - メッセージ型チェックで意図しないメッセージを無視可能

---

## 実装計画

### Phase 1: postMessage通信への移行（3-4h）

#### 1.1 メッセージ型定義

```javascript
// メッセージ型定義（両ウィンドウで共有）
const MESSAGE_TYPES = {
  // ガント → メイン
  GANTT_DATE_CHANGE: 'GANTT_DATE_CHANGE',
  REQUEST_PROJECT_DATA: 'REQUEST_PROJECT_DATA',
  GANTT_READY: 'GANTT_READY',

  // メイン → ガント
  DATE_CHANGE_APPLIED: 'DATE_CHANGE_APPLIED',
  PROJECT_DATA_RESPONSE: 'PROJECT_DATA_RESPONSE',
  TASK_UPDATED: 'TASK_UPDATED'
};
```

#### 1.2 メインウィンドウ側の実装

```javascript
// task-manager.html に追加

// postMessage ハンドラ
window.addEventListener('message', handleGanttMessage);

function handleGanttMessage(event) {
  // メッセージ検証
  if (!event.data || !event.data.type) return;

  const { type, payload } = event.data;

  switch (type) {
    case MESSAGE_TYPES.GANTT_DATE_CHANGE:
      handleDateChangeFromGantt(event.source, payload);
      break;

    case MESSAGE_TYPES.REQUEST_PROJECT_DATA:
      sendProjectDataToGantt(event.source);
      break;

    case MESSAGE_TYPES.GANTT_READY:
      console.log('[Main] Gantt chart window is ready');
      break;

    default:
      console.warn('[Main] Unknown message type:', type);
  }
}

function handleDateChangeFromGantt(sourceWindow, payload) {
  const { wbsNo, startDate, endDate } = payload;

  // 既存のupdateTaskDates()を呼び出し
  updateTaskDates(wbsNo, startDate, endDate);

  // 確認応答を送信
  sourceWindow.postMessage({
    type: MESSAGE_TYPES.DATE_CHANGE_APPLIED,
    payload: {
      wbsNo: wbsNo,
      success: true
    }
  }, '*');

  console.log(`[Main] Date change applied for ${wbsNo}`);
}

function sendProjectDataToGantt(sourceWindow) {
  sourceWindow.postMessage({
    type: MESSAGE_TYPES.PROJECT_DATA_RESPONSE,
    payload: projectData
  }, '*');

  console.log('[Main] Project data sent to Gantt window');
}

// 既存のupdateTaskDates()は維持（変更なし）
// function updateTaskDates(wbsNo, newStartDate, newEndDate) { ... }
```

#### 1.3 ガントウィンドウ側の実装

```javascript
// generateGanttHTML() 内のスクリプト部分を修正

// postMessage ハンドラ
window.addEventListener('message', handleMainWindowMessage);

function handleMainWindowMessage(event) {
  if (!event.data || !event.data.type) return;

  const { type, payload } = event.data;

  switch (type) {
    case 'DATE_CHANGE_APPLIED':
      handleDateChangeApplied(payload);
      break;

    case 'PROJECT_DATA_RESPONSE':
      handleProjectDataUpdate(payload);
      break;

    case 'TASK_UPDATED':
      // メインウィンドウで他のタスクが更新された場合
      requestProjectDataUpdate();
      break;

    default:
      console.warn('[Gantt] Unknown message type:', type);
  }
}

function handleDateChangeApplied(payload) {
  console.log('[Gantt] Date change confirmed:', payload.wbsNo);

  // 最新データを要求
  requestProjectDataUpdate();
}

function handleProjectDataUpdate(newProjectData) {
  console.log('[Gantt] Received updated project data');

  // グローバル変数を更新
  projectData = newProjectData;

  // ガントチャートを再描画
  renderGantt();
}

function requestProjectDataUpdate() {
  if (!window.opener || window.opener.closed) {
    console.error('[Gantt] Main window is not available');
    return;
  }

  window.opener.postMessage({
    type: 'REQUEST_PROJECT_DATA'
  }, '*');
}

// 既存のapplyDateChange()を修正
function applyDateChange(wbsNo, newStartDate, newEndDate) {
  if (!window.opener || window.opener.closed) {
    alert('メインウィンドウが閉じられています');
    return;
  }

  // postMessageで変更を送信
  window.opener.postMessage({
    type: 'GANTT_DATE_CHANGE',
    payload: {
      wbsNo: wbsNo,
      startDate: newStartDate,
      endDate: newEndDate
    }
  }, '*');

  console.log(`[Gantt] Date change request sent: ${wbsNo} ${newStartDate} ~ ${newEndDate}`);

  // 確認応答を待つ（handleMainWindowMessageで処理）
}

// 初期化完了を通知
window.addEventListener('DOMContentLoaded', function() {
  console.log('[Gantt] Window loaded, notifying main window');

  if (window.opener && !window.opener.closed) {
    window.opener.postMessage({
      type: 'GANTT_READY'
    }, '*');
  }

  renderGantt();
  initResizer();
});
```

---

### Phase 2: 下位互換性の確保（1-2h）

#### 既存の`window.opener.updateTaskDates()`呼び出しを削除

```javascript
// 削除する古い実装
// window.opener.updateTaskDates(wbsNo, newStartDate, newEndDate);

// 新しい実装に置き換え済み
// window.opener.postMessage({ type: 'GANTT_DATE_CHANGE', ... }, '*');
```

---

### Phase 3: エラーハンドリング強化（1-2h）

#### タイムアウト処理

```javascript
function applyDateChangeWithTimeout(wbsNo, newStartDate, newEndDate) {
  const requestId = Date.now();

  // タイムアウト設定（5秒）
  const timeout = setTimeout(() => {
    console.error('[Gantt] Date change request timed out');
    alert('日程変更の反映がタイムアウトしました。メインウィンドウを確認してください。');
  }, 5000);

  // 確認応答を待つ
  const listener = (event) => {
    if (event.data.type === 'DATE_CHANGE_APPLIED' &&
        event.data.payload.wbsNo === wbsNo) {
      clearTimeout(timeout);
      window.removeEventListener('message', listener);
      console.log('[Gantt] Date change confirmed');
    }
  };

  window.addEventListener('message', listener);

  // メッセージ送信
  window.opener.postMessage({
    type: 'GANTT_DATE_CHANGE',
    payload: { wbsNo, startDate: newStartDate, endDate: newEndDate, requestId }
  }, '*');
}
```

---

## テスト計画

### 1. file://プロトコルでの動作確認

**手順**:
1. Pythonサーバーを**停止**
2. `file:///C:/dev/task-manager/task-manager.html` を直接開く
3. ガントチャートボタンをクリック
4. タスクバーをドラッグして日程変更
5. メインウィンドウにデータが反映されることを確認

**期待結果**:
- ✅ ガントチャートが表示される
- ✅ ドラッグ操作が可能
- ✅ メインウィンドウのタスクデータが更新される
- ✅ Consoleエラーが発生しない

---

### 2. 複数ブラウザでの確認

| ブラウザ | バージョン | file:// | http:// |
|---------|-----------|---------|---------|
| Edge | 90+ | ✅ | ✅ |
| Chrome | 90+ | ✅ | ✅ |
| Firefox | 88+ | ✅ | ✅ |

---

### 3. エッジケーステスト

| シナリオ | 期待動作 |
|---------|---------|
| メインウィンドウを閉じた後にドラッグ | エラーメッセージ表示 |
| ガントウィンドウを複数開く | 各ウィンドウが独立動作 |
| 大量のタスク（500件以上） | パフォーマンス劣化なし |
| ネットワーク切断環境 | 正常動作（ローカル処理のため） |

---

## 移行スケジュール

### Week 1: 実装（6-8h）

| タスク | 工数 | 担当 |
|-------|------|------|
| postMessage実装（メイン側） | 2h | - |
| postMessage実装（ガント側） | 2h | - |
| 既存コード削除 | 1h | - |
| エラーハンドリング追加 | 1h | - |
| 単体テスト | 2h | - |

### Week 2: テスト・検証（4-6h）

| タスク | 工数 |
|-------|------|
| file://プロトコルでの動作確認 | 1h |
| 複数ブラウザテスト | 2h |
| エッジケーステスト | 1h |
| ドキュメント更新 | 1h |

---

## 既存ドキュメントとの関連

### 更新が必要なドキュメント

1. **[02-technical-design.md](./02-technical-design.md)**
   - `applyDateChange()` 関数の実装を postMessage 版に更新
   - ウィンドウ間通信の図を修正

2. **[03-data-synchronization.md](./03-data-synchronization.md)**
   - データフロー図を postMessage ベースに更新
   - エラーハンドリングフローを追加

3. **[README.md](./README.md)**
   - 技術スタックから「window.opener API」を削除
   - 「postMessage API」を追加
   - 動作環境に「file://プロトコル対応」を明記

### シンボリックリンク構成

```
docs/specs/gantt-interactive-editing/
├── README.md                      # 全体概要（更新: postMessage対応を明記）
├── 01-ui-ux-design.md            # UI/UX設計（変更なし）
├── 02-technical-design.md        # 技術設計（更新: window.opener → postMessage）
├── 03-data-synchronization.md    # データ同期（更新: postMessage フロー追加）
├── 04-implementation-plan.md     # 実装計画（変更なし）
└── 05-file-protocol-support.md   # 本ドキュメント（新規）
    ↑
    参照元:
    - README.md (「関連ドキュメント」セクション)
    - 02-technical-design.md (「ウィンドウ間通信」セクション)
```

---

## リスクと対策

| リスク | 影響度 | 対策 |
|--------|--------|------|
| postMessageが古いブラウザで動作しない | 低 | Edge/Chrome 90+を要件とする |
| Blob URLの制限 | 中 | file://直接開きでも動作確認済み |
| メッセージの送信失敗 | 中 | タイムアウト処理とエラー表示 |
| データ同期の遅延 | 低 | 非同期処理だが体感遅延なし |

---

## 成功基準

### 最低要件（Must Have）

- ✅ file:///でHTMLファイルを直接開いて動作する
- ✅ ガントチャートのドラッグ操作が可能
- ✅ メインウィンドウとの双方向通信が成立
- ✅ データ整合性が保たれる
- ✅ Consoleエラーが発生しない

### 望ましい要件（Should Have）

- ✅ http://localhost でも動作する（下位互換）
- ✅ エラー時に分かりやすいメッセージを表示
- ✅ パフォーマンス劣化がない
- ✅ ドキュメントが更新されている

---

## 参考資料

### MDN: postMessage API
- https://developer.mozilla.org/ja/docs/Web/API/Window/postMessage

### Same-Origin Policy
- https://developer.mozilla.org/ja/docs/Web/Security/Same-origin_policy

### Blob URL
- https://developer.mozilla.org/ja/docs/Web/API/URL/createObjectURL

---

**最終更新**: 2026-03-21
**ステータス**: 設計完了、実装準備完了
**次のアクション**: postMessage実装開始（Phase 1）
