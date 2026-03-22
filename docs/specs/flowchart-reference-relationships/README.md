# フローチャート参照関係管理機能 - 仕様書

---
**feature**: フローチャート参照関係管理
**version**: 2.0
**status**: approved
**created**: 2026-03-22
**updated**: 2026-03-22
**author**: Claude Code Session
---

## 概要

フローチャート要素（正規ノード、サブグラフ、メモノード）間の関連性を構造化データとして記録・管理する機能の仕様書。

v2.0 メタデータスキーマの中核機能であり、以下の目的で設計されている：
1. **AI解析の最適化**: 視覚的な矢印だけでなく、意味的な関連性をメタデータとして提供
2. **永続化**: SVGファイルへの保存・復元により、情報の損失を防ぐ
3. **明示的な管理**: 自動推測ではなく、人間が明示的に設定する設計

---

## ドキュメント構成

### 仕様書（Specifications）

1. [**設計原則**](./01-design-principles.md) - 設計コンセプト、データフロー、利点
2. [**データ構造定義**](./02-data-structure.md) - v2.0メタデータスキーマ完全仕様
3. [**実装計画**](./03-implementation-plan.md) - 実装フェーズ、工数見積もり
4. [**クリックto Add UX**](./04-click-to-add-ux.md) - クリックで参照追加する UX 仕様
5. [**統合ロードマップ**](./05-integrated-roadmap.md) - 全体の実装ロードマップ

### セッション履歴（Archive）

元のセッションドキュメント（外部AI有識者レビュー含む）は以下に保管：
- `/docs/archive/session/04_node-reference/`

---

## 主要機能

### ✅ Phase 2 機能

- **参照関係管理**: `references` フィールドによるノード間の関連性記録
- **全要素対応**: 正規ノード、サブグラフ、メモノードすべてに適用可能
- **一方向参照**: A → B を設定しても、B → A は自動設定しない
- **人間による入力**: UI経由で明示的に設定
- **SVGメタデータ化**: 保存・復元により永続化
- **クリックto Add UX**: フローチャート上でノードをクリックして直接参照追加

### ✅ Phase 4 統合（ステータス色分け機能）

参照関係管理機能は、Phase 4 のステータス色分け機能と並行して v2.0 メタデータに統合される。

```javascript
// v2.0 メタデータの全体像
{
  "version": "2.0",
  "schemaVersion": "2.0",

  // 既存機能（v1.0から継続）
  "elements": { ... },
  "manualNodes": { ... },
  "manualEdges": { ... },

  // v2.0 新機能
  "elementStatuses": { ... },      // Phase 4: ステータス色分け
  "references": { ... },            // Phase 2: 参照関係管理
  "taskFlowchartLinks": { ... }     // Phase 2: タスク連携スナップショット
}
```

---

## データ構造

### references フィールド

```javascript
// グローバル変数
let elementReferences = {
  'flowchart-A': ['flowchart-B', 'memo_node_1'],  // 正規ノード → 正規ノード + メモノード
  'flowchart-B': ['flowchart-C'],                 // 正規ノード → 正規ノード
  'memo_node_1': ['flowchart-A'],                 // メモノード → 正規ノード
  'subgraph_1': ['flowchart-D']                   // サブグラフ → 正規ノード
};
```

**重要な設計原則**:
- **汎用的な関連性**: "関連", "補足", "前提" などの意味は将来拡張可能
- **ID安定性問題への対応**: Mermaid再生成でIDが変わる可能性を考慮
- **タスク連携との分離**: タスクは別ドメインのエンティティとして独立管理

---

## 設計の背景

### ❌ 廃止された設計: connectedEdges

Phase 1-2 の初期設計では `connectedEdges` フィールドが検討されていたが、以下の理由で廃止：

```javascript
// ❌ 陳腐化した設計（採用されず）
manualNodes["MEMO_001"] = {
  metadata: {
    connectedEdges: ["edge-1", "edge-2"]  // エッジIDを追跡
  }
};
```

**廃止理由**:
1. **意味的に不明確**: 「接続エッジ」ではなく「関連ノード」を記録すべき
2. **Phase 2 との重複**: `references` と機能が重複
3. **UX の問題**: ユーザーはエッジではなくノードを選択したい

### ✅ 採用された設計: references

```javascript
// ✅ 正式採用された設計
manualNodes["MEMO_001"] = {
  references: ["U_START", "node_002"]  // 関連ノードIDを追跡
};
```

**採用理由**:
1. **意味的に明確**: 「このメモノードはこの正規ノードの補足である」
2. **Phase 2 との統合**: 既存設計との完全互換
3. **優れた UX**: クリックto Add パターンで直感的に操作可能

---

## 外部レビュー

本仕様は以下の外部AI有識者によるレビューを受け、承認されている：

- **Claude Opus** - 設計原則、データフロー、正本管理の評価
- **GPT-4** - IDの安定性問題、参照の意味論への提言
- **Gemini** - 削除・改名・複製の意味論、セキュリティ対策の評価

レビューフィードバックは `/docs/archive/session/04_node-reference/feedback/` に保管。

---

## 関連ドキュメント

- [フローチャート手動編集機能](../flowchart-manual-editing/README.md) - 手動エッジ・メモノードの基本仕様
- [タスク管理仕様](../task-management/README.md) - タスクマネージャーとの連携
- [データ統合仕様](../data-integration/README.md) - task-manager.html との同期

---

## 開発工数

| フェーズ | 機能 | 工数 |
|---------|------|------|
| Phase 2-1 | 基本参照管理UI（dialog方式） | 4-6時間 |
| Phase 2-2 | クリックto Add UX実装 | 2-3時間 |
| Phase 2-3 | 参照矢印の視覚化 | 3-4時間 |
| Phase 2-4 | データ永続化・復元 | 2-3時間 |
| **合計（MVP）** | | **11-16時間（1.5-2日間）** |

---

## 技術スタック

- **言語**: JavaScript（ES6+）
- **外部ライブラリ**: なし（ブラウザ標準APIのみ）
- **対象ブラウザ**: Edge/Chrome 86+
- **主要API**:
  - SVG DOM API
  - File System Access API
  - LocalStorage API（タスク連携用）

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-22 | 1.0 | 初版作成 - セッションアーカイブから正式仕様に昇格 |
| 2026-03-22 | 2.0 | クリックto Add UX追加、connectedEdges 廃止を明記 |

---

## セッション継続のための重要情報

### このディレクトリ構造の目的

新しいClaude Codeセッションでも、以下のファイルを読むだけで完全に文脈を復元できる：

1. **このREADME.md**: 全体像と目次
2. **01-design-principles.md**: 設計原則、データフロー、利点
3. **02-data-structure.md**: v2.0メタデータスキーマ完全仕様
4. **04-click-to-add-ux.md**: クリックto Add UXの詳細仕様

### セッション開始時の確認事項

1. 現在の実装ファイル: `/mnt/c/dev/task-manager/flowchart-editor.html`
2. 既存機能: v2.0メタデータ（ステータス色分け、参照関係管理）
3. **重要**: `connectedEdges` は廃止された設計（実装しない）
4. **次に実装する機能**: クリックto Add UX（Phase 2-2）

---

**最終更新**: 2026-03-22
**ステータス**: 正式仕様として承認済み
**次のアクション**: Phase 2-2（クリックto Add UX）の実装開始
