# データ連携機能 - 今後のロードマップ

---
**feature**: task-manager.html ⇔ flowchart-editor.html データ連携
**version**: 1.0
**status**: planning
**created**: 2026-03-22
**updated**: 2026-03-22
**author**: Claude Code Session
---

## 📋 概要

task-manager.htmlとflowchart-editor.htmlの間でデータを連携し、タスク管理とフローチャート編集をシームレスに統合するための実装計画。

## 🎯 現状分析

### 完成度

| コンポーネント | 完成度 | 状態 |
|--------------|--------|------|
| **task-manager.html** | 95% | プロジェクト名管理完了、データ連携受信側準備済み |
| **flowchart-editor.html** | 85% | メモ・エッジ・ノード編集完了、データ連携送信未実装 |
| **双方向データ連携** | 20% | 受信側のみ実装、送信側未実装 |
| **統合エクスポート** | 30% | 構造は準備済みだがデータ未統合 |

### 技術的負債

| 負債 | 影響度 | 対策優先度 |
|------|--------|-----------|
| flowchart-attributesの送信未実装 | 🔴 高 | 最優先 |
| manualEdges/Nodesの統合未実装 | 🟡 中 | Phase 2 |
| WBS↔ノードの自動紐付け未実装 | 🟢 低 | Phase 3以降 |

---

## 📊 Phase別実装計画

### Phase 0: ドキュメント整備（完了）

**目的**: 現状の仕様を明文化し、次期開発者が理解できるようにする

**実装内容**:
- ✅ データ連携調査レポート作成
- ✅ 今後のステップ文書作成（本文書）
- ✅ データ構造仕様の整理

**工数**: 2-3時間

---

### Phase 1: 最小限のデータ連携（実装中）

**目的**: flowchart-editor.htmlで編集したメモ・ラベルをtask-manager.htmlで参照できるようにする

**実装内容**:

#### 1. flowchart-editor.htmlの修正

**追加関数**:

1. `loadTaskManagerData()` - タスクマネージャーデータの読み込み
2. `publishFlowchartAttributes()` - フローチャート属性の送信
3. `updateTaskManagerConnectionStatus()` - 接続状態の表示（Phase 1.5）

**修正箇所**:
- `saveToFileAuto()` - 自動保存時にpublishFlowchartAttributes()を呼び出し
- `saveToFileAs()` - 名前を付けて保存時にpublishFlowchartAttributes()を呼び出し
- SVG読み込み完了時 - loadTaskManagerData()で接続確認

**データフロー**:
```
flowchart-editor.html                     task-manager.html
┌────────────────────┐                   ┌────────────────────┐
│ メモ・ラベル編集    │                   │                    │
│  ↓                 │                   │                    │
│ saveToFileAuto()   │                   │                    │
│  ↓                 │                   │                    │
│ publishFlowchart   │─ localStorage ──→│ handleFlowchart    │
│ Attributes()       │  (flowchart-     │ Update()           │
│                    │   attributes)    │  ↓                 │
│                    │                   │ task.flowchart     │
│                    │                   │  ↓                 │
│                    │                   │ saveToLocalStorage │
└────────────────────┘                   └────────────────────┘
```

**成功基準**:
- ✅ flowchart-editor.htmlで保存したメモがLocalStorageに送信される
- ✅ task-manager.htmlのタスクにflowchart属性が追加される
- ✅ JSONエクスポート時にflowchart属性が含まれる
- ✅ コンソールエラーなし

**工数**: 3-4時間
- flowchart-editor.html修正: 2-2.5時間
- 動作確認・デバッグ: 1-1.5時間

**リスク**: 🟢 低
- 既存機能への影響なし
- 受信側は既に実装済み
- ロールバック容易

---

### Phase 1.5: UI改善・エラーハンドリング（Phase 1後）

**目的**: ユーザーに連携状態を可視化し、エラーを適切に処理する

**実装内容**:

#### 1. flowchart-editor.htmlのステータス表示

