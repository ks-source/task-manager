# データ連携機能 - task-manager.html ⇔ flowchart-editor.html

---

## 📖 概要

このディレクトリは、タスクマネージャー（task-manager.html）とフローチャートエディター（flowchart-editor.html）間のデータ連携機能の仕様・実装計画・実装レポートをまとめています。

### 連携の目的

1. **タスク情報の共有**: タスクマネージャーで管理するタスク一覧をフローチャートエディターで参照
2. **フローチャート属性の同期**: フローチャートで追加したメモ・ラベル・リンク情報をタスクに反映
3. **双方向の即時更新**: LocalStorageとstorage eventを使った軽量なリアルタイム連携
4. **統合的なプロジェクト管理**: WBS（タスク階層）とフローチャート（処理フロー）の一体化

---

## 🚦 実装ステータス

| Phase | 内容 | ステータス | 完了日 |
|-------|------|-----------|--------|
| **Phase 0** | ドキュメント整備・実装計画策定 | ✅ **完了** | 2026-03-22 |
| **Phase 1** | flowchart-editor → task-manager データ連携 | ✅ **完了** | 2026-03-22 |
| **Phase 1.5** | UI改善（接続状態表示・通知） | 📋 計画中 | - |
| **Phase 2-A** | task-manager → flowchart-editor タスク一覧送信 | ⏳ **仕様策定中** | - |
| **Phase 2-B** | 双方向ハイライト（ノード↔タスク） | 📋 計画中 | - |
| **Phase 2-C** | タスク一覧パネルUI改善 | 📋 計画中 | - |
| **Phase 2-D** | manualEdges/Nodesのデータ統合 | 📋 計画中 | - |
| **Phase 2-E** | 統合エクスポート機能 | 📋 計画中 | - |
| **Phase 3** | AI統合JSON出力 | 📋 構想中 | - |

### Phase 1の状況 ✅ 完了

**実装済み（v2.15.0 + v2.16.0予定）**:
- ✅ flowchart-editor.htmlでメモ・カスタムラベルをLocalStorageへ送信
- ✅ task-manager.htmlでflowchart属性を受信・保存
- ✅ JSONエクスポート時にflowchart属性を含める
- ✅ **mermaid_ids双方向同期**（Phase 1修正で追加）

Phase 1は基本実装（v2.15.0）とmermaid_ids修正（v2.16.0予定）の両方が完了済みです。

詳細:
- 基本実装: [phase1-implementation-summary.md](./phase1-implementation-summary.md)
- 修正: [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md)

### Phase 2-Aの状況 ⏳ 仕様策定中

**目的**: A側の全タスクをB側の左サイドパネルに自動表示

**実装予定内容**:
- A側のタスクリストをLocalStorageから読み込み
- mockTasksをA側データで動的に置き換え
- mermaid_ids空のタスクも表示
- タスク追加・削除時の自動更新（storage event）

**工数見積もり**: 2-2.5時間

詳細: [phase2a-task-list-sync.md](./phase2a-task-list-sync.md)

---

## 📚 ドキュメント一覧

### 1. [future-roadmap.md](./future-roadmap.md)
**種類**: 実装計画書
**作成日**: 2026-03-22
**内容**:
- 全Phase（0〜3）の実装計画
- 各Phaseの目標・成功基準・工数見積もり
- データフロー図
- 技術選定の背景（LocalStorage選定理由）
- リスク分析と対策

**こんな時に読む**:
- 全体像を把握したい時
- 次のPhaseの実装計画を確認したい時
- 工数見積もりが必要な時

---

### 2. [phase1-implementation-summary.md](./phase1-implementation-summary.md)
**種類**: 実装完了レポート
**作成日**: 2026-03-22
**内容**:
- Phase 1で実装した関数の詳細（行番号付き）
- データフロー図（flowchart-editor → task-manager）
- 成功基準の確認状況
- 検証シナリオ（基本連携テスト・手動エッジ連携テスト）
- 想定される問題と対処法

**こんな時に読む**:
- Phase 1の実装内容を確認したい時
- デバッグ方法を知りたい時
- テストシナリオを実行したい時

**⚠️ 注意**: このレポートはmermaid_ids同期のバグ発見前に作成されたため、Phase 1が「完了」と記載されていますが、実際には**修正が必要**です。最新の修正仕様は [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) を参照してください。

---

