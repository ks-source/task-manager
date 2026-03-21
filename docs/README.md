# Task Manager - ドキュメント

Task Managerプロジェクトの全ドキュメントへのエントリーポイント。

---

## 📂 ドキュメント構成

```
docs/
├── README.md (このファイル)                     # ドキュメントエントリーポイント
│
├── specs/                                      # 機能別仕様書
│   ├── README.md                               # 仕様書ディレクトリ概要
│   ├── gantt-interactive-editing/              # ガントチャート・インタラクティブ編集
│   │   ├── README.md
│   │   ├── 01-ui-ux-design.md
│   │   ├── 02-technical-design.md
│   │   ├── 03-data-synchronization.md
│   │   └── 04-implementation-plan.md
│   ├── flowchart-manual-editing/               # フローチャート手動編集
│   │   ├── README.md
│   │   ├── 01-overview.md
│   │   ├── 02-manual-edge.md
│   │   ├── 04-ai-integrated-export.md
│   │   ├── 05-data-structure.md
│   │   └── 06-implementation-plan.md
│   └── current -> flowchart-manual-editing     # 現在開発中の機能（シンボリックリンク）
│
├── ui/                                         # UIモックアップ・アイコン
│   ├── flowchart-mockup-interactive.html       # フローチャート実装ファイル
│   └── icon/                                   # SVGアイコンカタログ
│
├── archive/                                    # アーカイブ
│   ├── prompts/                                # 過去のプロンプト
│   └── sesson/                                 # セッション記録
│       └── 01_connector-adsorption/            # ノード吸着機能の開発記録
│
├── flowchart-spec.md                           # フローチャート基本仕様
├── task-edit-ui-flowchart-link.md              # タスク・フローチャート連携仕様
│
└── (その他)
    ├── implementation/                         # 実装ガイド（将来用）
    ├── sample/                                 # サンプルデータ
    └── screenshots/                            # スクリーンショット
```

---

## 🎯 目的別ドキュメントガイド

### 1. プロジェクト全体を理解したい

1. [プロジェクトREADME](../README.md) - 機能概要、使い方
2. [CHANGELOG](../CHANGELOG.md) - v1.0.0からの変更履歴
3. [specs/README.md](./specs/README.md) - 全機能の仕様書一覧

---

### 2. 新機能の実装を開始したい

#### ガントチャート・インタラクティブ編集機能（優先度: 高）

**開始手順:**
1. [gantt-interactive-editing/README.md](./specs/gantt-interactive-editing/README.md) - 全体概要
2. [04-implementation-plan.md](./specs/gantt-interactive-editing/04-implementation-plan.md) - Phase 1から実装開始
3. [02-technical-design.md](./specs/gantt-interactive-editing/02-technical-design.md) - 実装詳細

**工数見積もり:** 15-21h（2-3日間）

---

#### フローチャート手動編集機能（優先度: 中）

**開始手順:**
1. [flowchart-manual-editing/README.md](./specs/flowchart-manual-editing/README.md) - 全体概要
2. [06-implementation-plan.md](./specs/flowchart-manual-editing/06-implementation-plan.md) - Phase 1から実装開始
3. [05-data-structure.md](./specs/flowchart-manual-editing/05-data-structure.md) - データ構造確認

**工数見積もり:** 24-29h（3-4日間、MVP版）

---

### 3. セッション継続・引き継ぎ

新しいClaude Codeセッションでプロジェクトを再開する場合：

1. **このREADME.md**を読む（全体像把握）
2. **[CHANGELOG.md](../CHANGELOG.md)**の最新版を確認（最新状態把握）
3. 開発中の機能の**README.md**を読む（文脈復元）
4. **実装計画（04または06）**を読む（次のアクション確認）

---

### 4. 過去の開発履歴を調べたい

