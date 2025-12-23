## 7. React/UIコンポーネント

**優先度レベル**: `03-XX` - **一般（MAY）** - 推奨されるルール

### 7-1. パフォーマンス最適化（useMemoの活用）

**原則**: コンポーネントの再レンダリングごとに実行される計算処理は`useMemo`でメモ化する。

#### ❌ 悪い例: レンダリングごとに計算

```typescript
// ❌ 悪い例: レンダリングごとに配列を走査
export function TransactionClassificationPage() {
  const [transactions, setTransactions] = useState<Transaction[]>([]);

  return (
    <div>
      <p>未分類: {transactions.filter((tx) => !tx.subcategoryId).length}</p>
      <p>
        低信頼度:{' '}
        {
          transactions.filter(
            (tx) =>
              tx.classificationConfidence !== undefined &&
              tx.classificationConfidence !== null &&
              tx.classificationConfidence < 0.7
          ).length
        }
      </p>
    </div>
  );
}
```

**問題点**:

- コンポーネントが再レンダリングされるたびに配列全体を走査
- 取引件数が多い場合、パフォーマンスのボトルネックになる
- 不要な計算が繰り返される

#### ✅ 良い例: useMemoでメモ化

```typescript
// ✅ 良い例: useMemoでメモ化
export function TransactionClassificationPage() {
  const [transactions, setTransactions] = useState<Transaction[]>([]);

  // 統計情報のメモ化
  const stats = useMemo(() => {
    return {
      unclassifiedCount: transactions.filter((tx) => !tx.subcategoryId).length,
      lowConfidenceCount: transactions.filter(
        (tx) =>
          tx.classificationConfidence !== undefined &&
          tx.classificationConfidence !== null &&
          tx.classificationConfidence < 0.7
      ).length,
    };
  }, [transactions]);

  return (
    <div>
      <p>未分類: {stats.unclassifiedCount}</p>
      <p>低信頼度: {stats.lowConfidenceCount}</p>
    </div>
  );
}
```

**利点**:

- `transactions`配列が変更された場合にのみ再計算
- 不要な計算を避け、パフォーマンスが向上
- コードの可読性も向上

### 7-2. イミュータブルな状態更新

**原則**: Reactの状態更新は常にイミュータブルに行う。配列やオブジェクトを直接変更しない。

#### ❌ 悪い例: ミュータブルな更新

```typescript
// ❌ 悪い例: 配列を直接変更
const handleBatchClassify = async () => {
  const result = await subcategoryApi.batchClassify({ transactions: requests });

  const updatedTransactions = [...transactions];
  for (const classificationResult of result.results) {
    if (classificationResult.success && classificationResult.subcategoryId) {
      const txIndex = updatedTransactions.findIndex(
        (tx) => tx.id === classificationResult.transactionId
      );
      if (txIndex !== -1) {
        // 配列の要素を直接変更（ミュータブル）
        updatedTransactions[txIndex] = {
          ...updatedTransactions[txIndex],
          subcategoryId: classificationResult.subcategoryId,
        };
      }
    }
  }
  setTransactions(updatedTransactions);
};
```

**問題点**:

- 配列の要素を直接変更している（ミュータブル）
- Reactのイミュータブルな状態更新の原則に反する
- 意図しない副作用やバグの原因となる可能性

#### ✅ 良い例: イミュータブルな更新（map使用）

```typescript
// ✅ 良い例: mapを使用したイミュータブルな更新
const handleBatchClassify = async () => {
  const result = await subcategoryApi.batchClassify({ transactions: requests });

  // 結果をMapに変換して効率的に検索
  const resultMap = new Map(
    result.results.filter((r) => r.success && r.subcategoryId).map((r) => [r.transactionId, r])
  );

  // mapを使用してイミュータブルに更新
  setTransactions((prev) =>
    prev.map((tx) => {
      const classificationResult = resultMap.get(tx.id);
      if (classificationResult) {
        return {
          ...tx,
          subcategoryId: classificationResult.subcategoryId,
          classificationConfidence: classificationResult.confidence ?? null,
          classificationReason: classificationResult.reason ?? null,
        };
      }
      return tx;
    })
  );
};
```

**利点**:

- 完全にイミュータブルな更新
- コードの可読性と予測可能性が向上
- Reactのベストプラクティスに準拠

### 7-3. 共通ロジックのユーティリティ化

**原則**: 複数のコンポーネントで使用されるロジックは、共通のユーティリティ関数として抽出する。

#### ❌ 悪い例: ロジックの重複

