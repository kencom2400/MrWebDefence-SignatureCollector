## 5. ESLint設定のベストプラクティス

**優先度レベル**: `03-XX` - **一般（MAY）** - 推奨されるルール

### 基本方針

#### 5-1. 型情報を活用した静的解析（Type-aware Linting）

```javascript
// ✅ 推奨: typescript-eslint の型チェック有効化
export default tseslint.config(...tseslint.configs.recommendedTypeChecked, {
  languageOptions: {
    parserOptions: {
      projectService: true, // 型情報を利用
      tsconfigRootDir: import.meta.dirname,
    },
  },
});
```

#### 5-2. 包括的なルールセットの適用

```javascript
// ✅ Next.jsプロジェクトでの推奨設定
export default tseslint.config(js.configs.recommended, ...tseslint.configs.recommendedTypeChecked, {
  plugins: {
    react,
    'react-hooks': reactHooks,
    'jsx-a11y': jsxA11y,
    '@next/next': nextPlugin,
  },
  rules: {
    ...react.configs.recommended.rules,
    ...reactHooks.configs.recommended.rules,
    ...jsxA11y.configs.recommended.rules,
    ...nextPlugin.configs.recommended.rules,
    ...nextPlugin.configs['core-web-vitals'].rules,
  },
});
```

#### 5-3. 環境別の適切な設定

```javascript
// ✅ 推奨: 環境別設定
export default tseslint.config(
  // ソースコード: 厳格な設定
  {
    files: ['**/*.{ts,tsx}'],
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/explicit-function-return-type': 'warn',
    },
  },

  // テストコード: 一部緩和
  {
    files: ['**/__tests__/**/*.{ts,tsx}', '**/*.test.{ts,tsx}'],
    languageOptions: {
      globals: { ...globals.jest },
    },
    rules: {
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/explicit-function-return-type': 'off',
    },
  }
);
```

---

### 5-3. フロントエンドのエラーハンドリングとパフォーマンス

#### ❌ 避けるべきパターン: ユーザーへの通知なしのエラー処理

```typescript
// ❌ 悪い例: エラーがコンソールのみに出力される
useEffect(() => {
  const fetchCategories = async (): Promise<void> => {
    try {
      const data = await getCategories();
      setCategories(data);
    } catch (err) {
      console.error('カテゴリの取得に失敗しました:', err); // ユーザーには通知されない
    }
  };
  void fetchCategories();
}, []);
```

**問題**:

- ユーザーにエラーが通知されない
- 空のドロップダウンが表示され、UXが低下
- ユーザーは何が問題なのか分からない

#### ✅ 正しいパターン: ユーザーへの明示的なエラー表示

```typescript
// ✅ 良い例: エラーをUIに表示
useEffect(() => {
  const fetchCategories = async (): Promise<void> => {
    try {
      const data = await getCategories();
      setCategories(data);
    } catch (err) {
      setError('カテゴリの取得に失敗しました。ページを再読み込みしてください。');
      console.error('カテゴリの取得に失敗しました:', err);
    }
  };
  void fetchCategories();
}, []);

// UIでエラーを表示
{error && (
  <div className="mb-4 text-red-600 p-3 bg-red-50 rounded-md">{error}</div>
)}
```

**重要なポイント**:

- **エラーをユーザーに通知**: エラーメッセージを視覚的に表示
- **リカバリー方法を提示**: 「ページを再読み込みしてください」など
- **開発者向けログは維持**: `console.error`でデバッグ情報を残す

#### ❌ 避けるべきパターン: コンポーネント内でのヘルパー関数定義

```typescript
// ❌ 悪い例: コンポーネント内に定義
export function MyComponent({ data }: Props) {
  // この関数はレンダリングごとに再定義される
  const flattenTree = (nodes: Node[]): Item[] => {
    // ... 実装 ...
  };

  const flatData = flattenTree(data);
  // ...
}
```

**問題**:

- コンポーネントが再レンダリングされるたびに関数が再定義される
- パフォーマンスが低下
- メモリ使用量が増加

#### ✅ 正しいパターン: モジュールレベルでのヘルパー関数定義

```typescript
// ✅ 良い例: コンポーネント外に定義
const flattenTree = (nodes: Node[]): Item[] => {
  const result: Item[] = [];
  const traverse = (node: Node): void => {
    result.push(node.item);
    node.children.forEach(traverse);
  };
  nodes.forEach(traverse);
  return result;
};

export function MyComponent({ data }: Props) {
  const flatData = flattenTree(data);
  // ...
}
```

**重要なポイント**:

- **propsやstateに依存しない関数はコンポーネント外に**: 再定義を避ける
- **パフォーマンス向上**: 関数の参照が一定になる
- **可読性向上**: コンポーネントのロジックがシンプルになる

---
