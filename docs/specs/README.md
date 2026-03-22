# Task Manager - 仕様書ディレクトリ

このディレクトリには、Task Managerの各機能の詳細仕様書が格納されています。

---

## 📂 ディレクトリ構成

```
docs/specs/
├── README.md (このファイル)
├── gantt-interactive-editing/      # ガントチャート・インタラクティブ編集機能
│   ├── README.md
│   ├── 01-ui-ux-design.md
│   ├── 02-technical-design.md
│   ├── 03-data-synchronization.md
│   └── 04-implementation-plan.md
├── flowchart-manual-editing/       # フローチャート手動編集機能
│   ├── README.md
│   ├── 01-overview.md
│   ├── 02-manual-edge.md
│   ├── 04-ai-integrated-export.md
│   ├── 05-data-structure.md
│   └── 06-implementation-plan.md
├── data-integration/               # データ連携機能（task-manager ⇔ flowchart-editor）
│   ├── README.md
│   ├── future-roadmap.md
│   ├── phase1-implementation-summary.md
│   └── phase1-mermaid-ids-fix.md
├── cross-html-sync/                # ⚠️ 陳腐化（data-integration/に統合済み）
└── versions/                       # バージョン別仕様（将来用）
```

---

## 📋 機能別仕様書

### 1. ガントチャート・インタラクティブ編集機能

**ステータス**: 📋 仕様確定、実装待ち
**バージョン**: 1.0
**工数見積もり**: 15-21h（2-3日間）

ガントチャート専用画面で、日程バーを直接ドラッグ&リサイズして日程調整を行える機能。

#### 仕様書一覧
- [README.md](./gantt-interactive-editing/README.md) - 全体概要
- [01-ui-ux-design.md](./gantt-interactive-editing/01-ui-ux-design.md) - UI/UX設計（ASCIIアート、操作イメージ）
- [02-technical-design.md](./gantt-interactive-editing/02-technical-design.md) - 技術設計（座標変換、ドラッグ処理）
- [03-data-synchronization.md](./gantt-interactive-editing/03-data-synchronization.md) - データ同期・整合性管理
- [04-implementation-plan.md](./gantt-interactive-editing/04-implementation-plan.md) - 実装計画・工数見積もり

#### 主要機能
- ✅ 日程バー左端リサイズ（開始日変更）
- ✅ 日程バー右端リサイズ（終了日変更）
- ✅ 日程バー中央ドラッグ（期間維持して移動）
- ✅ リアルタイムプレビュー
- ✅ メイン画面への即時反映

---

### 2. フローチャート手動編集機能

**ステータス**: 📋 仕様確定、実装待ち
**バージョン**: 2.0
**工数見積もり**: 24-29h（3-4日間、MVP版）

Mermaid生成SVGフローチャートに対して、メモノード・手動エッジを追加し、AI解析用統合JSONを出力する機能。

#### 仕様書一覧
- [README.md](./flowchart-manual-editing/README.md) - 全体概要
- [01-overview.md](./flowchart-manual-editing/01-overview.md) - 全体設計・方針
- [02-manual-edge.md](./flowchart-manual-editing/02-manual-edge.md) - 手動エッジ仕様
- [04-ai-integrated-export.md](./flowchart-manual-editing/04-ai-integrated-export.md) - AI解析用統合JSON出力
- [05-data-structure.md](./flowchart-manual-editing/05-data-structure.md) - データ構造定義
- [06-implementation-plan.md](./flowchart-manual-editing/06-implementation-plan.md) - 実装計画・工数見積もり

#### 主要機能
- ✅ メモノード作成・ドラッグ移動（Phase 1）
- ✅ 手動エッジ端点編集（Phase 2）
- ✅ AI解析用統合JSON出力（Phase 3-AI MVP）

---

### 3. データ連携機能（task-manager.html ⇔ flowchart-editor.html）

**ステータス**: ✅ Phase 1完了、⏳ Phase 2-A仕様策定完了
**バージョン**: v2.15.0（Phase 1基本実装）+ v2.16.0予定（Phase 1修正完了） → v2.17.0予定（Phase 2-A）
**工数見積もり**: Phase 0-1: 完了、Phase 2-A: 2-2.5h

タスクマネージャーとフローチャートエディター間でタスク情報・フローチャート属性を双方向同期する機能。LocalStorageとstorage eventを用いた軽量なリアルタイム連携を実現。

#### 仕様書一覧
- [README.md](./data-integration/README.md) - 全体概要、実装ステータス、FAQ
- [future-roadmap.md](./data-integration/future-roadmap.md) - Phase 0-3の実装計画、技術選定の背景
- [phase1-implementation-summary.md](./data-integration/phase1-implementation-summary.md) - Phase 1基本実装レポート
- [phase1-mermaid-ids-fix.md](./data-integration/phase1-mermaid-ids-fix.md) - Phase 1修正仕様
- [phase2a-task-list-sync.md](./data-integration/phase2a-task-list-sync.md) - ⭐Phase 2-A実装仕様（最新）

