# Phase 1実装完了レポート - 最小限のデータ連携

---
**feature**: task-manager.html ⇔ flowchart-editor.html データ連携
**phase**: Phase 1 - 最小限のデータ連携
**status**: 実装完了（テスト待ち）
**created**: 2026-03-22
**completed**: 2026-03-22
---

## 📋 実装内容

### 1. 新規追加関数（flowchart-editor.html）

#### `loadTaskManagerData()` (lines 3487-3502)
- **目的**: LocalStorageからtask-manager.htmlのプロジェクトデータを読み込む
- **戻り値**: プロジェクトデータ（存在しない場合はnull）
- **使用箇所**:
  - SVG読み込み完了時（line 5013）- 接続確認のため
  - publishFlowchartAttributes()内（line 3509）- データ連携のため

#### `publishFlowchartAttributes()` (lines 3508-3564)
- **目的**: フローチャート属性（メモ・カスタムラベル・手動エッジ）をtask-manager.htmlへ送信
- **処理フロー**:
  1. task-manager.htmlからプロジェクトデータを読み込み
  2. 各タスクの`mermaid_ids`に紐づくノード情報を収集
  3. メモ・カスタムラベル・関連手動エッジを抽出
  4. LocalStorageへ`flowchart-attributes`として送信
  5. `flowchart-update-trigger`でtask-manager.htmlへ通知
- **使用箇所**:
  - saveToFileAuto()（line 5657）- 自動保存時
  - saveToFileAs()（line 5701）- 名前を付けて保存時

### 2. 修正箇所

#### `saveToFileAuto()` (line 5657)
- **変更内容**: 保存完了後にpublishFlowchartAttributes()を呼び出し
- **効果**: フローチャートの自動保存（Ctrl+S）時にtask-manager.htmlへデータ送信

#### `saveToFileAs()` (line 5701)
- **変更内容**: 保存完了後にpublishFlowchartAttributes()を呼び出し
- **効果**: 名前を付けて保存時にtask-manager.htmlへデータ送信

#### `loadSVGFile()` - reader.onload (line 5013)
- **変更内容**: SVG読み込み完了後にloadTaskManagerData()を呼び出し
- **効果**: SVG読み込み時にtask-manager.htmlとの接続状態をコンソールログで確認

---

## 🔍 データフロー

```
flowchart-editor.html                     task-manager.html
┌────────────────────┐                   ┌────────────────────┐
│ SVG読み込み完了     │                   │                    │
│  ↓                 │                   │                    │
│ loadTaskManager    │←─ localStorage ───│ task-manager-data  │
│ Data()             │    (読み取り)     │ (送信済み)         │
│  ↓                 │                   │                    │
│ コンソールログ出力  │                   │                    │
│ "X tasks"          │                   │                    │
└────────────────────┘                   └────────────────────┘

flowchart-editor.html                     task-manager.html
┌────────────────────┐                   ┌────────────────────┐
│ メモ・ラベル編集    │                   │                    │
│  ↓                 │                   │                    │
│ Ctrl+S             │                   │                    │
│  ↓                 │                   │                    │
│ saveToFileAuto()   │                   │                    │
│  ↓                 │                   │                    │
│ publishFlowchart   │─ localStorage ──→│ storage event      │
│ Attributes()       │  (flowchart-     │  ↓                 │
│                    │   attributes)    │ handleFlowchart    │
│                    │  (update-trigger)│ Update()           │
│                    │                   │  ↓                 │
│                    │                   │ task.flowchart     │
│                    │                   │  更新              │
│                    │                   │  ↓                 │
│                    │                   │ saveToLocal        │
│                    │                   │ Storage()          │
└────────────────────┘                   └────────────────────┘
```

---

## ✅ 成功基準の確認状況

| 基準 | 状態 | 備考 |
|------|------|------|
| ✅ flowchart-editor.htmlで保存したメモがLocalStorageに送信される | 🟡 実装完了 | テスト待ち |
| ✅ task-manager.htmlのタスクにflowchart属性が追加される | 🟡 実装完了 | テスト待ち（受信側は既存実装） |
| ✅ JSONエクスポート時にflowchart属性が含まれる | 🟡 実装完了 | テスト待ち（受信側は既存実装） |
| ✅ コンソールエラーなし | 🟡 実装完了 | テスト待ち |

---

## 🧪 検証シナリオ

### シナリオ1: 基本的なデータ連携テスト

**前提条件**:
- task-manager.htmlで新規プロジェクト作成済み
- タスクに`mermaid_ids`が設定されている（例: "node_1,node_2"）
- flowchart-editor.htmlでSVGファイルを開いている

**検証手順**:

1. **task-manager.htmlでプロジェクト作成**
   - task-manager.htmlを開く
   - 新規タスクを作成（例: WBS1.1.0 "要件定義"）
   - タスク詳細で`mermaid_ids`を設定（例: "node_1"）
   - LocalStorageに保存されることを確認

2. **flowchart-editor.htmlでSVG読み込み**
   - flowchart-editor.htmlを開く
   - SVGファイルをアップロード
   - コンソールログで「Task Manager connected: X tasks」を確認
   - ✅ **期待結果**: タスク数が正しく表示される

