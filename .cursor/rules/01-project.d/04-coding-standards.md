# コーディング規約

## コーディング規約

### 🚫 絶対禁止事項（CRITICAL）

#### 1. any型の使用禁止

**❌ 絶対に使用禁止:**

```typescript
function process(data: any) { ... }           // ❌ 禁止
const value: any = getSomeValue();            // ❌ 禁止
interface Props { data: any }                 // ❌ 禁止
```

**✅ 正しい方法:**

```typescript
// 型が不明な場合はunknownを使用
function process(data: unknown): void {
  if (typeof data === 'string') {
    // 型ガードで安全に使用
  }
}

// ジェネリクスを活用
function process<T>(data: T): T { ... }

// 適切な型定義を作成
interface ApiResponse {
  status: number;
  data: TransactionData;
}
```

**例外規定（テストファイルのみ）:**

```typescript
// ✅ テストファイルでモック作成時のみ許可
// ただし、必ず理由コメントを記載
const mockRepository = {
  findById: jest.fn() as any, // Jest型定義の制約によりany使用
};
```

**ESLint設定:**

- `@typescript-eslint/no-explicit-any`: `error` （厳守）
- テストファイル以外で`any`を使用したらビルド・コミットが失敗する

#### 2. Error/Warningの握りつぶし禁止

**❌ 絶対に禁止:**

```typescript
// ESLintルールを無効化
// eslint-disable-next-line @typescript-eslint/no-explicit-any  // ❌ 禁止

// TSエラーを無視
// @ts-ignore  // ❌ 禁止（理由なし）

// Warningをoffに変更
'@typescript-eslint/explicit-function-return-type': 'off',  // ❌ 禁止
```

**✅ 正しい対応:**

**原則: エラー・警告の根本原因を修正する**

```typescript
// 悪い例
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const _unused = value;

// 良い例: 不要な変数を削除
// (変数自体を削除)
```

**やむを得ない場合のみ、詳細なコメント付きで許可:**

```typescript
// ✅ 理由を明記した上で例外処理
// ESLint: ライブラリの型定義が不完全なため、
// 独自の型定義を作成するまでの暫定対応として使用
// TODO: #123 - 独自型定義を作成してこのコメントを削除
// @ts-expect-error Library type definition is incomplete
const result = externalLibrary.someMethod();
```

**コメントに必須の内容:**

1. **なぜ**このルール無効化が必要なのか
2. **いつまで**この状態が続くのか（TODO/Issue番号）
3. **どうすれば**解決できるのか

**警告レベルへの変更も原則禁止:**

```javascript
// ❌ 悪い例: 警告を握りつぶす
rules: {
  '@typescript-eslint/explicit-function-return-type': 'warn',  // 禁止
}

// ✅ 良い例: エラーのまま維持し、コードを修正
rules: {
  '@typescript-eslint/explicit-function-return-type': 'error',
}
// → コード側で戻り値型を明示的に追加
```

**例外: テストファイル・React/Next.jsコンポーネント**

```javascript
// ✅ テストファイルでのみ型安全性ルールを緩和
// 理由: モック作成時にany型が必要なため
{
  files: ['**/*.spec.ts', '**/*.test.ts'],
  rules: {
    '@typescript-eslint/no-explicit-any': 'off',  // テストのみ許可
  },
}

// ✅ React/Next.jsコンポーネントで戻り値型を省略
// 理由: JSX.Element型は冗長で、フレームワークの慣例に従う
{
  files: ['**/app/**/*.tsx', '**/components/**/*.tsx'],
  rules: {
    '@typescript-eslint/explicit-function-return-type': 'warn',  // warnレベルに緩和
  },
}
```

### TypeScript型安全性ルール

- 厳格な型定義を使用（`strict: true`）
- 共通型は`libs/types`に集約
- **型安全性の徹底** - すべての変数、引数、戻り値に対して明示的に型を付与すること：
  - **すべての変数に明示的な型注釈を付与**
    - 型推論に頼らず、常に明示的に型を指定すること
    - 例: `const value: string = "hello"` （`const value = "hello"` は不可）
  - **すべての関数の引数と戻り値に明示的な型を付与**
    - 例: `function process(data: string): number { ... }`
  - **exportされた関数・クラス・インターフェースは必ず型を明示**
    - モジュール境界での型安全性を確保
  - **interface、type定義も明示的に型を付与**
    - ジェネリクスパラメータも明示
    - 例: `interface Repository<T extends Entity> { ... }`
- nullableな値は明示的に型定義（例: `string | null`）
- **実装時点でlintエラーが基本的に出ないようにする**
  - ESLintの型安全性ルールがすべて`error`レベルで設定されている
  - 型エラーは即座に修正すること

### ネーミング規則

- **エンティティ**: PascalCase（例: `Transaction`, `Account`）
- **ユースケース**: 動詞 + 名詞（例: `FetchTransactionHistory`, `CalculateMonthlyBalance`）
- **リポジトリ**: 名詞 + Repository（例: `TransactionRepository`）
- **API接続**: 金融機関名 + ApiClient（例: `BankApiClient`, `CardApiClient`）

### ファイル構成

- 1ファイル1クラス/1機能
- テストファイルは対象ファイルと同じディレクトリに配置（`.spec.ts`）
- バレルファイル（`index.ts`）でエクスポートを整理

## コミットメッセージ

コミットメッセージの規約については、[Commit Rules](../03-git-workflow.d/02-commit-rules.md) を参照してください。
