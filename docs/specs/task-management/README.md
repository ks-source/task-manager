# タスク管理機能 - コア仕様書

---

**ステータス**: ✅ 実装済み（v2.13.4時点）
**バージョン**: 1.0
**対象HTML**: task-manager.html
**作成日**: 2026-03-22
**最終更新**: 2026-03-22

---

## 📖 概要

Task Managerのタスク管理機能のコア仕様を定義します。このディレクトリには、タスクデータ構造、WBS番号体系、削除ポリシー等、タスク管理の基本的な仕様が含まれます。

### 対象範囲

- タスクデータ構造（JSON Schema、全フィールド定義）
- WBS番号体系（階層構造、自動生成アルゴリズム）
- 削除ポリシー（論理削除、ID再利用管理）
- タスクステータス管理（将来拡張）
- タスク復元機能（将来拡張）

---

## 📚 ドキュメント一覧

### 1. [01-data-structure.md](./01-data-structure.md)
**種類**: データ構造定義書
**内容**:
- タスクオブジェクトの完全なJSON Schema定義
- 全フィールド（`wbs_no`, `deleted`, `deleted_at`, `predecessors`, `flowchart`等）の詳細説明
- プロジェクトデータの全体構造（`meta`, `tasks`配列）
- データバージョン管理・マイグレーション戦略

**こんな時に読む**:
- タスクデータの構造を確認したい時
- JSONエクスポート・インポートの仕様を確認したい時
- 新しいフィールドを追加する前に既存構造を把握したい時

---

### 2. [02-wbs-numbering-system.md](./02-wbs-numbering-system.md)
**種類**: WBS番号体系仕様書
**内容**:
- WBS番号の構造定義（`{Phase}.{Major}.{Minor}`）
- 自動生成アルゴリズム（`generateWBSNumber()` 関数仕様）
- ⚠️ **現在のバグ**（Opus指摘: Minor番号をMajor番号として使用）
- 修正後のアルゴリズム
- Phase/Major/Minor番号の抽出ロジック
- 手動入力時のバリデーション

**こんな時に読む**:
- WBS番号の自動生成ロジックを理解したい時
- WBS番号のバグ修正を実装する時
- 新しいタスクのWBS番号を決定する時

---

### 3. [03-deletion-policy.md](./03-deletion-policy.md) ⭐ **重要**
**種類**: 削除ポリシー仕様書
**内容**:
- **採用方針**: Option 1（削除済みID除外 + 全履歴考慮）
- ID不変性の原則（業界標準準拠）
- 論理削除の実装仕様（`deleted: true`, `deleted_at`フィールド）
- WBS番号自動生成時の削除済みタスク扱い
- 実装修正内容（`generateWBSNumber()` の1行変更）
- 外部AI評価結果の要約（Opus, GPT, Gemini）
- 将来の拡張計画（復元機能、ステータス管理、内部ID分離）

**こんな時に読む**:
- **削除機能の実装を修正する前に必読**
- WBS番号が再利用される理由を理解したい時
- 削除済みタスクの扱いを確認したい時
- 外部AI評価結果を確認したい時

---

## 🔗 関連ドキュメント

### プロジェクト内
- [データ連携仕様](../data-integration/) - フローチャートエディターとの連携
- [ガントチャート機能](../gantt-interactive-editing/) - 日程バー編集機能
- [フローチャート手動編集機能](../flowchart-manual-editing/) - フローチャートメモ・手動エッジ

### 詳細分析・外部AI評価
- [archive/session/03_ID-policy/](../../archive/session/03_ID-policy/) - WBS番号再利用の詳細分析（72KB、10ファイル）
  - 00_PROMPT_TO_AI.md - 外部AI評価用プロンプト
  - 01-06 - 現在の実装、問題定義、選択肢分析、業界標準、データ構造
  - feedback/ - Claude Opus, GPT, Gemini による評価レポート（3ファイル）

### 陳腐化した旧仕様
- ~~[cross-html-sync/02-data-structure.md](../cross-html-sync/02-data-structure.md)~~ → 本ディレクトリの `01-data-structure.md` を参照

---

## 🚦 実装ステータス

| 機能 | ステータス | バージョン | 備考 |
|------|-----------|-----------|------|
| **タスクデータ構造** | ✅ 実装済み | v1.0.0~ | JSON形式、LocalStorage保存 |
| **WBS番号自動生成** | ⚠️ **バグあり** | v2.13.4 | Minor→Major番号抽出バグ（修正必要） |
| **論理削除** | ✅ 実装済み | v2.13.0 | `deleted`, `deleted_at`フィールド |
| **削除後ID再利用** | ⚠️ **未修正** | v2.13.4 | Option 1実装待ち（1行変更） |
| **復元機能** | 📋 未実装 | - | Phase 2計画 |
| **ステータス管理拡張** | 📋 未実装 | - | Phase 2計画（cancelled/archived） |
| **内部ID分離** | 📋 未実装 | - | Phase 3計画（task_id導入） |

---

## 🎯 次のアクション

### 🔴 **最優先: WBS番号バグ修正 + Option 1実装**

1. **Major/Minor番号抽出バグの修正** (task-manager.html:3338)
   - 詳細: [02-wbs-numbering-system.md](./02-wbs-numbering-system.md)
   - 工数: 15-20分

2. **Option 1実装（削除済みID除外 + 全履歴考慮）**
   - 詳細: [03-deletion-policy.md](./03-deletion-policy.md)
   - 工数: 5分（1行変更）

**合計工数**: 20-25分

---

### 🟡 **Phase 2: 削除機能の拡張**（将来）

- 削除済みタスク表示トグル
- タスク復元機能
- ステータス管理（cancelled/archived）

---

### 🟢 **Phase 3: 内部ID分離**（将来）

- `task_id`（内部ID）と`wbs_no`（表示ID）の分離
- predecessorsを`task_id`参照に移行
- WBS番号の柔軟な変更を可能に

詳細: [archive/session/03_ID-policy/feedback/02_gpt_WBS番号再利用ポリシー 再評価と補完提言.md](../../archive/session/03_ID-policy/feedback/02_gpt_WBS番号再利用ポリシー%20再評価と補完提言.md)

---

## 📊 外部AI評価結果サマリー

### 評価したAI
- **Claude Opus 4.5** - 総合評価レポート（21.6KB）
- **GPT-4** - 再評価と補完提言（14.4KB）
- **Gemini** - 批判的レビュー（14.4KB）

### 全AI一致の推奨事項
1. ✅ **短期**: Option 1（削除済みID除外 + 全履歴考慮）を実装
2. ⚠️ **必須**: Major/Minor番号抽出バグを先に修正
3. 🟡 **中期**: Option 3（内部ID分離）を検討
4. 🟢 **代替**: ステータス管理（cancelled/archived）を検討

詳細: [archive/session/03_ID-policy/feedback/](../../archive/session/03_ID-policy/feedback/)

---

## 📝 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | task-management/ディレクトリ作成、仕様書3ファイル追加 |

---

**最終更新**: 2026-03-22
**管理者**: Claude Code Session
