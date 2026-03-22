# WBS番号体系 - 階層構造と自動生成

---

**feature**: タスク管理
**document**: WBS番号体系・自動生成アルゴリズム
**version**: 1.0
**status**: 仕様確定、バグ修正待ち
**created**: 2026-03-22
**updated**: 2026-03-22

---

## 📖 概要

Task ManagerのWBS（Work Breakdown Structure）番号体系を定義します。WBS番号は階層構造（`{Phase}.{Major}.{Minor}`）を持ち、タスクの組織化と自動生成を可能にします。

---

## 🏗️ WBS番号の構造

### 基本フォーマット

```
{Phase}.{Major}.{Minor}
```

### 各要素の定義

| 要素 | 型 | 範囲 | 説明 | 例 |
|-----|-----|------|------|-----|
| **Phase** | string | 任意 | フェーズ名（英大文字推奨） | TEST, DESIGN, RELEASE |
| **Major** | integer | 1~ | フェーズ内の主要タスク番号 | 1, 2, 15, 100 |
| **Minor** | integer | 0~ | Majorタスクのサブタスク番号 | 0, 1, 2, 3 |

---

### 具体例

```
TEST.1.0      - テストフェーズの1番目のタスク（親タスク）
TEST.1.1      - TEST.1.0のサブタスク1
TEST.1.2      - TEST.1.0のサブタスク2
TEST.15.0     - テストフェーズの15番目のタスク（親タスク）
TEST.15.1     - TEST.15.0のサブタスク1
DESIGN.3.0    - 設計フェーズの3番目のタスク
RELEASE.100.5 - リリースフェーズの100番目のタスクのサブタスク5
```

---

### 階層関係

```
Phase: TEST
├── TEST.1.0 (Major: 1, Minor: 0) - 親タスク
│   ├── TEST.1.1 (Major: 1, Minor: 1) - サブタスク
│   └── TEST.1.2 (Major: 1, Minor: 2) - サブタスク
├── TEST.2.0 (Major: 2, Minor: 0) - 親タスク
│   └── TEST.2.1 (Major: 2, Minor: 1) - サブタスク
└── TEST.15.0 (Major: 15, Minor: 0) - 親タスク
    ├── TEST.15.1 (Major: 15, Minor: 1) - サブタスク
    └── TEST.15.2 (Major: 15, Minor: 2) - サブタスク
```

---

## 🔢 WBS番号の自動生成

### 現在の実装（v2.13.4）

#### 関数: `generateWBSNumber()`
**ファイル**: task-manager.html
**行番号**: 3322-3351

```javascript
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // Find all tasks in the same phase
  const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);

  // Extract numeric parts and find max
  let maxNumber = 0;
  phaseTasks.forEach(t => {
    const parts = t.wbs_no.split('.');
    if (parts.length > 1) {
      const lastNum = parseFloat(parts[parts.length - 1]); // ⚠️ BUG: Minor番号を抽出
      if (!isNaN(lastNum) && lastNum > maxNumber) {
        maxNumber = lastNum;
      }
    }
  });

  const newNumber = Math.floor(maxNumber) + 1;
  const generatedWbsNo = `${phase}.${newNumber}.1`; // ⚠️ BUG: Minor番号をMajorとして使用

  document.getElementById('new-wbs-no').value = generatedWbsNo;
}
```

---

### ⚠️ **重大なバグ: Major/Minor番号抽出エラー**

#### 問題の詳細（Opus指摘）

**現在のコード（3338行目）**:
```javascript
const lastNum = parseFloat(parts[parts.length - 1]); // Minor番号を抽出
```

**問題点**:
- `parts[parts.length - 1]` は**Minor番号**（最後の要素）を抽出
- これを`maxNumber`として使い、Major番号として生成してしまう

#### バグの影響

**シナリオ1**: TEST.2.5 が存在する場合
```
既存タスク: TEST.2.5 (Phase: TEST, Major: 2, Minor: 5)
↓
parts = ["TEST", "2", "5"]
parts[parts.length - 1] = "5" ← Minor番号
maxNumber = 5
↓
生成されるWBS番号: TEST.6.1 ← 本来はTEST.3.1であるべき
```

**シナリオ2**: TEST.2.0, TEST.2.5, TEST.3.0 が存在する場合
```
既存タスク:
- TEST.2.0 (Major: 2, Minor: 0)
- TEST.2.5 (Major: 2, Minor: 5) ← Minor番号が最大
- TEST.3.0 (Major: 3, Minor: 0)
↓
maxNumber = 5 (TEST.2.5のMinor番号)
↓
生成されるWBS番号: TEST.6.1 ← 本来はTEST.4.1であるべき
```

---

### ✅ **修正版: 正しいMajor番号抽出**

