# 現在の実装状況

## 📊 フローチャートエディターの概要

### プロジェクト構成
- **ファイル名**: `flowchart-editor.html`
- **タイプ**: 単一HTMLファイル（スタンドアロン）
- **サイズ**: 約250KB
- **外部依存**: なし（Vanilla JavaScript）

### 主要機能
1. **Mermaid SVGフローチャートの読込・表示**
2. **メモノード・手動エッジの追加**
3. **要素へのメモ・カスタムラベル付与**
4. **編集済みSVGの保存・復元**
5. **タスクマネージャーとの双方向連携**（LocalStorage経由）

---

## 🗂️ 既存のメタデータ管理

### 1. SVG保存時のメタデータ構造（現状）

**保存場所**: SVGコメント内にJSON埋め込み

```javascript
// flowchart-editor.html:5968-6002
const metadata = {
  version: "1.0",
  exportedAt: new Date().toISOString(),
  svgFile: currentSvgFileName || 'unknown.svg',
  elements: {},              // 各要素のメモ・カスタムラベル
  originalLabels: {},        // SVG読込時の元のラベル
  manualNodes: {},           // メモノード（手動追加）
  manualEdges: {}            // 手動エッジ
};
```

### 2. 保存される情報の詳細

#### elements（要素のメタ情報）
```javascript
elements[elementId] = {
  type: "node",                          // node, edge, cluster
  displayText: "[USR_01] ユーザー",      // SVG上の表示テキスト
  memo: "初期状態のノード",              // ユーザーが入力したメモ
  customLabel: "[USR_01] ユーザー：問い合わせ発生",  // カスタムラベル
  originalLabel: "[USR_01] ユーザー"     // 元のラベル（復元用）
};
```

#### manualNodes（メモノード）
```javascript
manualNodes[nodeId] = {
  id: "MEMO_001",
  label: "重要な注意事項",
  x: 100,
  y: 200,
  width: 120,
  height: 60
};
```

#### manualEdges（手動エッジ）
```javascript
manualEdges[edgeId] = {
  id: "E_MANUAL_001",
  from: "MEMO_001",
  to: "U_START",
  label: "補足説明",
  x1: 100, y1: 200,
  x2: 300, y2: 250,
  controlX: 200, controlY: 225
};
```

---

## ❌ 現在の問題点

### 問題1: タスク連携情報がSVGに保存されない

**影響範囲**: タスクマネージャー（A側）⇔ フローチャートエディター（B側）連携

**問題の詳細**:
```javascript
// B側のメモリ上には存在
mockTasks = [
  {
    wbs: "0.1",
    taskName: "要件定義",
    status: "Completed",
    mermaidIds: "U_START, node_002",  // ← タスクとフローチャート要素の連携
    flowchartLinks: {
      nodes: ["U_START", "node_002"],
      edges: []
    }
  }
];

// しかしSVG保存時には含まれない（flowchart-editor.html:5968-6002）
// → リロード時にmockTasksが初期化され、連携情報が失われる
```

**現在の回避策**:
- A側（task-manager.html）のLocalStorageから毎回読み込み
- SVGをアップロードするたびに `initializeTasksFromTaskManager()` を実行

**問題点**:
- A側のデータがない場合、B側で編集した連携情報が失われる
- B側を単独で使用する場合に情報が永続化されない

### 問題2: メモノードと正規ノードの参照関係が不明確

**具体例**:
```
フローチャート画面:
┌────────────────────────────────────────┐
│   ┌─────────┐                         │
│   │ U_START │  正規ノード             │
│   └────┬────┘                         │
│        │                              │
│        ↓ 正規エッジ                   │
│   ┌─────────┐       ┌──────────┐     │
│   │ node_002│◄──────│ MEMO_001 │ メモノード
│   │[SYS_01] │  メモエッジ│ 重要！   │     │
│   └─────────┘ (手動)└──────────┘     │
└────────────────────────────────────────┘
```

**問題の内訳**:
1. **視覚的な矢印のみで判断**
   - MEMO_001がnode_002に関連することは矢印から推測できる
   - しかしメタデータとしては記録されていない

2. **SVG再読込時に意味が失われる**
   - manualEdgesには `from: "MEMO_001", to: "node_002"` のみ
   - なぜこのエッジが引かれたのか（参照関係の意図）が不明