```typescript
// ❌ 悪い例: 各コンポーネントで同じロジックを実装
// ClassificationBadge.tsx
const getReasonText = (): string => {
  switch (reason) {
    case ClassificationReason.MERCHANT_MATCH:
      return merchantName ? `店舗マスタ一致: ${merchantName}` : '店舗マスタ一致';
    case ClassificationReason.KEYWORD_MATCH:
      return 'キーワード一致';
    // ...
  }
};

// TransactionDetailModal.tsx
<p>分類理由: {transaction.classificationReason}</p> // enumキーのまま表示
```

**問題点**:

- 同じロジックが複数箇所に存在（DRY原則違反）
- 修正時に複数箇所を更新する必要がある
- ユーザーフレンドリーでない表示（enumキーのまま）

#### ✅ 良い例: ユーティリティ関数の共通化

```typescript
// ✅ 良い例: 共通ユーティリティ関数を作成
// utils/classification.utils.ts
export function getClassificationReasonText(
  reason: ClassificationReason,
  merchantName?: string | null
): string {
  switch (reason) {
    case ClassificationReason.MERCHANT_MATCH:
      return merchantName ? `店舗マスタ一致: ${merchantName}` : '店舗マスタ一致';
    case ClassificationReason.KEYWORD_MATCH:
      return 'キーワード一致';
    case ClassificationReason.AMOUNT_INFERENCE:
      return '金額推測';
    case ClassificationReason.RECURRING_PATTERN:
      return '定期性判定';
    case ClassificationReason.DEFAULT:
      return 'デフォルト';
    default:
      return '不明';
  }
}

// ClassificationBadge.tsx
import { getClassificationReasonText } from '@/utils/classification.utils';

const reasonText = getClassificationReasonText(reason, merchantName);

// TransactionDetailModal.tsx
import { getClassificationReasonText } from '@/utils/classification.utils';

<p>
  分類理由:{' '}
  {getClassificationReasonText(
    transaction.classificationReason,
    transaction.merchantName
  )}
</p>
```

**利点**:

- DRY原則の遵守
- 一箇所で管理・修正が可能
- ユーザーフレンドリーな表示を統一

### 7-4. データ構造の最適化（Mapの活用）

**原則**: 頻繁に参照されるデータは、配列の`find`や`filter`ではなく、`Map`を使ったO(1)参照に最適化する。

#### ❌ 悪い例: O(n)の線形検索

```typescript
// ❌ 悪い例: 配列のfindでO(n)検索
interface SubcategoryStore {
  subcategories: Subcategory[];
  getSubcategoryById: (id: string) => Subcategory | undefined;
}

export const useSubcategoryStore = create<SubcategoryStore>((set, get) => ({
  subcategories: [],
  getSubcategoryById: (id: string) => {
    // O(n)の線形検索
    return get().subcategories.find((sub) => sub.id === id);
  },
}));

// コンポーネント内で各取引のレンダリング時に呼び出される
// 取引がT個、サブカテゴリがS個ある場合、計算量はO(T*S)
```

**問題点**:

- 各レンダリング時に配列全体を走査（O(n)）
- 取引数が多い場合、パフォーマンスのボトルネックになる
- 計算量がO(T\*S)となり、スケーラビリティが低い

#### ✅ 良い例: Mapを使ったO(1)参照

```typescript
// ✅ 良い例: Mapを使ったO(1)参照
interface SubcategoryStore {
  subcategories: Subcategory[];
  subcategoryMap: Map<string, Subcategory>; // IDをキーとするMap
  getSubcategoryById: (id: string) => Subcategory | undefined;
}

export const useSubcategoryStore = create<SubcategoryStore>((set, get) => ({
  subcategories: [],
  subcategoryMap: new Map<string, Subcategory>(),
  fetchSubcategories: async (categoryType?: CategoryType) => {
    const data = await subcategoryApi.getByCategory(categoryType);
    // IDをキーとするMapを作成（O(1)参照用）
    const map = new Map<string, Subcategory>();
    for (const subcategory of data) {
      map.set(subcategory.id, subcategory);
    }
    set({ subcategories: data, subcategoryMap: map });
  },
  getSubcategoryById: (id: string) => {
    // O(1)の参照
    return get().subcategoryMap.get(id);
  },
}));
```

**利点**:

- O(1)の参照により、パフォーマンスが大幅に改善
- 計算量がO(T)に削減（T: 取引数）
- スケーラビリティが向上

### 7-5. 階層構造構築の最適化

**原則**: 階層構造を構築する際は、親IDをキーとするMapを作成してからツリーを構築することで、計算量を削減する。

#### ❌ 悪い例: 再帰的なfilter呼び出し