### 3. [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) ⭐ **最新**
**種類**: 修正仕様書
**作成日**: 2026-03-22
**内容**:
- **問題の発見**: Phase 1実装でmermaid_ids同期が欠落していた経緯
- **現状分析**: flowchart-editorのタスク編集機能は完璧に動作しているが、データ送信時に反映されていない
- **技術仕様**: publishFlowchartAttributes()とhandleFlowchartUpdate()の修正内容（Before/After比較）
- **データスキーマ**: TypeScriptインターフェースで定義された完全なデータ構造
- **実装計画**: 2つの修正（工数1-1.5時間）
- **検証シナリオ**: mermaid_ids編集 → 保存 → task-manager受信の確認手順
- **リスク分析**: データ破損・後方互換性の考慮

**こんな時に読む**:
- **Phase 1の修正を実装する前に必読**
- mermaid_ids双方向同期の仕組みを理解したい時
- データスキーマの全体像を確認したい時

---

### 4. [phase2a-task-list-sync.md](./phase2a-task-list-sync.md) ⭐ **最新**
**種類**: 実装仕様書
**作成日**: 2026-03-22
**内容**:
- **概要**: A→Bタスク一覧同期の詳細実装仕様
- **現状分析**: mockTasksのハードコード問題
- **技術仕様**: initializeTasksFromTaskManager(), parseMermaidIds()等の実装詳細（Before/After）
- **データフロー**: Phase 2-A実装前後の比較図
- **データスキーマ**: task-manager-data → mockTasks変換ロジック
- **実装計画**: 工数2-2.5時間、詳細な実装順序
- **検証シナリオ**: 4つの統合テストシナリオ

**こんな時に読む**:
- **Phase 2-Aの実装を開始する前に必読**
- mockTasksを実データで置き換える方法を知りたい時
- storage eventリスナーの実装方法を確認したい時

---

## 🔄 データフロー概要

### 現在の実装（Phase 1 - 修正前）

```
flowchart-editor.html                     task-manager.html
┌────────────────────┐                   ┌────────────────────┐
│ ユーザーがノードに  │                   │                    │
│ メモ・ラベル追加    │                   │                    │
│  ↓                 │                   │                    │
│ Ctrl+S で保存       │                   │                    │
│  ↓                 │                   │                    │
│ publishFlowchart   │─ localStorage ──→│ storage event      │
│ Attributes()       │  (flowchart-     │  ↓                 │
│                    │   attributes)    │ handleFlowchart    │
│ ✅ メモ・ラベル送信 │                   │ Update()           │
│ ❌ mermaid_ids送信  │                   │  ↓                 │
│                    │                   │ task.flowchart{}   │
│                    │                   │ 更新（メモ・ラベル）│
│                    │                   │ ❌ mermaid_ids     │
│                    │                   │    更新なし        │
└────────────────────┘                   └────────────────────┘
```

### Phase 1修正後の目標

```
flowchart-editor.html                     task-manager.html
┌────────────────────┐                   ┌────────────────────┐
│ ユーザーがタスク編集│                   │                    │
│ モーダルでノード    │                   │                    │
│ 追加・削除          │                   │                    │
│  ↓                 │                   │                    │
│ mermaidIds更新     │                   │                    │
│  ↓                 │                   │                    │
│ Ctrl+S で保存       │                   │                    │
│  ↓                 │                   │                    │
│ publishFlowchart   │─ localStorage ──→│ storage event      │
│ Attributes()       │  (flowchart-     │  ↓                 │
│                    │   attributes)    │ handleFlowchart    │
│ ✅ メモ・ラベル送信 │                   │ Update()           │
│ ✅ mermaid_ids送信  │                   │  ↓                 │
│                    │                   │ task.flowchart{}   │
│                    │                   │ 更新（すべて）      │
│                    │                   │ ✅ task.mermaid_ids│
│                    │                   │    更新            │
└────────────────────┘                   └────────────────────┘
```

---

## 🚀 クイックスタート（開発者向け）

### Phase 1修正を実装する場合

1. **仕様を理解**
   [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) を熟読

2. **修正箇所を確認**
   - `/mnt/c/dev/task-manager/flowchart-editor.html`
     - `publishFlowchartAttributes()` (lines 3508-3564)
   - `/mnt/c/dev/task-manager/task-manager.html`
     - `handleFlowchartUpdate()` (lines 1722-1759)

3. **修正を実装**
   仕様書のBefore/Afterコードを参照して実装

4. **検証**
   [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) の検証シナリオ1を実行

5. **コミット**
   ```bash
   git add flowchart-editor.html task-manager.html
   git commit -m "fix: mermaid_ids双方向同期を追加 - Phase 1修正完了"
   ```

### Phase 2-A（タスク一覧送信）を実装する場合 ⭐ 次のステップ

1. **仕様を熟読**
   [phase2a-task-list-sync.md](./phase2a-task-list-sync.md)