3. **AI解析時に構造的理解が困難**
   - 将来的にAI解析用のJSON出力を予定
   - 視覚的な矢印だけでは意味的な関係性を伝えられない

### 問題3: ノード吸着機能の実装困難

**当初の計画**:
- メモエッジを正規ノードに自動吸着させる
- 接続ポイント（コネクタ）に紐づくノードIDを自動取得
- メタデータとして `connectedTo: "node_002"` を保存

**実装の困難点**（過去セッション `01_connector-adsorption` より）:
- SVG座標変換の複雑性（ズーム・パン・CTM）
- 正規ノードの境界検出の難しさ
- クラスタノード（サブグラフ）の入れ子構造

**現在の状況**:
- ノード吸着機能は保留
- 代替策として「人間が手動で参照関係を入力する」方式を検討中

---

## 🔧 関連する既存コード

### SVG保存処理（exportEditedSvg）

**ファイル**: `flowchart-editor.html:5948-6045`

```javascript
async function exportEditedSvg() {
  try {
    // File System Access APIでSVG保存
    if (!('showSaveFilePicker' in window)) {
      alert('このブラウザはFile System Access APIに対応していません。');
      return;
    }

    const svg = /* SVG要素取得 */;
    const svgClone = svg.cloneNode(true);

    // ★ メタデータをJSON形式で準備
    const metadata = {
      version: "1.0",
      exportedAt: new Date().toISOString(),
      svgFile: currentSvgFileName || 'unknown.svg',
      elements: {},
      originalLabels: originalLabels,
      manualNodes: manualNodes,
      manualEdges: manualEdges
      // ❌ mockTasks が含まれていない
      // ❌ 参照関係（references）も含まれていない
    };

    // elementMemos と elementCustomLabels を elements に統合
    const allElementIds = new Set([
      ...Object.keys(elementMemos),
      ...Object.keys(elementCustomLabels)
    ]);

    for (const elementId of allElementIds) {
      const element = document.querySelector(`[data-id="${elementId}"]`);
      if (!element) continue;

      metadata.elements[elementId] = {
        type: element.getAttribute('data-et') || 'unknown',
        displayText: getElementDisplayText(element),
        memo: elementMemos[elementId] || '',
        customLabel: elementCustomLabels[elementId] || '',
        originalLabel: originalLabels[elementId] || ''
        // ❌ references フィールドがない
      };
    }

    // ★ メタデータをSVGコメントとして埋め込み
    const metadataJson = JSON.stringify(metadata, null, 2);
    const metadataComment = document.createComment(`
FLOWCHART-MANUAL-DATA
${metadataJson}
`);

    svgClone.insertBefore(metadataComment, svgClone.firstChild);

    // SVGをシリアライズして保存
    const serializer = new XMLSerializer();
    const svgString = serializer.serializeToString(svgClone);

    const handle = await window.showSaveFilePicker({
      suggestedName: `${baseName}_editable.svg`,
      types: [{ description: 'SVG Files', accept: { 'image/svg+xml': ['.svg'] } }]
    });

    const writable = await handle.createWritable();
    await writable.write(svgString);
    await writable.close();

    console.log('編集済みSVGをエクスポートしました:', handle.name);
  } catch (err) {
    if (err.name !== 'AbortError') {
      console.error('エクスポートエラー:', err);
    }
  }
}
```

### SVG読込時の復元処理

**ファイル**: `flowchart-editor.html:4890-5050`