#### 修正後のコード
```javascript
function generateWBSNumber() {
  const phase = document.getElementById('new-phase').value;
  if (!phase) {
    alert('フェーズを先に選択してください');
    return;
  }

  // Find all tasks in the same phase (削除済みタスクも含む - Option 1対応)
  const phaseTasks = projectData.tasks.filter(t => t.phase === phase);

  // Extract Major number and find max
  let maxMajor = 0;
  phaseTasks.forEach(t => {
    const parts = t.wbs_no.split('.');
    if (parts.length >= 2) {
      const majorNum = parseInt(parts[1], 10); // ← Major番号を抽出（2番目の要素）
      if (!isNaN(majorNum) && majorNum > maxMajor) {
        maxMajor = majorNum;
      }
    }
  });

  const newMajor = maxMajor + 1;
  const generatedWbsNo = `${phase}.${newMajor}.1`;

  document.getElementById('new-wbs-no').value = generatedWbsNo;
}
```

#### 修正箇所
1. **3330行目**: `&& !t.deleted` を削除（Option 1対応）
2. **3338-3342行目**: Major番号抽出ロジックを修正

```diff
- const phaseTasks = projectData.tasks.filter(t => t.phase === phase && !t.deleted);
+ const phaseTasks = projectData.tasks.filter(t => t.phase === phase);

- let maxNumber = 0;
+ let maxMajor = 0;
  phaseTasks.forEach(t => {
    const parts = t.wbs_no.split('.');
-   if (parts.length > 1) {
-     const lastNum = parseFloat(parts[parts.length - 1]);
-     if (!isNaN(lastNum) && lastNum > maxNumber) {
-       maxNumber = lastNum;
+   if (parts.length >= 2) {
+     const majorNum = parseInt(parts[1], 10);
+     if (!isNaN(majorNum) && majorNum > maxMajor) {
+       maxMajor = majorNum;
      }
    }
  });

- const newNumber = Math.floor(maxNumber) + 1;
- const generatedWbsNo = `${phase}.${newNumber}.1`;
+ const newMajor = maxMajor + 1;
+ const generatedWbsNo = `${phase}.${newMajor}.1`;
```

---

### 修正後の動作

**シナリオ1**: TEST.2.5 が存在する場合
```
既存タスク: TEST.2.5 (Phase: TEST, Major: 2, Minor: 5)
↓
parts = ["TEST", "2", "5"]
parts[1] = "2" ← Major番号
maxMajor = 2
↓
生成されるWBS番号: TEST.3.1 ✅ 正しい
```

**シナリオ2**: TEST.2.0, TEST.2.5, TEST.3.0 が存在する場合
```
既存タスク:
- TEST.2.0 (Major: 2, Minor: 0)
- TEST.2.5 (Major: 2, Minor: 5)
- TEST.3.0 (Major: 3, Minor: 0) ← Major番号が最大
↓
maxMajor = 3
↓
生成されるWBS番号: TEST.4.1 ✅ 正しい
```

---

## 🔍 Major/Minor番号の抽出ロジック

### 正しい抽出方法

```javascript
// WBS番号: "TEST.15.3"
const parts = wbsNo.split('.');

const phase = parts[0];           // "TEST"
const major = parseInt(parts[1], 10); // 15
const minor = parseInt(parts[2], 10); // 3
```

### エッジケース

| WBS番号 | parts配列 | Phase | Major | Minor | 備考 |
|--------|----------|-------|-------|-------|------|
| `TEST.1.0` | ["TEST", "1", "0"] | TEST | 1 | 0 | 正常 |
| `TEST.15.1` | ["TEST", "15", "1"] | TEST | 15 | 1 | 正常 |
| `TEST.100.25` | ["TEST", "100", "25"] | TEST | 100 | 25 | 正常 |
| `TEST.1` | ["TEST", "1"] | TEST | 1 | undefined | ⚠️ Minor省略 |
| `TEST` | ["TEST"] | TEST | undefined | undefined | ⚠️ 不正 |
| `TEST.1.2.3` | ["TEST", "1", "2", "3"] | TEST | 1 | 2 | ⚠️ 4階層 |

### バリデーション

```javascript
function validateWBSNumber(wbsNo) {
  const parts = wbsNo.split('.');

  // 最低3要素が必要
  if (parts.length < 3) {
    return { valid: false, error: 'WBS番号は「Phase.Major.Minor」形式である必要があります' };
  }

  // Phase（文字列）
  if (!parts[0] || parts[0].trim() === '') {
    return { valid: false, error: 'Phaseが空です' };
  }

  // Major（整数）
  const major = parseInt(parts[1], 10);
  if (isNaN(major) || major < 1) {
    return { valid: false, error: 'Major番号は1以上の整数である必要があります' };
  }

  // Minor（整数）
  const minor = parseInt(parts[2], 10);
  if (isNaN(minor) || minor < 0) {
    return { valid: false, error: 'Minor番号は0以上の整数である必要があります' };
  }

  return { valid: true };
}
```

---

## 🧪 テストケース

### 自動生成のテストケース

#### テストケース1: 空のフェーズ
```javascript
// 既存タスク: なし（TESTフェーズが空）
// 期待結果: TEST.1.1
```

