# クロスHTML連携仕様 - タスクマネージャー ↔ フローチャートビューワー

---
**feature**: クロスHTML連携
**version**: 1.0
**status**: ⚠️ **DEPRECATED（陳腐化）**
**created**: 2026-03-21
**updated**: 2026-03-22
**priority**: 🔴 Critical
---

## ⚠️ 重要: このドキュメントは陳腐化しています

### 最新の実装とドキュメントはこちら

**新しいドキュメント**: [../data-integration/](../data-integration/)

このディレクトリ（`cross-html-sync/`）は**設計段階のドキュメント**として2026-03-21に作成されましたが、実際の実装は**data-integration/**ディレクトリで完了しました。

### 陳腐化の理由

| 項目 | cross-html-sync/ (本ディレクトリ) | data-integration/ (最新) |
|------|----------------------------------|-------------------------|
| **ステータス** | 設計のみ、実装なし | Phase 1実装完了 |
| **作成日** | 2026-03-21 | 2026-03-22 |
| **内容** | 理論的な設計仕様 | 実装レポート・修正仕様 |
| **コミット** | なし | v2.15.0でコミット済み |
| **動作確認** | なし | 検証シナリオあり |

### 移行ガイド

以下のドキュメントを参照してください:

1. **全体概要を知りたい**
   → [../data-integration/README.md](../data-integration/README.md)

2. **Phase 0-3の実装計画を見たい**
   → [../data-integration/future-roadmap.md](../data-integration/future-roadmap.md)

3. **Phase 1の実装内容を確認したい**
   → [../data-integration/phase1-implementation-summary.md](../data-integration/phase1-implementation-summary.md)

4. **最新の修正仕様（mermaid_ids双方向同期）**
   → [../data-integration/phase1-mermaid-ids-fix.md](../data-integration/phase1-mermaid-ids-fix.md) ⭐ **最優先で読むべきドキュメント**

### このドキュメントを残す理由

- **設計思想の記録**: LocalStorage選定理由、データ永続化戦略などの背景情報
- **技術比較資料**: postMessage vs LocalStorageの比較
- **FAQ参考資料**: 容量超過時の対処など、一部情報は有用

### 今後の方針

- ✅ このディレクトリは**参考資料**として保持
- ✅ 新規実装は**data-integration/**を参照
- ✅ 仕様変更は**data-integration/**のみ更新
- ⚠️ このディレクトリのドキュメントは今後更新されません

---

## 概要（以下は設計段階の記録です）

**独立したHTMLファイル間**でタスクデータとフローチャート属性を双方向同期する仕組み。

### 連携対象

```
HTMLファイル1: task-manager.html
  - タスクデータの管理
  - ガントチャート表示
  - タスク属性編集

HTMLファイル2: flowchart-editor.html
  - フローチャートSVG表示
  - タスク↔ノード関連付け
  - ビジュアル編集
```

### 背景

2つのHTMLファイルは**別々に開かれる**ため、以下の制約があります：

- ❌ 親子関係なし（window.opener不使用）
- ❌ postMessage直接通信不可
- ❌ 同一ウィンドウではない

→ **LocalStorage + storage イベント**で連携

---

## ユースケース

### ケース1: タスク更新 → フローチャート反映

```
ユーザー操作:
1. task-manager.html でタスクを編集
   - タスク名変更
   - 期間変更
   - ステータス更新

自動連携:
2. LocalStorage に保存
3. storage イベント発火
4. flowchart-editor.html が検知
5. SVGを自動更新
```

### ケース2: フローチャート編集 → タスク反映

```
ユーザー操作:
1. flowchart-editor.html でノードとタスクを関連付け
   - ドラッグ&ドロップでタスクをノードに配置
   - ノードタイプを選択（process/decision/start/end）

自動連携:
2. フローチャート属性を LocalStorage に保存
3. storage イベント発火
4. task-manager.html が検知
5. タスクにフローチャート情報を追加
```

### ケース3: ガントチャート編集 → フローチャート反映

```
ユーザー操作:
1. ガントチャートウィンドウでタスク期間をドラッグ変更

自動連携:
2. メインウィンドウ（task-manager.html）が更新
3. LocalStorage に保存
4. flowchart-editor.html が検知
5. 該当タスクのノード表示を更新
```

---

## 技術アプローチ

### 選択した方式: LocalStorage + storage イベント

#### メリット

- ✅ **独立したHTMLファイル間で動作**
- ✅ **file://プロトコル対応**
- ✅ **リアルタイム同期**（storage イベント）
- ✅ **永続化**（ブラウザを閉じても保持）
- ✅ **実装がシンプル**

#### デメリット

- ⚠️ 容量制限（5-10MB、問題ないレベル）
- ⚠️ 同一ウィンドウでは storage イベント発火しない
  → 連携対象は別HTMLなので問題なし

---

## データ永続化戦略

### 二重保護構造

本システムは**LocalStorage（一時バッファ）**と**本体ファイル（マスターデータ）**の二重構造でデータを保護します。

#### 1. LocalStorage（一時バッファ）

```
役割: HTMLファイル間のリアルタイム同期
特性: 揮発性（ブラウザキャッシュクリアで消える）
容量: 5MB上限（Chrome/Edge基準）
用途: 編集中の即座の反映、storage イベント経由の同期
```

**重要**: LocalStorageは**通信用の一時バッファ**であり、**マスターデータではありません**。

#### 2. 本体ファイル（マスターデータ）

```
役割: プロジェクトデータの永続保存
特性: 永続性（ファイルシステム上に実在）
容量: 制限なし（ディスク容量のみ）
用途: プロジェクトの長期保存、バックアップ、完全復元
保存先: File System Access API経由で任意のローカルフォルダ
```

**保証**: LocalStorage容量超過時でも、本体ファイルに保存済みのデータは**完全に保護**されます。

### データ保存フロー

#### 正常時のフロー

```
┌─────────────────────────────────────────────────┐
│ 1. task-manager.html でタスク編集               │
│    ↓                                            │
│ 2. ✅ LocalStorage へ即座に保存                 │
│    ├─ task-manager-data                        │
│    └─ task-manager-update-trigger              │
│    ↓                                            │
│ 3. ✅ 本体ファイルへ自動保存（3秒デバウンス）    │
│    File System Access API                      │
│    → C:\Projects\myproject.json                │
│    ↓                                            │
│ 4. flowchart-editor.html が検知     │
│    (storage イベント)                           │
│    ↓                                            │
│ 5. フローチャートでノード関連付け               │
│    ↓                                            │
│ 6. ✅ LocalStorage へ即座に保存                 │
│    ├─ flowchart-attributes                     │
│    └─ flowchart-update-trigger                 │
│    ↓                                            │
│ 7. task-manager.html が検知 & 統合              │
│    タスクに flowchart 属性を追加                │
│    ↓                                            │
│ 8. ✅ 本体ファイルへ自動保存                    │
│    → flowchart属性を含むデータを永続化          │
└─────────────────────────────────────────────────┘
```

#### LocalStorage容量超過時のフロー

```
┌─────────────────────────────────────────────────┐
│ 状況: LocalStorageが5MB上限に到達               │
├─────────────────────────────────────────────────┤
│ 1. ❌ LocalStorage への新規保存が失敗            │
│    QuotaExceededError 発生                      │
│    ↓                                            │
│ 2. ⚠️ リアルタイム同期が一時的に機能しない       │
│    ・task-manager → flowchart への即座の反映停止│
│    ・flowchart → task-manager への即座の反映停止│
│    ↓                                            │
│ 3. ✅ 本体ファイルは影響を受けない               │
│    ・C:\Projects\myproject.json は無傷          │
│    ・すべてのタスク + flowchart属性を保持       │
│    ・File System Access API は LocalStorage不要 │
│    ↓                                            │
│ 4. 🔧 自動対処: アーカイブ提案                   │
│    ・完了タスクを別ファイルへアーカイブ         │
│    ・LocalStorageの空き容量を確保               │
│    ↓                                            │
│ 5. ✅ 本体ファイルから再読み込み                 │
│    ・myproject.json を読み込み                  │
│    ・LocalStorage へ再展開                      │
│    ・リアルタイム同期が復活                      │
└─────────────────────────────────────────────────┘
```

### 自動保存戦略

#### LocalStorage自動保存（即時）

```javascript
// タスク編集時に即座にLocalStorageへ保存
function saveToLocalStorage() {
  const data = {
    tasks: projectData.tasks,
    meta: projectData.meta,
    timestamp: Date.now()
  };

  try {
    localStorage.setItem('task-manager-data', JSON.stringify(data));
    localStorage.setItem('task-manager-update-trigger', Date.now().toString());
  } catch (error) {
    if (error.name === 'QuotaExceededError') {
      handleQuotaExceeded(); // 容量超過時の処理
    }
  }
}
```

#### 本体ファイル自動保存（デバウンス付き）

```javascript
let autoSaveTimeout = null;

function autoSaveToFile() {
  // デバウンス: 3秒間編集がなければ保存
  if (autoSaveTimeout) clearTimeout(autoSaveTimeout);

  autoSaveTimeout = setTimeout(async () => {
    await saveToMasterFile();
  }, 3000);
}

async function saveToMasterFile() {
  if (!currentFileHandle) {
    console.warn('ファイルハンドルがありません。');
    return;
  }

  try {
    const writable = await currentFileHandle.createWritable();
    const data = {
      tasks: projectData.tasks,  // flowchart属性を含む
      meta: projectData.meta,
      timestamp: Date.now()
    };

    await writable.write(JSON.stringify(data, null, 2));
    await writable.close();

    console.log('✅ 本体ファイルへ自動保存完了');
  } catch (error) {
    console.error('❌ 本体ファイル保存エラー:', error);
    alert('ファイル保存に失敗しました。');
  }
}

// タスク編集時に両方へ保存
function onTaskEdit() {
  saveToLocalStorage();  // リアルタイム同期用（即時）
  autoSaveToFile();      // 永続保存用（3秒デバウンス）
}
```

### データ安全性の保証

#### ✅ 保証されること

| 項目 | 保証内容 |
|------|---------|
| **本体ファイルの独立性** | LocalStorage容量とは無関係に保存可能 |
| **データの永続性** | ブラウザキャッシュクリアしても本体ファイルは残る |
| **完全復元可能性** | いつでも本体ファイルから100%復元可能 |
| **フローチャート属性の保存** | `task.flowchart` として本体ファイルに永続化 |
| **容量無制限** | ディスク容量の範囲内で制限なし |

#### ⚠️ 前提条件

| 条件 | 重要性 | 対策 |
|------|--------|------|
| **定期的な本体ファイル保存** | 必須 | 3秒デバウンス自動保存を実装 |
| **保存エラーがないこと** | 必須 | エラー時にユーザーへ通知 |
| **ファイルハンドル権限** | 必須 | File System Access API権限取得 |

### 復旧手順

LocalStorage容量超過時の復旧例:

```javascript
async function recoverFromMasterFile() {
  try {
    // 1. 本体ファイルを開く
    const [fileHandle] = await window.showOpenFilePicker({
      types: [{
        description: 'Project JSON',
        accept: { 'application/json': ['.json'] }
      }]
    });

    const file = await fileHandle.getFile();
    const content = await file.text();
    const projectData = JSON.parse(content);

    // 2. ✅ すべてのデータが復元される
    console.log('復元タスク数:', projectData.tasks.length);
    console.log('フローチャート属性:',
      projectData.tasks.filter(t => t.flowchart).length);

    // 3. LocalStorageへ部分的に展開（進行中タスクのみ）
    const activeTasks = projectData.tasks.filter(
      t => t.status !== '完了'
    );

    localStorage.setItem('task-manager-data', JSON.stringify({
      tasks: activeTasks,
      meta: projectData.meta,
      timestamp: Date.now()
    }));

    alert('✅ データ復旧完了');
  } catch (error) {
    console.error('復旧失敗:', error);
  }
}
```

---

## データフロー

### 全体像

```
┌────────────────────────┐
│  task-manager.html     │
│  ┌──────────────────┐  │
│  │ タスク編集        │  │
│  │  ↓              │  │
│  │ saveToLocalStorage│ │
│  └──────────────────┘  │
└───────────┬────────────┘
            │ (1) setItem
            ↓
    ┌───────────────────┐
    │   LocalStorage    │
    │ ┌───────────────┐ │
    │ │task-manager-  │ │
    │ │data           │ │
    │ │flowchart-     │ │
    │ │attributes     │ │
    │ └───────────────┘ │
    └───────┬───────────┘
            │ (2) storage event
            ↓
┌────────────────────────────────┐
│ flowchart-editor.html│
│  ┌──────────────────────────┐  │
│  │ storage イベント検知       │  │
│  │  ↓                       │  │
│  │ タスクデータ読み込み       │  │
│  │  ↓                       │  │
│  │ SVG更新                  │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
```

### 逆方向（フローチャート → タスク）

```
flowchart-editor.html
  ↓ (1) ノード関連付け
  ↓ (2) saveFlowchartAttributes()
LocalStorage
  ↓ (3) storage event
task-manager.html
  ↓ (4) タスクに flowchart 属性追加
```

---

## ドキュメント構成

### 仕様書（本ディレクトリ）

1. [**README.md**](./README.md) - 本ドキュメント、全体概要
2. [**01-localstorage-design.md**](./01-localstorage-design.md) - LocalStorage設計、storage イベント実装
3. [**02-data-structure.md**](./02-data-structure.md) - データ構造定義、JSON Schema
4. [**03-flowchart-integration.md**](./03-flowchart-integration.md) - フローチャート連携仕様、UI/UX
5. [**04-implementation-plan.md**](./04-implementation-plan.md) - 実装計画、フェーズ別実装詳細

### 関連ドキュメント

- [../gantt-interactive-editing/05-file-protocol-support.md](../gantt-interactive-editing/05-file-protocol-support.md) - postMessage vs LocalStorage 比較
- [../flowchart-manual-editing/README.md](../flowchart-manual-editing/README.md) - フローチャート手動編集仕様
- [../../flowchart-spec.md](../../flowchart-spec.md) - フローチャート全体仕様

---

## 主要機能

### ✅ 実装する機能

| 機能 | 説明 | 優先度 |
|------|------|--------|
| **タスクデータ同期** | task-manager → flowchart へのデータ送信 | 🔴 必須 |
| **フローチャート属性同期** | flowchart → task-manager へのノード情報送信 | 🔴 必須 |
| **リアルタイム更新** | storage イベントによる即座の反映 | 🔴 必須 |
| **競合解決** | タイムスタンプによる最新データの採用 | 🟡 高 |
| **エラーハンドリング** | LocalStorage容量超過、パース失敗の対処 | 🟡 高 |

### ❌ 実装しない機能（明示的除外）

| 機能 | 実装しない理由 |
|------|--------------|
| **Undo/Redo** | 実装コスト高、Phase 2で検討 |
| **バージョン管理** | 複雑すぎる、タイムスタンプのみ |
| **マルチユーザー同期** | ローカル環境前提、サーバー不要 |
| **暗号化** | ローカルストレージ、セキュリティリスク低 |

---

## データ構造概要

### LocalStorageキー

```javascript
// メインタスクデータ
'task-manager-data': {
  tasks: Array<Task>,
  meta: ProjectMeta,
  timestamp: number
}

// フローチャート属性
'flowchart-attributes': {
  [wbsNo: string]: {
    nodeId: string,
    nodeType: 'process' | 'decision' | 'start' | 'end' | 'data',
    position: { x: number, y: number },
    timestamp: number
  }
}

// 更新トリガー
'task-manager-update-trigger': string  // timestamp
'flowchart-update-trigger': string     // timestamp
```

詳細は [02-data-structure.md](./02-data-structure.md) を参照

---

## 実装計画

### Phase 1: 基本連携（6-8h）

1. **task-manager.html 側の実装**
   - LocalStorage保存機能の強化
   - storage イベントハンドラ追加
   - フローチャート属性の統合

2. **flowchart-editor.html 側の実装**
   - タスクデータ読み込み
   - storage イベントハンドラ追加
   - ノード関連付けUI

### Phase 2: エラーハンドリング（2-3h）

1. LocalStorage容量チェック
2. JSONパースエラー処理
3. タイムスタンプ競合解決

### Phase 3: UX改善（3-4h）

1. 同期状態の表示
2. 手動リフレッシュボタン
3. 接続状態インジケーター

**合計工数**: 11-15h（2日間）

---

## 成功基準

### Phase 1完了の定義

- ✅ task-manager でタスク編集すると flowchart が自動更新される
- ✅ flowchart でノード関連付けすると task-manager に反映される
- ✅ ガントチャート変更が flowchart に反映される
- ✅ file://プロトコルで動作する

### Phase 2完了の定義

- ✅ LocalStorage容量超過時にエラー表示
- ✅ 不正なJSONデータでもクラッシュしない
- ✅ 競合時に最新データを採用

### Phase 3完了の定義

- ✅ 同期状態が視覚的に分かる
- ✅ 手動リフレッシュで強制同期できる
- ✅ ユーザーフィードバックが適切

---

## リスクと対策

| リスク | 影響度 | 対策 |
|--------|--------|------|
| LocalStorage容量超過 | 中 | データ圧縮、古いデータ削除 |
| 同一ウィンドウで storage イベント非発火 | 低 | 対象は別HTMLなので問題なし |
| ブラウザ間での非互換性 | 低 | Edge/Chrome 90+を要件とする |
| データ競合 | 中 | タイムスタンプで最新を採用 |
| パフォーマンス劣化 | 低 | 大量タスク時に検証が必要 |

---

## セッション継続のための重要情報

### このディレクトリ構造の目的

新しいClaude Codeセッションでも、以下のファイルを読むだけで完全に文脈を復元できる：

1. **このREADME.md**: 全体像と目次
2. **01-localstorage-design.md**: LocalStorage実装詳細
3. **02-data-structure.md**: データ構造定義
4. **03-flowchart-integration.md**: フローチャート連携UI/UX

### セッション開始時の確認事項

1. 現在のLocalStorage使用状況: task-manager.html の `saveToLocalStorage()`
2. 既存機能: タスクデータの永続化、ガントチャート同期
3. 次に実装する機能: **storage イベントハンドラ追加**
4. アーキテクチャ原則: **既存のLocalStorage実装を拡張**

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-21 | 1.0 | 初版作成 - 仕様書構成確定 |
| 2026-03-21 | 1.1 | データ永続化戦略セクション追加 - 二重保護構造の詳細化 |
| 2026-03-21 | 1.2 | 実装計画追加 - フェーズ別実装詳細、テスト計画 |

---

**最終更新**: 2026-03-21
**次のアクション**: Phase 0実行 - [04-implementation-plan.md](./04-implementation-plan.md) を参照

---

## 補足: データ永続化に関するFAQ

### Q1: LocalStorage容量超過時にデータは消えますか？

**A**: いいえ、**本体ファイルに保存済みのデータは消えません**。

- LocalStorage: リアルタイム同期が一時停止（容量超過）
- 本体ファイル: すべてのデータを保持（影響なし）

### Q2: フローチャート属性はどこに保存されますか？

**A**: 以下の2箇所に保存されます:

1. **LocalStorage** (`flowchart-attributes`) - リアルタイム同期用
2. **本体ファイル** (`task.flowchart`) - 永続保存用

本体ファイルに保存されていれば、LocalStorageをクリアしても復元可能です。

### Q3: 本体ファイル保存を怠るとどうなりますか？

**A**: LocalStorage容量超過時にデータロストのリスクがあります。

- ✅ 推奨: 3秒デバウンス自動保存を実装
- ⚠️ リスク: 手動保存のみの場合、保存忘れの可能性
- 🔧 対策: 未保存変更の警告表示

### Q4: 両HTMLファイル間で付与した属性は本体ファイルに残りますか？

**A**: はい、**本体ファイル保存を怠らなければ完全に残ります**。

```javascript
// task-manager.html で保存時
{
  "tasks": [
    {
      "wbs_no": "WBS1.1.0",
      "task_name": "UI設計",
      "flowchart": {  // ← flowchart側で付与された属性
        "nodeId": "node-process-1",
        "nodeType": "process",
        "position": { "x": 150, "y": 200 }
      }
    }
  ]
}
```

この状態で本体ファイルへ保存されれば、LocalStorageをクリアしても完全に復元できます。