```typescript
// ❌ 悪い例: 再帰的にfilterを呼び出す
const buildTree = (allSubcategories: Subcategory[]): Subcategory[] => {
  const rootCategories = allSubcategories.filter((sub) => sub.parentId === null);

  const buildChildren = (parentId: string | null): Subcategory[] => {
    // 毎回配列全体を走査（O(n)）
    const children = allSubcategories.filter((sub) => sub.parentId === parentId);
    return children.map((child) => ({
      ...child,
      children: buildChildren(child.id), // 再帰的にfilterを呼び出し
    }));
  };

  return rootCategories.map((root) => ({
    ...root,
    children: buildChildren(root.id),
  }));
};
```

**問題点**:

- 再帰的に`filter`を呼び出すため、計算量がO(n²)になる可能性
- サブカテゴリ数が多い場合に非効率

#### ✅ 良い例: Mapを使った効率的な構築

```typescript
// ✅ 良い例: 親IDをキーとするMapを作成
const buildTree = (allSubcategories: Subcategory[]): Subcategory[] => {
  // 親IDをキーとする子のMapを作成（O(n)）
  const childrenMap = new Map<string | null, Subcategory[]>();
  for (const sub of allSubcategories) {
    const parentId = sub.parentId;
    if (!childrenMap.has(parentId)) {
      childrenMap.set(parentId, []);
    }
    childrenMap.get(parentId)!.push(sub);
  }

  // 親カテゴリ（parentIdがnull）を取得
  const rootCategories = childrenMap.get(null) || [];

  // 階層構造を構築（Mapを使用してO(1)参照）
  const buildChildren = (parentId: string | null): Subcategory[] => {
    const children = childrenMap.get(parentId) || [];
    return children.map((child) => ({
      ...child,
      children: buildChildren(child.id),
    }));
  };

  return rootCategories.map((root) => ({
    ...root,
    children: buildChildren(root.id),
  }));
};
```

**利点**:

- 計算量がO(n)に削減
- Mapを使ったO(1)参照により、パフォーマンスが向上
- サブカテゴリ数が多い場合でも効率的

### 7-6. ロジックの一元化（ストアへの集約）

**原則**: 複数のコンポーネントで使用されるロジックは、状態管理ストアに一元化する。

#### ❌ 悪い例: ロジックの重複

```typescript
// ❌ 悪い例: コンポーネント内とストア内の両方に階層構造構築ロジックが存在
// SubcategorySelector.tsx
const tree = useMemo(() => {
  const filtered = subcategories.filter(...);
  const buildChildren = (parentId: string | null) => {
    const children = filtered.filter((sub) => sub.parentId === parentId);
    // ...
  };
  // ...
}, [subcategories, categoryType, searchQuery]);

// subcategory.store.ts
buildTree: (categoryType?: CategoryType) => {
  const buildChildren = (parentId: string | null) => {
    const children = allSubcategories.filter((sub) => sub.parentId === parentId);
    // ...
  };
  // ...
}
```

**問題点**:

- ロジックが重複している
- 将来的な変更時に片方を修正し忘れる可能性
- 不整合の原因となる

#### ✅ 良い例: ストアに一元化

```typescript
// ✅ 良い例: ストアのbuildTreeを使用
// SubcategorySelector.tsx
const { buildTree } = useSubcategoryStore();

const tree = useMemo(() => {
  // ストアから階層構造を取得
  const fullTree = buildTree(categoryType);
  // 検索クエリでフィルタリング（必要に応じて）
  if (!searchQuery) {
    return fullTree;
  }
  // ...
}, [buildTree, categoryType, searchQuery]);
```

**利点**:

- ロジックの一元化により、保守性が向上
- 修正時に1箇所のみ更新すればよい
- 関心の分離が促進される

### 7-7. E2Eテストの信頼性向上

**原則**: E2Eテストでは、固定時間の待機（`waitForTimeout`）を避け、UIの状態変化を待つ適切な方法を使用する。

#### ❌ 悪い例: waitForTimeoutの使用

```typescript
// ❌ 悪い例: 固定時間での待機
test('フィルターが機能する', async ({ page }) => {
  await page.getByLabel('カテゴリ').selectOption('EXPENSE');
  await page.waitForTimeout(500); // 固定時間での待機
  // アサーション
});
```

**問題点**:

- テストの実行環境によって成功したり失敗したりする（flaky test）
- 実際のUI更新を待たずにアサーションを実行する可能性
- テストの信頼性が低い

#### ✅ 良い例: 適切な待機方法

