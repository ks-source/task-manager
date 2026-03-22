# セッションアーカイブ: フローチャート参照関係管理機能

---
**セッション**: Phase 2 - 参照関係管理機能の設計
**日付**: 2026-03-22
**ステータス**: ✅ 正式仕様に昇格済み
---

## 🎉 本設計は正式仕様として承認されました

本ディレクトリの設計内容は、外部AI有識者レビュー（Opus, GPT-4, Gemini）を経て、**正式仕様として採用**されました。

**正式仕様の場所**:
- `/docs/specs/flowchart-reference-relationships/`

---

## 移行されたファイル

以下のファイルは正式仕様ディレクトリに移動されました：

| 旧ファイル名（archive） | 新ファイル名（specs） | 内容 |
|----------------------|-------------------|------|
| `02_proposed-design.md` | `01-design-principles.md` | 設計原則、データフロー、利点 |
| `04_data-structure.md` | `02-data-structure.md` | v2.0メタデータスキーマ完全仕様 |
| `05_implementation-plan.md` | `03-implementation-plan.md` | 実装計画、工数見積もり |
| `07_integrated-roadmap.md` | `05-integrated-roadmap.md` | 統合実装ロードマップ |

**新規作成された仕様書**:
- `README.md` - 概要・目次
- `04-click-to-add-ux.md` - クリックで参照追加する UX 仕様

---

## このディレクトリに残っているファイル

以下のファイルは**セッション履歴**として本ディレクトリに残されています：

### セッション記録
- `00_PROMPT_to_external_AI.md` - 外部AIへのプロンプト
- `01_current-implementation.md` - 当時の実装状況
- `03_ui-mockup.md` - UIモックアップ（初期案）
- `06_technical-concerns.md` - 技術的懸念事項

### 外部レビュー
- `feedback/01_opus.md` - Claude Opus によるレビュー
- `feedback/02_gpt_フローチャート参照関係設計に対する補完的・批判的レビュー.md` - GPT-4 によるレビュー
- `feedback/03_gemini_フローチャート参照関係設計に対する補完的・批判的レビュー（統合版）.md` - Gemini によるレビュー

**これらのファイルは歴史的価値があるため保存されています。**

---

## 主要な設計決定

### ✅ 採用された設計

**Phase 2: `references` フィールド**
```javascript
elementReferences = {
  'flowchart-A': ['flowchart-B', 'memo_node_1'],  // ノードIDの配列
  'memo_node_1': ['flowchart-A']
};
```

**利点**:
- 意味的に明確（「関連ノード」を記録）
- クリックto Add UX で直感的に操作可能
- v2.0 メタデータに完全統合

### ❌ 廃止された設計

**初期案: `connectedEdges` フィールド**
```javascript
// ❌ 採用されなかった設計
manualNodes["MEMO_001"] = {
  metadata: {
    connectedEdges: ["edge-1", "edge-2"]  // エッジIDの配列
  }
};
```

**廃止理由**:
1. 意味的に不明確（エッジではなくノードを記録すべき）
2. Phase 2 の `references` と機能重複
3. UX が煩雑（dialog/select ベースの操作）

---

## 外部レビューのハイライト

### Claude Opus の評価
- **設計原則の妥当性**: ✅ 高く評価
- **データフローの明確さ**: ✅ 正本管理の明確化を評価
- **実装可能性**: ✅ 現実的なアプローチ

### GPT-4 の提言
- **IDの安定性問題**: ⚠️ Mermaid再生成でIDが変わるリスクを指摘
- **対策案**: ラベルベースの再解決候補をユーザーに提示
- **Phase 3拡張**: `targetHint` フィールドの導入を提案

### Gemini の評価
- **削除・改名・複製の意味論**: ✅ 明確に定義されていると評価
- **セキュリティ対策**: ✅ CSS.escape(), Prototype Pollution 対策を評価
- **テストケース**: ✅ 循環参照検出、バリデーション関数を評価

---

## 実装状況

| フェーズ | 機能 | ステータス |
|---------|------|-----------|
| Phase 2-1 | 基本参照管理UI | ⏳ 未実装（仕様確定済み） |
| Phase 2-2 | クリックto Add UX | ⏳ 未実装（仕様確定済み） |
| Phase 2-3 | 参照矢印の視覚化 | ⏳ 未実装 |
| Phase 2-4 | データ永続化・復元 | ⏳ 未実装 |

**次のアクション**: Phase 2-2（クリックto Add UX）の実装開始

---

## 関連リンク

- **正式仕様**: `/docs/specs/flowchart-reference-relationships/`
- **実装ファイル**: `/mnt/c/dev/task-manager/flowchart-editor.html`
- **関連仕様**: `/docs/specs/flowchart-manual-editing/` - 手動エッジ・メモノード

---

## 変更履歴

| 日付 | イベント |
|------|---------|
| 2026-03-22 | セッション開始 - Phase 2 参照関係管理機能の設計 |
| 2026-03-22 | 外部AI有識者レビュー実施（Opus, GPT-4, Gemini） |
| 2026-03-22 | 正式仕様として承認・移行完了 |

---

**最終更新**: 2026-03-22
**ステータス**: アーカイブ完了（正式仕様として `/docs/specs/flowchart-reference-relationships/` に移行）
