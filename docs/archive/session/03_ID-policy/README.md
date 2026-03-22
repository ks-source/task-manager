# WBS番号再利用ポリシー - 詳細分析・外部AI評価

---

**セッション日**: 2026-03-22
**目的**: WBS番号再利用ポリシーの評価・選択
**外部AI評価**: Claude Opus 4.5、GPT-4、Gemini
**総ドキュメントサイズ**: 約72KB（10ファイル）

---

## 📖 概要

タスク削除後のWBS番号再利用について、2つの選択肢を詳細分析し、3つの外部AI（Claude Opus、GPT、Gemini）による評価を実施しました。結果、**全AI一致でOption 1（削除済みID除外 + 全履歴考慮）を推奨**という結論に至りました。

---

## 📂 ドキュメント構成

### プロンプトファイル（1ファイル）

#### [00_PROMPT_TO_AI.md](./00_PROMPT_TO_AI.md)
外部AI評価用のメインプロンプト。

**内容**:
- 評価依頼内容
- 評価観点（実装コスト、データ整合性、業界標準準拠、将来拡張性）
- 必要な出力形式

---

### 参考情報ファイル（6ファイル）

#### [01_current-implementation.md](./01_current-implementation.md)
現在の実装状況を詳細記述。

**内容**:
- `generateWBSNumber()` 関数の実装（行番号付き）
- `deleteTask()` 関数の実装
- 現在の問題（削除済みID再利用）の具体例

---

#### [02_problem-statement.md](./02_problem-statement.md)
問題定義と要求事項。

**内容**:
- 問題の発見経緯
- 2つの選択肢の概要
- 評価基準（実装コスト、データ整合性、業界標準準拠等）
- 制約条件

---

#### [03_option-1-exclude-only.md](./03_option-1-exclude-only.md)
**Option 1（削除済みID除外 + 全履歴考慮）** の詳細仕様。

**内容**:
- 基本方針: WBS番号不変、論理削除
- 実装詳細（1行変更）
- メリット・デメリット
- predecessors参照への影響
- テストシナリオ
- 実装工数見積もり（5分）

---

#### [04_option-2-suffix-approach.md](./04_option-2-suffix-approach.md)
**Option 2（ID変更 + `_del`サフィックス）** の詳細仕様。

**内容**:
- 基本方針: 削除時にWBS番号を変更（例: `TEST.15.1` → `TEST.15.1_del`）
- 実装詳細（複雑な参照更新ロジック）
- メリット・デメリット
- predecessors参照の更新処理
- リスク分析（更新漏れ、パフォーマンス）
- 実装工数見積もり（4-6時間）

---

#### [05_industry-standards.md](./05_industry-standards.md)
業界標準の比較分析。

**内容**:
- 主要ツールの比較表（GitHub, Jira, Asana, Trello, Monday.com, ClickUp, Notion）
- ID再利用の統計（86%のツールが再利用なし）
- 各ツールの詳細分析
- PMBOK（プロジェクトマネジメント標準）との関係
- ベストプラクティス

**結論**: ID不変性が業界標準（Trelloのみ例外）

---

#### [06_data-structure-context.md](./06_data-structure-context.md)
データ構造とコンテキスト。

**内容**:
- プロジェクトデータの全体構造（JSON）
- タスクオブジェクトの詳細（全フィールド定義）
- predecessorsフィールドへの影響（Option 1 vs Option 2）
- flowchartフィールドの参照（WBS番号変更の影響なし）
- スケーラビリティへの影響（タスク数増加時のパフォーマンス）
- JSONエクスポート・インポートの処理

**結論**: Option 1がデータ整合性の観点から圧倒的に優れている

---

## 🤖 外部AI評価レポート（3ファイル）

### [feedback/01_opus_WBS番号再利用ポリシー 総合評価レポート.md](./feedback/01_opus_WBS番号再利用ポリシー%20総合評価レポート.md)

**評価者**: Claude Opus 4.5
**サイズ**: 21.6KB
**評価日**: 2026-03-22

**主な内容**:
- **推奨**: Option 1を強く推奨
- **リスク評価**: Option 1: 2/10、Option 2: 7/10
- **実装時間**: Option 1: 15-20分、Option 2: 6-10時間
- **重大な発見**: ⚠️ **Major/Minor番号抽出バグを発見**（task-manager.html:3338）
  - 現在のコードは Minor番号をMajor番号として使用している
  - 例: `TEST.2.5` → 生成される番号は `TEST.6.1`（本来は `TEST.3.1`）
  - Option 1実装前にこのバグ修正が必須
- **詳細な比較表**: 15項目の観点から両選択肢を評価
- **段階的実装計画**: Phase 1（緊急）、Phase 2（中期）、Phase 3（長期）

**結論**: Option 1を実装し、Major/Minor番号バグを先に修正すべき

---

### [feedback/02_gpt_WBS番号再利用ポリシー 再評価と補完提言.md](./feedback/02_gpt_WBS番号再利用ポリシー%20再評価と補完提言.md)

**評価者**: GPT-4
**サイズ**: 14.4KB
**評価日**: 2026-03-22