```javascript
// SVG挿入前にメタデータを抽出
const metadataMatch = svgText.match(/<!--\s*FLOWCHART-MANUAL-DATA\s*([\s\S]*?)-->/);
let extractedMetadata = null;

if (metadataMatch) {
  try {
    extractedMetadata = JSON.parse(metadataMatch[1]);
    console.log('✅ SVGからメタデータを抽出:', {
      version: extractedMetadata.version,
      manualNodesCount: Object.keys(extractedMetadata.manualNodes || {}).length,
      manualEdgesCount: Object.keys(extractedMetadata.manualEdges || {}).length,
      elementsCount: Object.keys(extractedMetadata.elements || {}).length
    });
  } catch (e) {
    console.warn('⚠️ メタデータ解析失敗:', e);
  }
}

// メタデータを復元
if (extractedMetadata) {
  // 元のラベル情報を復元
  if (extractedMetadata.originalLabels) {
    originalLabels = extractedMetadata.originalLabels;
  }

  // メモとカスタムラベルを復元
  if (extractedMetadata.elements) {
    elementMemos = {};
    elementCustomLabels = {};

    for (const [elementId, elementData] of Object.entries(extractedMetadata.elements)) {
      if (elementData.memo) {
        elementMemos[elementId] = elementData.memo;
      }
      if (elementData.customLabel) {
        elementCustomLabels[elementId] = elementData.customLabel;
        // SVG DOMに即座に反映
        const element = document.querySelector(`[data-id="${elementId}"]`);
        if (element) {
          updateSvgLabel(element, elementData.customLabel);
        }
      }
    }
  }

  // 手動ノードを復元
  if (extractedMetadata.manualNodes) {
    manualNodes = extractedMetadata.manualNodes;
    // イベントハンドラを再アタッチ
    for (const nodeId of Object.keys(manualNodes)) {
      attachDragBehavior(nodeId);
      // ...
    }
  }

  // 手動エッジを復元
  if (extractedMetadata.manualEdges) {
    manualEdges = extractedMetadata.manualEdges;
    // イベントハンドラを再アタッチ
  }

  // ❌ mockTasks の復元処理がない
  // ❌ references の復元処理もない
}
```

---

## 🔗 タスクマネージャー連携の現状

### データフロー（Phase 2-A完了時点）

```
A側（task-manager.html）              B側（flowchart-editor.html）
┌──────────────────┐                 ┌──────────────────┐
│ タスクデータ     │                 │ mockTasks        │
│ {                │                 │ （メモリ上のみ） │
│   wbs_no: "0.1", │ ─ LocalStorage → │ {               │
│   mermaid_ids: ""│     送信        │   wbs: "0.1",   │
│ }                │                 │   mermaidIds: ""│
└──────────────────┘                 │ }               │
        ↑                             └──────────────────┘
        │                                      │
        │  flowchart-attributes                │
        │  （フローチャート属性）              │
        └─────── LocalStorage ─────────────────┘
               publishFlowchartAttributes()
```

**問題点**:
- B側のmockTasksはSVGに保存されない
- SVG再読込時、A側のLocalStorageから毎回取得する必要がある
- A側のデータがない場合、B側で編集した連携情報が失われる

---

## 📝 変数とデータ構造

### グローバル変数（flowchart-editor.html）

```javascript
// 要素選択とメモ機能の変数
let selectedElement = null;
let elementMemos = {};           // { elementId: memoText }
let elementCustomLabels = {};    // { elementId: customLabel }
let originalLabels = {};         // { elementId: originalLabel }

// 手動エッジ・ノード
let manualEdges = {};            // { edgeId: edgeData }
let manualNodes = {};            // { nodeId: nodeData }

// タスク連携（メモリ上のみ）
let mockTasks = [];              // ❌ SVGに保存されない
```

### mockTasks の構造

```javascript
mockTasks = [
  {
    wbs: "0.1",
    taskName: "要件定義",
    status: "Completed",
    mermaidIds: "U_START, node_002",  // カンマ区切り文字列
    flowchartLinks: {
      nodes: ["U_START", "node_002"], // 配列形式
      edges: []
    }
  }
];
```

---

## 🎯 まとめ：修正が必要な箇所

### 1. SVG保存時（exportEditedSvg関数）
- [ ] `metadata.taskFlowchartLinks` を追加
- [ ] `metadata.elements[id].references` を追加
- [ ] `metadata.manualNodes[id].references` を追加

### 2. SVG読込時（復元処理）
- [ ] `extractedMetadata.taskFlowchartLinks` の復元
- [ ] `extractedMetadata.elements[id].references` の復元
- [ ] `extractedMetadata.manualNodes[id].references` の復元

### 3. UI（要素情報パネル）
- [ ] 参照関係セクションの追加
- [ ] 参照追加・削除UI
- [ ] オートコンプリート機能

---

**作成日**: 2026-03-22
**バージョン**: 1.0
**対象ファイル**: flowchart-editor.html（v2.17.1）
