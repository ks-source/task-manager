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

Mermaid生成SVGフローチャートに対して、会議中のメモ書きレベルで手動エッジ・メモノードを追加し、AI解析用の統合JSONデータとして出力できる機能の仕様書。

## 背景と目的

### 背景
- 会議中にフローチャートを柔軟に修正したい
- Mermaid記法の制約を受けずに自由な接続を追加したい
- 追加した情報を構造化データとして出力したい
- **外部AIに包括的に解析してもらいたい（フロー最適化、ドキュメント生成等）**

### 目的
プロフェッショナルな編集ツール（draw.io等）に匹敵する機能ではなく、**会議中のメモ書き程度の柔軟な編集機能**と**AI解析最適化されたデータ出力**を実装する。

## ドキュメント構成

### 仕様書（Specifications）
1. [**全体設計・方針**](./01-overview.md) - 設計方針、実装範囲、技術制約、工数見積もり
2. [**手動エッジ追加仕様**](./02-manual-edge.md) - UI操作フロー、SVG構造、実装関数
3. ~~**ノード吸着仕様**~~ - スコープ外（実装難航のため除外）
4. [**AI解析用統合JSON出力**](./04-ai-integrated-export.md) - 統合データ構造、ユースケース、実装方針
5. [**データ構造定義**](./05-data-structure.md) - JavaScript内部構造、JSON保存形式
6. [**実装計画・工数**](./06-implementation-plan.md) - フェーズ分け、独立要素アーキテクチャ、工数見積もり

### 実装ガイド（Implementation）
- [Phase 1: 基本エッジ実装](../../implementation/flowchart-manual-editing/phase-1-edge-basic.md)
- [Phase 2: ノード吸着](../../implementation/flowchart-manual-editing/phase-2-node-snap.md)
- [Phase 3: エクスポート機能](../../implementation/flowchart-manual-editing/phase-3-export.md)
- [トラブルシューティング](../../implementation/flowchart-manual-editing/troubleshooting.md)

## 関連ドキュメント

- [**フローチャート参照関係管理**](../flowchart-reference-relationships/README.md) - Phase 2: 要素間の関連性記録（v2.0）
- [元のフローチャート仕様](../../flowchart-spec.md) - Mermaid連携の基本仕様
- [タスク・フローチャート手動リンク仕様](../../task-edit-ui-flowchart-link.md) - タスク管理連携
- [UIモックアップ](../../ui/flowchart-mockup-interactive.html) - 現在の実装ファイル

## 主要機能

### ✅ 実装する機能
- **メモノード追加**: 任意の位置に付箋風のメモノードを配置
- **ノードドラッグ移動**: メモノードをマウスドラッグで自由に移動
- **手動エッジ追加**: 2点クリックで直線エッジを追加（自由配置モード）
- **エッジ端点編集**: エッジをクリックして端点ハンドルで座標調整
- **エッジラベル**: エッジに任意のテキストラベルを付与
- **AI解析用統合JSON出力**: Mermaid要素+手動要素の座標・メタデータを統合出力
- **視覚的フィードバック**: 追加モード中のプレビュー表示

### ❌ 実装しない機能（明示的除外）
- ノード自動吸着（実装難航のため除外）
- ノード移動時のエッジ自動追従（独立要素モデルのため）
- システム上の依存関係自動管理（メタデータで手動管理）
- 複雑な曲線エッジ（Bézier制御点計算）
- 自動レイアウト（Dagreアルゴリズム等）
- Mermaid記法への逆変換

## 開発工数

| フェーズ | 機能 | 工数 |
|---------|------|------|
| Phase 1 | メモノード作成・ドラッグ移動 | 4-5時間 |
| Phase 2 | 手動エッジ端点編集 | 8-10時間 |
| Phase 3-AI MVP | AI解析用統合JSON出力 | 12-14時間 |
| **合計（MVP）** | | **24-29時間（3-4日間）** |
| Phase 3-AI 拡張 | 詳細情報追加（オプション） | +6-8時間 |
| **Phase 4** | **ステータス色分け機能** | **6-9時間** |
| **合計（完全版）** | | **30-38時間（4-5日間）** |