3. **ノードにメモ・カスタムラベルを追加**
   - node_1をクリックして選択
   - メモを追加（例: "詳細な要件をヒアリング"）
   - カスタムラベルを設定（例: "要件"）

4. **保存してデータ送信**
   - Ctrl+Sで保存（またはメニューから保存）
   - コンソールログで「Published flowchart attributes for X tasks」を確認
   - ✅ **期待結果**: 送信されたタスク数が表示される

5. **LocalStorageの確認**
   - ブラウザのDevTools → Application → Local Storage
   - `flowchart-attributes`キーを確認
   - ✅ **期待結果**: 以下のようなJSON構造が保存されている
     ```json
     {
       "WBS1.1.0": {
         "nodeIds": ["node_1"],
         "memos": {"node_1": "詳細な要件をヒアリング"},
         "customLabels": {"node_1": "要件"},
         "relatedManualEdges": []
       }
     }
     ```

6. **task-manager.htmlでデータ受信確認**
   - task-manager.htmlに戻る（または更新）
   - タスクWBS1.1.0をJSONエクスポート
   - ✅ **期待結果**: タスクデータに`flowchart`属性が含まれる
     ```json
     {
       "wbs_no": "WBS1.1.0",
       "taskName": "要件定義",
       "mermaid_ids": "node_1",
       "flowchart": {
         "nodeIds": ["node_1"],
         "memos": {"node_1": "詳細な要件をヒアリング"},
         "customLabels": {"node_1": "要件"},
         "relatedManualEdges": []
       }
     }
     ```

### シナリオ2: 手動エッジ連携テスト

**前提条件**:
- シナリオ1の状態から継続
- flowchart-editor.htmlで手動エッジを追加している

**検証手順**:

1. **手動エッジを追加**
   - flowchart-editor.htmlでエッジ追加モード起動
   - node_1からnode_2へ手動エッジを追加
   - ラベルを設定（例: "依存関係"）

2. **保存してデータ送信**
   - Ctrl+Sで保存
   - コンソールログ確認

3. **LocalStorageの確認**
   - `flowchart-attributes`を確認
   - ✅ **期待結果**: `relatedManualEdges`に手動エッジIDが含まれる
     ```json
     {
       "WBS1.1.0": {
         "nodeIds": ["node_1"],
         "memos": {"node_1": "詳細な要件をヒアリング"},
         "customLabels": {"node_1": "要件"},
         "relatedManualEdges": ["edge_12345"]
       }
     }
     ```

4. **task-manager.htmlで確認**
   - JSONエクスポート
   - ✅ **期待結果**: `relatedManualEdges`配列が含まれる

---

## 🐛 想定される問題と対処法

| 問題 | 原因 | 対処法 |
|------|------|--------|
| コンソールに「Task Manager data not found」 | task-manager.htmlが開かれていない、またはデータ未保存 | task-manager.htmlを開いてタスクを作成・保存 |
| 「Task Manager not connected - skipping publish」 | LocalStorageに`task-manager-data`が存在しない | task-manager.htmlで一度保存操作を実行 |
| flowchart属性が送信されない | mermaid_idsが設定されていない | タスク詳細でmermaid_idsを設定 |
| JSONパースエラー | LocalStorageのデータが破損 | LocalStorageをクリアして再作成 |

---

## 📊 コード統計

| 項目 | 数値 |
|------|------|
| 新規追加関数 | 2個 |
| 修正関数 | 3個 |
| 追加行数 | 約90行 |
| コメント行数 | 約30行 |
| 修正ファイル | 1個（flowchart-editor.html） |

---

## 🔗 関連ファイル

- [future-roadmap.md](./future-roadmap.md) - 実装計画の全体像
- [data-integration-investigation.md](./data-integration-investigation.md) - 調査レポート
- `/mnt/c/dev/task-manager/flowchart-editor.html` - 修正対象ファイル
- `/mnt/c/dev/task-manager/task-manager.html` - 受信側（既存実装）

---

## 📝 次のステップ

### 即座に実行すべきこと

1. ✅ **Phase 1: 動作確認とデバッグ**
   - 上記の検証シナリオを実行
   - コンソールログでデータフローを確認
   - LocalStorageの内容を検証
   - エラーがあれば修正

### Phase 1完了後の判断

2. **Phase 1.5: UI改善の判断**
   - Phase 1が正常動作すれば、Phase 1.5へ進む
   - 接続状態表示を追加（flowchart-editor.htmlフッター）
   - 通知メッセージを追加（task-manager.html）

3. **Phase 2以降の判断**
   - Phase 1.5完了後、ユーザーフィードバックを確認
   - 完全統合（タスク一覧パネル、双方向ハイライト）の必要性を判断

---

## 📅 実装履歴

| 日付 | 内容 |
|------|------|
| 2026-03-22 | Phase 1実装完了 - loadTaskManagerData(), publishFlowchartAttributes()追加 |
| 2026-03-22 | saveToFileAuto(), saveToFileAs()修正 - publishFlowchartAttributes()呼び出し追加 |
| 2026-03-22 | loadSVGFile()修正 - SVG読み込み時の接続確認追加 |

---

**最終更新**: 2026-03-22
**ステータス**: Phase 1実装完了（テスト待ち）
**次のアクション**: 検証シナリオ1を実行してデータ連携を確認