- **[CHANGELOG.md](../CHANGELOG.md)** - v1.0.0からv2.12.0までの全変更履歴
- **[archive/sesson/](./archive/sesson/)** - 特定機能の開発セッション記録
  - 例: `01_connector-adsorption/` - ノード吸着機能の試行錯誤記録
  - 外部AI有識者（Claude Opus、GPT、Gemini）のフィードバック含む

---

## 📋 主要仕様書クイックリンク

### ガントチャート機能

| ドキュメント | 説明 |
|------------|------|
| [gantt-interactive-editing/README.md](./specs/gantt-interactive-editing/README.md) | **NEW** インタラクティブ編集機能の全体概要 |
| [gantt-interactive-editing/01-ui-ux-design.md](./specs/gantt-interactive-editing/01-ui-ux-design.md) | ASCIIアートで見る操作イメージ |
| [gantt-interactive-editing/04-implementation-plan.md](./specs/gantt-interactive-editing/04-implementation-plan.md) | Phase別実装計画 |

### フローチャート機能

| ドキュメント | 説明 |
|------------|------|
| [flowchart-spec.md](./flowchart-spec.md) | Mermaid連携の基本仕様 |
| [flowchart-manual-editing/README.md](./specs/flowchart-manual-editing/README.md) | 手動編集機能の全体概要 |
| [flowchart-manual-editing/04-ai-integrated-export.md](./specs/flowchart-manual-editing/04-ai-integrated-export.md) | AI解析用JSON出力仕様 |

### タスク管理機能

| ドキュメント | 説明 |
|------------|------|
| [task-edit-ui-flowchart-link.md](./task-edit-ui-flowchart-link.md) | タスクとフローチャートの連携仕様 |

---

## 🔧 開発プロセス

### 仕様書 → 実装の流れ

```
1. 要求分析
   ↓
2. 仕様書作成（specs/新機能名/）
   - README.md（概要）
   - 01-ui-ux-design.md
   - 02-technical-design.md
   - 03-data-*.md
   - 04-implementation-plan.md
   ↓
3. Phase 1実装開始
   ↓
4. テスト・ユーザー検証
   ↓
5. Phase 2実装
   ↓
6. リリース（CHANGELOG.md更新）
```

### ドキュメント更新ルール

1. **機能追加時**: `specs/` 以下に新ディレクトリ作成
2. **リリース時**: `CHANGELOG.md` 更新
3. **アーカイブ時**: `archive/` へ移動

---

## 📊 ドキュメント統計

| カテゴリ | ファイル数 | 備考 |
|---------|----------|------|
| 仕様書（Markdown） | 24+ | specs/, archive/ 含む |
| UIモックアップ（HTML） | 1+ | flowchart-mockup-interactive.html |
| SVGアイコンカタログ（HTML） | 7+ | ui/icon/ |

---

## 🔗 外部リンク

- [プロジェクトルート README](../README.md)
- [CHANGELOG](../CHANGELOG.md)
- [GitHub Repository](https://github.com/ks-source/task-manager)（想定）

---

## 📝 ドキュメント作成ガイドライン

### 新しい仕様書を作成する場合

1. **テンプレートに従う**: [specs/README.md](./specs/README.md) のテンプレート参照
2. **ヘッダー情報を記載**: feature, version, status, created, updated
3. **ASCIIアートを活用**: 視覚的な説明を優先
4. **クロスリファレンス**: 関連ドキュメントへのリンクを明記
5. **セッション継続性**: 新セッションでも文脈復元できる構成

### Markdownスタイル

- **見出し**: `#` は1ファイルに1つ、`##` で章立て
- **コードブロック**: 言語指定（```javascript, ```css）
- **テーブル**: 整列を意識
- **ASCIIアート**: プレーンテキストでも読みやすく

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-21 | ドキュメント構造整理、gantt-interactive-editing仕様書追加 |
| 2026-03-21 | 本README.md作成、全体ナビゲーション確立 |

---

**最終更新**: 2026-03-21
**管理者**: Claude Code Session