#### テストケース2: 単一タスク存在
```javascript
// 既存タスク: TEST.5.0
// 期待結果: TEST.6.1
```

#### テストケース3: 複数タスク、Minor番号が大きい
```javascript
// 既存タスク:
// - TEST.2.0
// - TEST.2.5 (Minor番号が大きい)
// - TEST.3.0
// 期待結果: TEST.4.1（修正前: TEST.6.1 ← バグ）
```

#### テストケース4: 削除済みタスク含む（Option 1対応）
```javascript
// 既存タスク:
// - TEST.5.0 (deleted: false)
// - TEST.6.0 (deleted: true) ← 削除済み
// - TEST.7.0 (deleted: false)
// 期待結果: TEST.8.1（削除済みも履歴に含める）
```

#### テストケース5: 飛び番
```javascript
// 既存タスク:
// - TEST.1.0
// - TEST.3.0 (TEST.2.0は存在しない)
// - TEST.10.0
// 期待結果: TEST.11.1
```

---

## 🔧 手動入力のガイドライン

### 推奨フォーマット

#### 親タスク（Major.0）
```
TEST.1.0
TEST.2.0
TEST.15.0
```

#### サブタスク（Major.Minor）
```
TEST.1.1
TEST.1.2
TEST.15.1
TEST.15.2
TEST.15.3
```

---

### 手動入力時の注意事項

1. **Phase名の統一**: 大文字英字推奨（例: TEST, DESIGN, RELEASE）
2. **Major番号の連番**: 飛び番を避ける（1, 2, 3, ...）
3. **Minor番号の開始**: 親タスクは`.0`、サブタスクは`.1`から
4. **重複チェック**: 既存のWBS番号と重複しないこと

---

## 📊 統計と分析

### 現在のプロジェクトデータ分析（例）

```javascript
// 全タスクのWBS番号を分析
function analyzeWBSNumbers() {
  const analysis = {};

  projectData.tasks.forEach(task => {
    const parts = task.wbs_no.split('.');
    const phase = parts[0];

    if (!analysis[phase]) {
      analysis[phase] = { count: 0, maxMajor: 0, maxMinor: 0 };
    }

    analysis[phase].count++;
    const major = parseInt(parts[1], 10);
    const minor = parseInt(parts[2], 10);

    if (major > analysis[phase].maxMajor) analysis[phase].maxMajor = major;
    if (minor > analysis[phase].maxMinor) analysis[phase].maxMinor = minor;
  });

  return analysis;
}

// 例: { TEST: { count: 15, maxMajor: 10, maxMinor: 5 } }
```

---

## 🚀 将来の拡張

### 4階層対応（オプション）

```
{Phase}.{Major}.{Minor}.{SubMinor}

例:
TEST.1.1.1 - サブタスクのさらなる細分化
TEST.1.1.2
```

**実装要否**: 現時点では不要、大規模プロジェクトで検討

---

### WBS番号の自動リナンバリング

```javascript
// フェーズ内のMajor番号を1, 2, 3, ... に再割り当て
function renumberPhase(phase) {
  const phaseTasks = projectData.tasks
    .filter(t => t.phase === phase && !t.deleted)
    .sort((a, b) => {
      const aMajor = parseInt(a.wbs_no.split('.')[1], 10);
      const bMajor = parseInt(b.wbs_no.split('.')[1], 10);
      return aMajor - bMajor;
    });

  let newMajor = 1;
  let currentMajor = null;

  phaseTasks.forEach(task => {
    const parts = task.wbs_no.split('.');
    const oldMajor = parseInt(parts[1], 10);

    if (oldMajor !== currentMajor) {
      currentMajor = oldMajor;
      newMajor++;
    }

    task.wbs_no = `${phase}.${newMajor}.${parts[2]}`;
  });
}
```

**実装要否**: 現時点では不要、ユーザー要望があれば検討

---

## 📝 関連ドキュメント

- [01-data-structure.md](./01-data-structure.md) - タスクデータ構造
- [03-deletion-policy.md](./03-deletion-policy.md) - 削除ポリシー（Option 1実装）
- [archive/session/03_ID-policy/01_current-implementation.md](../../../archive/session/03_ID-policy/01_current-implementation.md) - 現在の実装詳細
- [archive/session/03_ID-policy/feedback/01_opus_WBS番号再利用ポリシー 総合評価レポート.md](../../../archive/session/03_ID-policy/feedback/01_opus_WBS番号再利用ポリシー%20総合評価レポート.md) - Opusのバグ指摘

---

## 📅 更新履歴

| 日付 | 変更内容 |
|------|---------|
| 2026-03-22 | 初版作成、Major/Minor番号抽出バグを明記、修正版を提示 |

---

**最終更新**: 2026-03-22
**ステータス**: 仕様確定、バグ修正待ち
**次のアクション**: Major/Minor番号抽出バグの修正（15-20分）
