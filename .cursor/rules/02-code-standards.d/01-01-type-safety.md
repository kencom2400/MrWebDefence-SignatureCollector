# 型安全性（最優先）

**優先度レベル**: `01-XX` - **必須（MUST）** - 絶対に守るべきルール

## 🎯 基本原則

### 🚨 型安全性（最優先・必須遵守）

- **any型の使用禁止**（テストのJestモック作成時のみ例外、理由コメント必須）
- **危険な型キャストの禁止**（`as unknown as`等）
- **Enum型の型安全な比較**（`Object.entries()`使用時は明示的型キャスト）
- **テストコードでも型安全性を保つ**

## 1. 型安全性のチェックリスト

### ❌ 絶対禁止事項

#### 1-1. 危険な型キャスト

```typescript
// ❌ 絶対禁止
apiClient: someObject as unknown as ITargetInterface;
apiClient: someObject as any;
```

**問題**: 型安全性を完全に損なう。実行時エラーの原因。

**✅ 正しい対応**:

```typescript
// ✅ インターフェース継承で型安全に
export interface ISourceInterface extends ITargetInterface {
  // 既存のメソッド
}

// ✅ 適切な型定義を作成
interface IAdapter {
  healthCheck(id: string): Promise<HealthCheckResult>;
}
```

#### 1-2. any型の安易な使用

```typescript
// ❌ 禁止
function process(data: any) {}
const result: any = fetchData();
```

**✅ 正しい対応**:

```typescript
// ✅ 適切な型定義
interface DataType {
  id: string;
  value: number;
}
function process(data: DataType): void {}

// ✅ 型が不明な場合はunknown
function process(data: unknown): void {
  if (typeof data === 'string') {
    // 型ガードで安全に使用
  }
}
```

**例外**: テストファイルでJestモック作成時のみ、コメント付きで許可

```typescript
// Jest型定義の制約によりany使用
const mockRepo = { findById: jest.fn() } as any;
```

#### 1-3. Enum型の型安全な比較

```typescript
// ❌ 避けるべきパターン
{Object.entries(CATEGORY_LABELS).map(([category, label]) => (
  <button
    onClick={() => setSelectedCategory(category as BankCategory)}
    className={String(selectedCategory) === category ? '...' : '...'}
  >
))}
```

**問題**:

- `Object.entries()`の戻り値は`[string, T][]`型
- 型アサーション（`as`）は型安全性を損なう

**✅ 正しい対応**:

```typescript
// ✅ 明示的な型キャストで型安全に
{(Object.entries(CATEGORY_LABELS) as [BankCategory, string][]).map(([category, label]) => (
  <button
    onClick={() => setSelectedCategory(category)}
    className={selectedCategory === category ? '...' : '...'}
  >
))}
```

#### 1-4. 型ガード関数の実装

型ガード関数（Type Guard）を実装する際は、型安全性を損なわないよう注意が必要です。

**❌ 避けるべきパターン**:

```typescript
// ❌ 型ガード関数内で、証明しようとしている型自身にキャストする
export function isHttpError(error: unknown): error is HttpError {
  return (
    error instanceof Error &&
    'statusCode' in error &&
    typeof (error as HttpError).statusCode === 'number' // ← 問題
  );
}
```

**問題点**:

- 型ガード関数が証明しようとしている型（`HttpError`）に、検証前にキャストしている
- 型ガードの目的（型の証明）と矛盾する
- 型安全性の観点で改善の余地がある

**✅ 正しい実装**:

```typescript
// ✅ より限定的な型アサーションを使用
export function isHttpError(error: unknown): error is HttpError {
  return (
    error instanceof Error &&
    'statusCode' in error &&
    typeof (error as { statusCode: unknown }).statusCode === 'number'
  );
}
```

**改善点**:

- `{ statusCode: unknown }` という最小限の型アサーションを使用
- 型ガード自体の堅牢性が向上
- TypeScriptの型システムをより適切に活用

**参考**: PR #237 - Gemini Code Assistレビュー指摘