#### 実装フェーズ
- ✅ Phase 0: ドキュメント整備・実装計画策定（完了）
- ✅ Phase 1: flowchart-editor → task-manager データ連携（完了）
- 📋 Phase 1.5: UI改善（接続状態表示・通知）
- ⏳ **Phase 2-A**: task-manager → flowchart-editor タスク一覧送信（仕様策定完了）
- 📋 Phase 2-B: 双方向ハイライト（ノード↔タスク）
- 📋 Phase 2-C: タスク一覧パネルUI改善
- 📋 Phase 2-D: manualEdges/Nodesのデータ統合
- 📋 Phase 2-E: 統合エクスポート機能
- 📋 Phase 3: AI統合JSON出力

#### 主要機能（実装済み）
- ✅ フローチャートエディターでノードにメモ・カスタムラベルを追加
- ✅ タスクマネージャーでflowchart属性として受信・保存
- ✅ JSONエクスポート時にflowchart属性を含める
- ✅ **mermaid_ids双方向同期**（Phase 1修正で実装完了）

#### 次のステップ
⭐ **Phase 2-A実装**: A側の全タスクをB側の左サイドパネルに自動表示する機能を実装します。詳細は [phase2a-task-list-sync.md](./data-integration/phase2a-task-list-sync.md) を参照してください。

---

## 🔗 関連ドキュメント

### プロジェクトルート
- [README.md](../../README.md) - プロジェクト全体概要
- [CHANGELOG.md](../../CHANGELOG.md) - 変更履歴（v1.0.0～現在）

### その他ドキュメント
- [docs/flowchart-spec.md](../flowchart-spec.md) - フローチャート基本仕様
- [docs/task-edit-ui-flowchart-link.md](../task-edit-ui-flowchart-link.md) - タスク・フローチャート連携仕様

### アーカイブ
- [docs/archive/](../archive/) - 過去のセッション記録、外部AI有識者フィードバック

---

## 📊 仕様書の読み方

### 新しいセッションを開始する場合

1. **機能概要を把握**: 各ディレクトリの `README.md` を読む
2. **実装計画を確認**: `04-implementation-plan.md` または `06-implementation-plan.md` を読む
3. **詳細設計を理解**: 必要に応じて個別の仕様書（UI/UX、技術設計等）を参照

### 実装を開始する場合

1. **Phase 1から着手**: 実装計画のPhase 1セクションを読む
2. **技術設計を参照**: `02-technical-design.md` で実装詳細を確認
3. **データ構造を確認**: `05-data-structure.md` または `03-data-synchronization.md` で整合性を確認

---

## 🎯 優先順位

| 機能 | 優先度 | 理由 |
|------|--------|------|
| データ連携（Phase 2-A） | 🔴 最優先 | Phase 1完了、タスク一覧同期の実装（工数2-2.5h） |
| ガントチャート・インタラクティブ編集 | 🔴 高 | ユーザー要望あり、既存機能の強化 |
| データ連携（Phase 2-B以降） | 🟡 中 | Phase 2-A完了後に実装可能 |
| フローチャート手動編集 | 🟡 中 | 新機能、AI連携の実験的要素 |

---

## 📝 仕様書テンプレート

新しい機能の仕様書を作成する場合は、以下の構成を推奨：

```
新機能名/
├── README.md                    # 全体概要、目次
├── 01-ui-ux-design.md          # UI/UX設計
├── 02-technical-design.md      # 技術設計
├── 03-data-structure.md        # データ構造
├── 04-implementation-plan.md   # 実装計画・工数見積もり
└── 05-test-plan.md (optional)  # テスト計画
```

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | Phase 2-A仕様書追加（phase2a-task-list-sync.md） |
| 2026-03-22 | Phase 1完了を反映、Phase 2-A仕様策定完了を明記 |
| 2026-03-22 | 優先順位テーブル更新（Phase 2-Aを最優先に） |
| 2026-03-22 | データ連携機能の仕様書追加（data-integration/） |
| 2026-03-22 | Phase 1修正仕様書追加（phase1-mermaid-ids-fix.md） |
| 2026-03-22 | cross-html-sync/を陳腐化として明記 |
| 2026-03-21 | ガントチャート・インタラクティブ編集機能の仕様書追加 |
| 2026-03-21 | フローチャート手動編集機能の仕様書整理 |
| 2026-03-21 | 本README.md作成 |

---

**最終更新**: 2026-03-22