接続状態インジケーターをフッターに追加:
- ● Task Manager: 12タスク (新規プロジェクト) - 緑色: 接続中
- ○ Task Manager: 未接続 - 灰色: 未接続

#### 2. task-manager.htmlの通知メッセージ

フローチャート情報統合時に通知を表示:
- "📊 フローチャート: 5件のタスク情報を更新しました"

**工数**: 1-2時間

**成功基準**:
- ✅ flowchart-editor.htmlで接続状態が表示される
- ✅ task-manager.htmlで更新通知が表示される
- ✅ エラー時に適切なメッセージが表示される

**リスク**: 🟢 低

---

### Phase 2: 完全統合（中期：2-3週間後）

**目的**: タスクマネージャーとフローチャートエディターを完全に統合し、双方向のシームレスな連携を実現

**実装内容**:

#### 1. flowchart-editor.htmlにタスク一覧パネルを追加

**UI構成**:
```
┌─────────────────────────────────────────────────┐
│ Flowchart Editor                    [×]         │
├──────────┬──────────────────────────────────────┤
│タスク一覧│         SVGキャンバス                 │
│          │                                      │
│☑ WBS1.1.0│   ┌─────┐                           │
│ 要件定義  │   │node1│                           │
│ 📌 F_01  │   └─────┘                           │
│          │      ↓                               │
│☐ WBS1.2.0│   ┌─────┐                           │
│ 設計     │   │node2│                           │
│          │   └─────┘                           │
└──────────┴──────────────────────────────────────┘
```

**実装**:
- タスク一覧の動的生成
- タスククリック時にノードをハイライト
- ノードクリック時にタスクをハイライト
- ステータスカラーの同期

**工数**: 6-8時間

#### 2. manualEdges/Nodesのプロジェクトデータへの統合

**データ構造拡張**:
```json
{
  "meta": { ... },
  "tasks": [ ... ],
  "flowchart": {
    "svgFile": "flowchart.svg",
    "lastUpdated": "2026-03-22T10:30:00.000Z",
    "manualEdges": { ... },
    "manualNodes": { ... }
  }
}
```

**工数**: 4-6時間

#### 3. 統合エクスポート機能

新しいエクスポート形式:
- 📄 JSON形式（完全データ）- プロジェクト名・タスク・フローチャートデータ含む
- 📊 CSV形式（Excel互換）- タスクリストのみ
- 🎨 フローチャート単体（SVG + JSON）- SVG画像とメモデータのみ

**工数**: 2-3時間

**Phase 2合計工数**: 12-17時間（2-3日間）

**成功基準**:
- ✅ flowchart-editor.htmlでタスク一覧が表示される
- ✅ ノード↔タスク間の双方向ハイライトが機能する
- ✅ manualEdges/Nodesがプロジェクトデータに統合される
- ✅ JSON一括エクスポート時に完全なデータが出力される
- ✅ インポート時にすべてのデータが復元される

**リスク**: 🟡 中
- データ構造の複雑化
- 既存エクスポート機能への影響
- ファイルサイズの増大

---

### Phase 3: AI統合JSON出力（長期：1-2ヶ月後）

**目的**: AI解析用の最適化されたJSONフォーマットを出力し、外部AIによるフロー最適化・ドキュメント生成を可能にする

**仕様書参照**: `docs/specs/flowchart-manual-editing/04-ai-integrated-export.md`

**実装内容**:
- Mermaidノード座標抽出
- Mermaidエッジ情報抽出
- タスク情報との紐付け
- AI解析用コンテキスト情報の付与

**工数**: 24-29時間（仕様書記載）

**成功基準**:
- ✅ AI解析用JSONが出力される
- ✅ 座標・スタイル情報が含まれる
- ✅ タスク情報との紐付けが明確
- ✅ 外部AI（Claude/GPT）で解析可能

**リスク**: 🟡 中
- AI連携の必要性が不明確
- データ構造の複雑化

---

## 🔍 検証方法

### Phase 1の検証シナリオ