## 技術スタック

- **言語**: JavaScript（ES6+）
- **外部ライブラリ**: なし（ブラウザ標準APIのみ）
- **対象ブラウザ**: Edge/Chrome 86+
- **主要API**:
  - SVG DOM API
  - File System Access API
  - Canvas 2D Context（座標変換用）

## 開発フェーズ

### Phase 1: メモノード作成・ドラッグ移動（4-5時間）
- `manualNodes` データ構造追加
- ノード作成機能（`createManualNode`）
- ドラッグ移動機能（`attachDragBehavior`）
- データ永続化（`buildMemoData`に追加）
- UI追加（📝 メモノードボタン）

### Phase 2: 手動エッジ端点編集（8-10時間）
- エッジクリックで編集モード移行
- 端点ハンドル表示・ドラッグ機能
- エッジライン/ラベルのリアルタイム更新
- エッジ削除機能
- UI改善（ホバー、キーボード操作等）

### Phase 3-AI MVP: AI解析用統合JSON出力（12-14時間）
- Mermaidノード座標抽出（`extractMermaidNodes`）
- Mermaidエッジ基本情報抽出（`extractMermaidEdges`）
- Mermaidクラスタ抽出（`extractMermaidClusters`）
- 統合データ生成（`extractIntegratedData`）
- エクスポート機能（`exportIntegratedJson`）
- UI統合（出力ドロップダウンメニュー）

### Phase 3-AI 拡張: 詳細情報追加（+6-8時間、オプション）
- Mermaidエッジpath詳細抽出
- シェイプタイプ自動判別
- スタイル情報詳細化

### Phase 4: ステータス色分け機能（6-9時間）
- ステータス色定義（task-manager.htmlと同一の7種類）
- 右クリックメニューUI実装
- 正規ノード・サブグラフ・メモノードへのステータス適用
- データ永続化（`elementStatuses`フィールド、v2.0メタデータ）
- UX改善（ステータスバッジ、フィルタリング機能）

**重要**: フローチャートステータスとタスクステータスは完全に独立（データ連携なし）

## セッション継続のための重要情報

### このディレクトリ構造の目的
新しいClaude Codeセッションでも、以下のファイルを読むだけで完全に文脈を復元できる：

1. **このREADME.md**: 全体像と目次
2. **06-implementation-plan.md**: 実装計画、独立要素アーキテクチャ、工数見積もり
3. **04-ai-integrated-export.md**: AI解析用統合JSON出力仕様
4. **05-data-structure.md**: データ構造定義
5. **02-manual-edge.md**: エッジ実装詳細（既存機能）

### セッション開始時の確認事項
1. 現在の実装ファイル: `/mnt/c/dev/task-manager/docs/ui/flowchart-mockup-interactive.html`
2. 既存機能: ノード選択、メモ編集、ラベル編集、手動エッジ追加（自由配置）、ミニマップ、SVGエクスポート
3. 次に実装する機能: **メモノード作成・ドラッグ移動（Phase 1）**
4. アーキテクチャ原則: **独立要素モデル（エッジ追従なし、手動メタデータ管理）**

## 変更履歴

| 日付 | バージョン | 変更内容 |
|------|-----------|---------|
| 2026-03-21 | 1.0 | 初版作成 - 仕様書構成確定 |
| 2026-03-21 | 2.0 | メモノード・AI統合JSON機能追加、ノード吸着除外、独立要素アーキテクチャ採用 |
| 2026-03-22 | 3.0 | Phase 4（ステータス色分け機能）仕様追加 |

---

**最終更新**: 2026-03-22
**次のアクション**:
- 別セッションのスキーマ修正完了を待機
- Phase 4実装開始 - [06-implementation-plan.md](./06-implementation-plan.md#phase-4-ステータス色分け機能6-9h) を参照