```typescript
// ✅ 良い例: UIの状態変化を待つ
test('フィルターが機能する', async ({ page }) => {
  // ページが読み込まれるまで待機
  await page.waitForLoadState('networkidle');

  // フィルターを選択
  await page.getByLabel('カテゴリ').selectOption('EXPENSE');

  // フィルターが適用されることを確認（テーブルが表示されるか、メッセージが表示される）
  await expect(
    page.getByRole('table').or(page.getByText('該当する取引がありません'))
  ).toBeVisible();
});
```

**利点**:

- 実際のUI更新を待ってからアサーションを実行
- テストの信頼性が向上
- 環境に依存しない安定したテスト

#### ✅ 良い例: getByRoleを使用したセレクタ

```typescript
// ❌ 悪い例: idをgetByLabelで探す
const checkbox = page.getByLabel('unclassified-only'); // idはgetByLabelの対象ではない

// ✅ 良い例: ロールとアクセシブルネームで要素を特定
const checkbox = page.getByRole('checkbox', { name: '未分類のみ' });
```

**利点**:

- より堅牢で意図が明確なテスト
- アクセシビリティの観点からも適切

**参考**: PR #303 - Gemini Code Assistレビュー指摘

### 7-8. コールバック関数の型定義

**非同期処理に対応したコールバック型**:

❌ **悪い例**:

```typescript
interface Props {
  onRetry?: () => void;
}

const handleRetry = (): void => {
  if (!onRetry) return;
  onRetry(); // 非同期処理の完了を待てない
  onClose(); // リトライ完了前に閉じてしまう
};
```

✅ **良い例**:

```typescript
interface Props {
  onRetry?: () => Promise<void> | void; // Promise対応
}

const handleRetry = async (): Promise<void> => {
  if (!onRetry) return;
  await onRetry(); // 完了を待つ
  onClose(); // 完了後に閉じる
};
```

### 7-9. 日時の固定化

❌ **悪い例**:

```typescript
<ErrorModal
  timestamp={new Date()}  // 再レンダリングで変わる
/>
```

✅ **良い例**:

```typescript
const errorTimestampRef = useRef<Date | null>(null);

const handleError = (message: string): void => {
  if (!errorTimestampRef.current) {
    errorTimestampRef.current = new Date();
  }
};

<ErrorModal
  timestamp={errorTimestampRef.current || undefined}
/>
```

### 7-10. クロージャとuseCallbackの注意点

**問題**: `useCallback`の依存配列に状態を含めると、コールバックがその時点の値をキャプチャしてしまい、後で状態が変更されても古い値を参照し続ける

❌ **悪い例**:

```typescript
const [formData, setFormData] = useState<FormData>({...});

// handleErrorが呼ばれた時点のformDataをキャプチャ
const handleError = useCallback(
  (errorMessage: string): void => {
    showErrorToast('error', errorMessage, {
      onRetry: () => {
        // ここでキャプチャされたformDataは古い可能性がある
        if (validate()) {
          onSubmit(formData); // ❌ ユーザーが値を変更しても古いデータが送信される
        }
      },
    });
  },
  [formData, validate, onSubmit] // formDataが依存配列に含まれる
);
```

**問題点**:

- エラー通知表示後にユーザーがフォームを変更しても、「再試行」ボタンで古いデータが送信される
- ユーザーの最新の入力が反映されない

✅ **良い例**:

```typescript
const [formData, setFormData] = useState<FormData>({...});
const formDataRef = useRef(formData);

// formDataRefを常に最新の状態に保つ
useEffect(() => {
  formDataRef.current = formData;
}, [formData]);

// validate関数がデータ引数を受け取るように変更
const validate = useCallback((dataToValidate: FormData): boolean => {
  const newErrors: Record<string, string> = {};
  // dataToValidateを使ってバリデーション
  setErrors(newErrors);
  return Object.keys(newErrors).length === 0;
}, []); // setErrorsは安定しているため依存配列は空

// handleErrorでformDataRefを使用
const handleError = useCallback(
  (errorMessage: string): void => {
    showErrorToast('error', errorMessage, {
      onRetry: () => {
        // 最新のformDataを参照
        if (validate(formDataRef.current)) {
          onSubmit(formDataRef.current); // ✅ 常に最新のデータが送信される
        }
      },
    });
  },
  [validate, onSubmit] // formDataは依存配列から除外
);
```

**改善点**:

- `formDataRef`を使って常に最新のフォームデータを参照
- `validate`関数をデータ引数を受け取るように変更し、依存配列を空に
- `handleError`の依存配列から`formData`を削除し、クロージャ問題を解決
- エラー通知表示後にユーザーがフォームを変更しても、「再試行」ボタンで最新のデータが送信される

**参考**: PR #238 - Gemini Code Assistレビュー指摘

---