**主な内容**:
- **推奨**: 短期: Option 1、中期: Option 3（内部ID分離）
- **重要な洞察**: 本当の問題は「WBS番号が参照キー」であること
- **Option 3の提案**: `task_id`（内部ID）と`wbs_no`（表示ID）を分離
  ```json
  {
    "task_id": "uuid-1234-5678",
    "wbs_no": "TEST.15.1",
    "predecessors": ["uuid-1234-5677"]
  }
  ```
- **リスク修正**: Option 1のリスクは3-4/10（Opusの2/10より現実的）
  - 理由: WBS番号生成ロジックの曖昧性（Major/Minorバグ等）
- **段階的移行計画**: Option 1 → Option 3への移行ロードマップ

**結論**: Option 1を短期実装し、Option 3を中長期検討すべき

---

### [feedback/03_gemini_WBS_Reuse_Policy_Critical_Review.md.md](./feedback/03_gemini_WBS_Reuse_Policy_Critical_Review.md.md)

**評価者**: Gemini
**サイズ**: 14.4KB
**評価日**: 2026-03-22

**主な内容**:
- **推奨**: 短期: Option 1、中期: Option 3（内部ID分離）
- **追加の視点**: 「削除」vs「キャンセル/アーカイブ」の区別
  - 削除: 本当に間違って作成したタスク
  - キャンセル: 計画変更で不要になったタスク
  - アーカイブ: 完了後の整理
- **ステータス管理の提案**:
  ```json
  {
    "status": "キャンセル" | "アーカイブ",
    "archived": true,
    "cancelled": true
  }
  ```
- **WBS番号仕様の形式化**: 曖昧性を排除するための仕様明確化が必要
- **リスク評価**: Option 1: 3-4/10（GPTと同様）

**結論**: Option 1を実装し、ステータス管理を拡張すべき

---

## 📊 評価結果サマリー

### 全AI一致の推奨事項

| 評価項目 | Option 1 | Option 2 | 全AI推奨 |
|---------|---------|---------|---------|
| **実装コスト** | ✅ 最小（5分） | ❌ 高（4-6h） | ✅ Option 1 |
| **predecessors参照** | ✅ 壊れない | ⚠️ 更新ロジック必須 | ✅ Option 1 |
| **業界標準準拠** | ✅ GitHub/Jira | ❌ 非標準 | ✅ Option 1 |
| **リスク評価** | ✅ 低（2-4/10） | ⚠️ 高（7/10） | ✅ Option 1 |
| **データ整合性** | ✅ 維持 | ⚠️ リスクあり | ✅ Option 1 |

---

### 全AI共通の指摘事項

1. ⚠️ **Major/Minor番号抽出バグが存在**（Opus発見）
   - Option 1実装前に修正必須
   - 工数: 15-20分

2. 🟡 **中長期的にはOption 3（内部ID分離）を検討**（GPT、Gemini提案）
   - `task_id`（内部ID）と`wbs_no`（表示ID）の分離
   - predecessorsを`task_id`参照に移行
   - WBS番号の柔軟な変更を可能に

3. 🟢 **ステータス管理の拡張を検討**（Gemini提案）
   - 削除 vs キャンセル/アーカイブの区別
   - より柔軟なタスク管理

---

## 🎯 最終結論

### 短期実装（Phase 1 - 緊急）

**工数**: 20-25分

1. **Major/Minor番号抽出バグの修正**（15-20分）
   - ファイル: task-manager.html:3338
   - 修正内容: `parts[parts.length - 1]` → `parts[1]`（Major番号を抽出）

2. **Option 1実装**（5分）
   - ファイル: task-manager.html:3330
   - 修正内容: `&& !t.deleted` を削除

---

### 中長期計画（Phase 2-3）

#### Phase 2（中期 - 数ヶ月）
- 削除済みタスク表示トグル
- タスク復元機能
- ステータス管理拡張（cancelled/archived）

#### Phase 3（長期 - 半年～1年）
- 内部ID分離（Option 3）
  - `task_id`導入
  - predecessors移行
  - データマイグレーション

---

## 🔗 正式仕様書へのリンク

本アーカイブの分析結果を基に、以下の正式仕様書を作成しました：

- [../../specs/task-management/README.md](../../specs/task-management/README.md) - タスク管理コア仕様書（全体概要）
- [../../specs/task-management/01-data-structure.md](../../specs/task-management/01-data-structure.md) - タスクデータ構造（完全版）
- [../../specs/task-management/02-wbs-numbering-system.md](../../specs/task-management/02-wbs-numbering-system.md) - WBS番号体系、バグ詳細
- [../../specs/task-management/03-deletion-policy.md](../../specs/task-management/03-deletion-policy.md) - **削除ポリシー（Option 1実装仕様）**

---

## 📅 セッション履歴

| 日付 | 作業内容 |
|------|---------|
| 2026-03-22 | プロンプトファイル作成、参考情報ファイル6個作成 |
| 2026-03-22 | 外部AI評価実施（Opus、GPT、Gemini） |
| 2026-03-22 | 評価結果分析、正式仕様書作成 |
| 2026-03-22 | 本README.md作成 |

---

**最終更新**: 2026-03-22
**次のアクション**: [../../specs/task-management/README.md](../../specs/task-management/README.md) の「次のアクション」セクションを参照
