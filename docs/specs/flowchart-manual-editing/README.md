# フローチャート手動編集機能 - 仕様書

---
**feature**: フローチャート手動編集
**version**: 1.0
**status**: draft
**created**: 2026-03-21
**updated**: 2026-03-21
**author**: Claude Code Session
---

## 概要

Mermaid生成SVGフローチャートに対して、会議中のメモ書きレベルで手動エッジ・メモノードを追加できる機能の仕様書。

## 背景と目的

### 背景
- 会議中にフローチャートを柔軟に修正したい
- Mermaid記法の制約を受けずに自由な接続を追加したい
- 追加した情報を構造化データとして出力したい

### 目的
プロフェッショナルな編集ツール（draw.io等）に匹敵する機能ではなく、**会議中のメモ書き程度の柔軟な編集機能**を実装する。

## ドキュメント構成

### 仕様書（Specifications）
1. [**全体設計・方針**](./01-overview.md) - 設計方針、実装範囲、技術制約、工数見積もり
2. [**手動エッジ追加仕様**](./02-manual-edge.md) - UI操作フロー、SVG構造、実装関数
3. [**ノード吸着仕様**](./03-node-snap.md) - 自動吸着ロジック、ビジュアルフィードバック（Phase 2で作成）
4. [**接続グラフ出力仕様**](./04-connection-export.md) - JSON/CSV出力形式、活用例（Phase 2で作成）
5. [**データ構造定義**](./05-data-structure.md) - JavaScript内部構造、JSON保存形式
6. [**実装計画・工数**](./06-implementation-plan.md) - フェーズ分け、振り返り（Phase 3で作成）

### 実装ガイド（Implementation）
- [Phase 1: 基本エッジ実装](../../implementation/flowchart-manual-editing/phase-1-edge-basic.md)
- [Phase 2: ノード吸着](../../implementation/flowchart-manual-editing/phase-2-node-snap.md)
- [Phase 3: エクスポート機能](../../implementation/flowchart-manual-editing/phase-3-export.md)
- [トラブルシューティング](../../implementation/flowchart-manual-editing/troubleshooting.md)

## 関連ドキュメント

- [元のフローチャート仕様](../../flowchart-spec.md) - Mermaid連携の基本仕様
- [タスク・フローチャート手動リンク仕様](../../task-edit-ui-flowchart-link.md) - タスク管理連携
- [UIモックアップ](../../ui/flowchart-mockup-interactive.html) - 現在の実装ファイル

## 主要機能

### ✅ 実装する機能
- **手動エッジ追加**: 2点クリックで直線エッジを追加
- **ノード自動吸着**: クリック位置がノード近く（半径50px）なら自動接続
- **エッジラベル**: エッジに任意のテキストラベルを付与
- **接続グラフ出力**: ノード間の接続関係をJSON/CSV形式で出力
- **視覚的フィードバック**: 追加モード中のプレビュー表示

### ❌ 実装しない機能
- 複雑な曲線エッジ（Bézier制御点計算）
- 自動レイアウト（Dagreアルゴリズム等）
- ノード移動時のエッジ追従
- Mermaid記法への逆変換

## 開発工数

| フェーズ | 機能 | 工数 |
|---------|------|------|
| Phase 1 | 基本エッジ追加機能 | 13時間 |
| Phase 2 | ノード吸着機能 | 2時間 |
| Phase 3 | 接続グラフ出力 | 12時間 |
| **合計** | | **27時間（3-4日間）** |

## 技術スタック

- **言語**: JavaScript（ES6+）
- **外部ライブラリ**: なし（ブラウザ標準APIのみ）
- **対象ブラウザ**: Edge/Chrome 86+
- **主要API**:
  - SVG DOM API
  - File System Access API
  - Canvas 2D Context（座標変換用）

## 開発フェーズ

### Phase 1: 基本エッジ追加（13時間）
- ツールバーにエッジ追加モードボタン実装
- 2点クリックで直線エッジ描画
- エッジラベル入力ダイアログ
- JSON保存・読込対応

### Phase 2: ノード吸着（2時間）
- クリック位置からノード検索
- 吸着時のビジュアルフィードバック
- ノード中心座標への自動調整

### Phase 3: 接続グラフ出力（12時間）
- 接続関係抽出ロジック
- JSON形式出力
- CSV形式出力（複数ファイル）
- エクスポートUI実装

## セッション継続のための重要情報

### このディレクトリ構造の目的
新しいClaude Codeセッションでも、以下のファイルを読むだけで完全に文脈を復元できる：

1. **このREADME.md**: 全体像と目次
2. **01-overview.md**: 設計方針と制約
3. **02-manual-edge.md**: エッジ実装詳細
4. **05-data-structure.md**: データ構造定義

### セッション開始時の確認事項
1. 現在の実装ファイル: `/mnt/c/dev/task-manager/docs/ui/flowchart-mockup-interactive.html`
2. 既存機能: ノード選択、メモ編集、ラベル編集、ミニマップ、SVGエクスポート
3. 次に実装する機能: **手動エッジ追加（Phase 1）**

## 変更履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-21 | 1.0 | 初版作成 - 仕様書構成確定 |

---

**最終更新**: 2026-03-21
**次のアクション**: [01-overview.md](./01-overview.md) の詳細設計確認