2. **Phase 1修正が完了していることを確認**
   mermaid_ids同期が動作していないとPhase 2-Aのデータ整合性が保証されません

3. **修正箇所を確認**
   - `/mnt/c/dev/task-manager/flowchart-editor.html`
     - `parseMermaidIds()` 新規追加
     - `initializeTasksFromTaskManager()` 新規追加
     - SVG読み込み時の初期化処理修正
     - storage eventリスナー追加

4. **実装**
   仕様書のBefore/Afterコードを参照して実装

5. **検証**
   [phase2a-task-list-sync.md](./phase2a-task-list-sync.md) の検証シナリオ1-4を実行

6. **コミット**
   ```bash
   git add flowchart-editor.html
   git commit -m "feat: Phase 2-A タスク一覧同期（A→B）を実装"
   git tag v2.17.0
   ```

---

## 🔗 関連ドキュメント

### 陳腐化したドキュメント

- **[../cross-html-sync/](../cross-html-sync/)**
  **作成日**: 2026-03-21
  **ステータス**: ⚠️ 陳腐化（設計のみで実装なし）
  **理由**: data-integration/ で実装が完了したため、このディレクトリは参考資料としてのみ保持
  **今後の対応**: README.mdに陳腐化の旨を明記予定

### 本機能に関連するCHANGELOG

- [/mnt/c/dev/task-manager/CHANGELOG.md](../../../CHANGELOG.md)
  - v2.15.0: Phase 1データ連携機能実装
  - v2.14.0: Mermaid IDs UI追加（task-manager.html）
  - v2.12.0-v2.13.0: 関連する前準備

---

## 📝 貢献ガイドライン

### 新しいPhaseを実装する時

1. **future-roadmap.mdを更新**
   実装計画を最新化

2. **実装**

3. **phase{N}-implementation-summary.mdを作成**
   実装内容を詳細に記録

4. **このREADME.mdを更新**
   - ステータステーブルを更新
   - ドキュメント一覧に追加

5. **CHANGELOGを更新**

### バグを発見した時

1. **phase{N}-{bug-name}-fix.mdを作成**
   問題分析・修正仕様を記述

2. **このREADME.mdのステータスを更新**

3. **修正後、ステータスを「完了」に戻す**

---

## 🏷️ タグ・バージョン

| バージョン | タグ | 内容 | 日付 |
|-----------|------|------|------|
| v2.15.0 | ✅ | Phase 1データ連携実装（mermaid_ids修正前） | 2026-03-22 |
| v2.16.0 | 📋 予定 | Phase 1修正完了（mermaid_ids双方向同期） | - |
| v2.17.0 | 📋 予定 | Phase 1.5 UI改善 | - |
| v2.18.0 | 📋 予定 | Phase 2-A タスク一覧送信 | - |

---

## ❓ FAQ

### Q1: LocalStorageを使う理由は？postMessage APIではダメ？

**A**: [future-roadmap.md](./future-roadmap.md) の「技術選定の背景」セクションを参照。LocalStorageの方がシンプルで、同一オリジン内での永続化が容易なため採用。

### Q2: Phase 1は完了しているの？修正中なの？

**A**: Phase 1は「基本実装は完了」していますが、**mermaid_ids双方向同期のバグ**が発見されたため、現在修正中です。[phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) を参照。

### Q3: Phase 2-Aを先に実装してもいい？

**A**: **推奨しません**。Phase 1修正（mermaid_ids同期）が完了していないと、Phase 2-AでA→Bにタスクを送信しても、B→Aの返信でmermaid_idsが失われます。Phase 1修正を優先してください。

### Q4: task-manager.htmlとflowchart-editor.htmlのデータ構造の違いは？

**A**: [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) の「データスキーマ」セクションにTypeScriptインターフェースで定義されています。

### Q5: テスト方法は？

**A**: [phase1-implementation-summary.md](./phase1-implementation-summary.md) および [phase1-mermaid-ids-fix.md](./phase1-mermaid-ids-fix.md) の検証シナリオを参照。

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成 - Phase 0-1の実装状況まとめ |
| 2026-03-22 | Phase 1修正（mermaid_ids双方向同期）の仕様追加 |
| 2026-03-22 | Phase 1完了を反映、Phase 2-A仕様策定完了 |
| 2026-03-22 | phase2a-task-list-sync.md追加、実装ステータス更新 |

---

**最終更新**: 2026-03-22
**ステータス**: Phase 1完了（v2.16.0予定）、Phase 2-A仕様策定完了
**次のアクション**: Phase 2-A実装（タスク一覧同期 A→B）
**担当者**: Claude Code
**レビュアー**: ユーザー様