1. task-manager.htmlで新規プロジェクト作成
2. タスクにmermaid_idsを設定（例: "node_1,node_2"）
3. JSONエクスポートしてLocalStorageに保存確認
4. flowchart-editor.htmlでSVGを開く
5. コンソールで "Task Manager connected: X tasks" を確認
6. ノードにメモ・カスタムラベルを追加
7. 保存（Ctrl+S）
8. LocalStorageの`flowchart-attributes`を確認
9. task-manager.htmlに戻る
10. タスクをJSONエクスポート
11. flowchart属性が含まれていることを確認

**期待結果**:
```json
{
  "tasks": [
    {
      "wbs_no": "WBS1.1.0",
      "taskName": "要件定義",
      "mermaid_ids": "node_1",
      "flowchart": {
        "nodeIds": ["node_1"],
        "memos": {"node_1": "詳細メモ"},
        "customLabels": {},
        "relatedManualEdges": []
      }
    }
  ]
}
```

---

## ⏱️ タイムライン（推奨スケジュール）

```
Week 0 (現在)
├─ ✅ Phase 0: ドキュメント整備
└─ ✅ データ連携調査完了

Week 1
├─ 📋 Phase 1: 最小連携実装（3-4h）
│   ├─ Day 1: flowchart-editor.html修正（2h）
│   ├─ Day 2: 動作確認・デバッグ（1-2h）
│   └─ Day 3: Phase 1.5: UI改善（1-2h）
└─ 🎯 マイルストーン: 基本連携完了

Week 2-3（余裕があれば）
├─ 📋 Phase 2: 完全統合（12-17h）
│   ├─ タスク一覧表示（6-8h）
│   ├─ manualEdges/Nodes統合（4-6h）
│   └─ 統合エクスポート（2-3h）
└─ 🎯 マイルストーン: 完全統合完了

Month 2-3（将来）
├─ 📋 Phase 3: AI統合JSON（24-29h）
└─ 🎯 マイルストーン: AI連携完了
```

---

## ⚠️ リスク管理

### 技術的リスク

| リスク | 影響度 | 発生確率 | 対策 |
|--------|--------|---------|------|
| LocalStorageサイズ制限（5-10MB） | 高 | 中 | データ圧縮、大規模プロジェクト時の警告表示 |
| 同時編集時のデータ競合 | 中 | 低 | タイムスタンプ比較、競合検出ロジック |
| ブラウザ互換性問題 | 低 | 低 | Edge/Chrome 90+に限定（既定方針） |
| JSON構造の後方互換性 | 中 | 中 | バージョン管理、マイグレーション関数 |

### スケジュールリスク

| リスク | 影響度 | 対策 |
|--------|--------|------|
| Phase 1の遅延 | 高 | Phase 1を最優先、他は後回し可 |
| 仕様変更要求 | 中 | MVPリリース後のフィードバックで判断 |
| テスト工数不足 | 中 | 自動テストは見送り、手動テスト重視 |

---

## 📝 意思決定ポイント

### Phase 1後の判断

**Q: Phase 2（完全統合）を実施するか？**
- 判断基準:
  - Phase 1で十分か？
  - タスク一覧表示が必要か？
  - 双方向ハイライトが必要か？
- 判断時期: Phase 1完了後、ユーザーフィードバック確認後

**Q: Phase 3（AI統合）を実施するか？**
- 判断基準:
  - 外部AI活用の具体的なユースケースがあるか？
  - AI解析の効果が見込めるか？
- 判断時期: Phase 2完了後、AI活用方針確定後

---

## 🔗 関連ドキュメント

- [データ連携調査レポート](./data-integration-investigation.md) - 現状分析の詳細
- [LocalStorage通信プロトコル](./localStorage-protocol.md) - 通信仕様（作成予定）
- [フローチャート手動編集仕様](../flowchart-manual-editing/README.md) - flowchart-editor.html機能仕様
- [タスクマネージャー仕様](../../README.md) - task-manager.html全体仕様

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成 - Phase 0～3の実装計画策定 |

---

**最終更新**: 2026-03-22
**ステータス**: Phase 1実装中
**次のアクション**: flowchart-editor.htmlにpublishFlowchartAttributes()を実装
