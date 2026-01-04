# Account Book - 資産管理アプリ開発ガイドライン

## ⚠️ AIアシスタントへの重要指示（最優先）

### 型安全性の絶対遵守

**このプロジェクトでは型安全性を最優先します。以下のルールを絶対に守ってください。**

1. **any型は絶対に使用しない**
   - ❌ 「とりあえずany型で」は厳禁
   - ✅ 型が不明な場合は`unknown`を使用
   - ✅ 適切な型定義・インターフェースを作成
   - ✅ ジェネリクスを活用

2. **エラー・警告を握りつぶさない**
   - ❌ ESLintエラーを`eslint-disable`で無効化しない
   - ❌ 警告を`warn`レベルに変更して隠さない
   - ❌ `@ts-ignore`で型エラーを無視しない
   - ✅ **根本原因を修正する**
   - ✅ やむを得ない場合は詳細なコメント（なぜ・いつまで・どうすれば）を必須化

3. **実装時の必須確認**
   - コード変更後は必ず`lint`と`build`を実行
   - エラー・警告が出たら、コミット前に**必ず修正**
   - テストコードも型安全性を守る（テストだからanyで良い、は誤り）

**違反例を提案した場合、ユーザーから指摘されます。必ず守ってください。**

## プロジェクト概要

金融機関APIを使用した個人資産管理アプリケーション。取引履歴の自動取得、カテゴリ分類、月次集計機能を提供する。

## 技術スタック

### モノレポ構成

- `apps/frontend`: Next.js（フロントエンド）
- `apps/backend`: NestJS（バックエンド）
- `libs/types`: 共通型定義（TypeScript）

### 環境管理

- env環境を使用し、プロジェクト内で完結した言語環境を構築
- パッケージマネージャー: pnpm推奨（モノレポ構成のため）

#### ローカル環境でのpnpm使用方法

このプロジェクトは`nodeenv`を使用しており、Node.js、pnpm、その他のツールは`.nodeenv`ディレクトリ内にローカルインストールされています。

**重要**: ターミナルで直接`pnpm`や`node`コマンドを実行する前に、必ずnodeenv環境をアクティベートする必要があります。

##### 環境のアクティベート

```bash
# プロジェクトルートで実行
cd /path/to/account-book
source scripts/setup/activate.sh
```

アクティベート後、以下のコマンドが使用可能になります：

- `node`
- `pnpm`
- `npm`
- `npx`

##### よく使うコマンド

```bash
# 環境をアクティベート
source scripts/setup/activate.sh

# 依存関係のインストール
pnpm install

# 開発サーバーの起動
pnpm dev

# Lint実行
pnpm lint

# Lint自動修正
pnpm run lint -- --fix

# テスト実行
pnpm test

# ビルド
pnpm build
```

##### トラブルシューティング

**問題**: `command not found: pnpm` や `command not found: node` が表示される

**原因**: nodeenv環境がアクティベートされていない

**解決方法**:

```bash
source scripts/setup/activate.sh
```

**問題**: `Error: .nodeenv directory not found`

**原因**: `.nodeenv`ディレクトリが存在しない

**解決方法**: セットアップスクリプトを実行

```bash
./scripts/full-setup.sh
```

##### AIアシスタント向けの注意

- ターミナルコマンドを実行する際は、必ず`source scripts/setup/activate.sh`を最初に実行すること
- サンドボックス環境の問題ではなく、PATHの問題であることに注意
- `required_permissions: ["all"]`を指定して実行すること

## アーキテクチャ原則

### Onion Architecture（オニオンアーキテクチャ）

レイヤを明確に分離し、依存関係を内側に向けることで拡張性と保守性を確保する。

**依存関係は常に内側に向かいます：**

- **Presentation Layer** は Application Layer に依存
- **Infrastructure Layer** は Application Layer に依存
- **Application Layer** は Domain Layer に依存
- **Domain Layer** は外部に依存しない（最内層）

#### レイヤ構成

1. **Domain Layer**: エンティティ、ドメインロジック、ドメインサービス
2. **Application Layer**: ユースケース、アプリケーションサービス
3. **Infrastructure Layer**: 外部API接続、データ永続化（JSON/DB）
4. **Presentation Layer**: コントローラー、API、UI

#### ディレクトリ構造（backend）

```
apps/backend/src/
├── domain/           # ドメイン層
│   ├── entities/
│   ├── value-objects/
│   └── repositories/ # インターフェース定義
├── application/      # アプリケーション層
│   ├── use-cases/
│   └── services/
├── infrastructure/   # インフラ層
│   ├── api/         # 金融機関API接続
│   ├── persistence/ # JSON/DB永続化
│   └── repositories/ # リポジトリ実装
└── presentation/     # プレゼンテーション層
    └── controllers/
```

## データモデル

### 取引カテゴリ

5つの基本分類を必ず守る：

1. **収入**: 資産増加（給与、賞与、配当など）
2. **支出**: 資産減少（生活費、娯楽費など）
3. **振替**: 資産間移動（銀行→証券、クレカ引き落としなど）
4. **返済**: ローン・借入の返済
5. **投資**: 投資商品の購入・売却

### クレジットカード管理

- 支出計上: カード明細取得時に「支出」として記録
- 引き落とし: 銀行引き落としは「振替」として記録
- 月次突合: カード支払い額と銀行引き落とし額を比較
- ステータス管理: 未支払い/支払済を明確に区別

### データ永続化

- 初期: JSON形式で保存
- 将来: SQLiteまたはRDBMSへの移行を想定した設計

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

## 機能要件

### 金融機関API連携

- 各金融機関の接続設定をUI上で管理
- アプリ起動時にバックグラウンドで接続確認
- 接続失敗時はポップアップで通知
- APIクライアントは抽象化し、金融機関ごとに実装を切り替え可能に

### 分類・カテゴリ

- MoneyTree、MoneyForwardの分類を参考
- ユーザーによる分類の手動修正を可能に
- デフォルト分類ルールの学習機能（将来実装）

### 集計機能

- 月次集計: 金融機関別、分類別
- 期間指定での集計
- グラフ・チャートでの可視化

## セキュリティ

- 金融機関の認証情報は環境変数で管理
- APIキー、パスワードはGitにコミットしない
- 通信は必ずHTTPSを使用
- データは暗号化して保存（将来対応）

## テスト戦略

- ユニットテスト: 各レイヤごとに実装
- 統合テスト: API連携部分
- E2Eテスト: 重要なユーザーフロー
- テストカバレッジ: 80%以上を目標

## コミットメッセージ

コミットメッセージの規約については、[Commit Rules](./commit.md) を参照してください。

## 注意事項

- 金融データの取り扱いには細心の注意を払う
- パフォーマンスを考慮し、大量データの処理は非同期で実装
- エラーハンドリングは必ず実装し、ユーザーにわかりやすいメッセージを表示
- ログは詳細に記録し、トラブルシューティングを容易にする

## 推奨ライブラリ

- バリデーション: `zod` または `class-validator`
- 日付処理: `dayjs` または `date-fns`
- HTTP通信: `axios`
- 状態管理（frontend）: `zustand` または `jotai`
- UIコンポーネント: `shadcn/ui` または `Chakra UI`

## Issue管理

### Epic管理

#### Epic作成とSub-issue設定

**Epic Issue作成時のルール:**

- Epic Issueには必ず`epic`ラベルを付与する
- Issue番号は連番で管理（例: #181-196）
- GitHub Projectに追加し、ステータスを「🎯 Epic」に設定する

**Sub-issue関係の設定:**

- **GraphQL APIの`addSubIssue` mutationを使用してプログラム的に設定する**（推奨）

  ```bash
  # Issue Node IDを取得
  EPIC_ID=$(gh api graphql -f query='query { repository(owner: "kencom2400", name: "account-book") { issue(number: 182) { id } } }' | jq -r '.data.repository.issue.id')
  SUB_ISSUE_ID=$(gh api graphql -f query='query { repository(owner: "kencom2400", name: "account-book") { issue(number: 23) { id } } }' | jq -r '.data.repository.issue.id')

  # Sub-issue関係を設定
  gh api graphql -f query="mutation { addSubIssue(input: { issueId: \"$EPIC_ID\" subIssueId: \"$SUB_ISSUE_ID\" replaceParent: false }) { issue { subIssuesSummary { total completed } } } }"
  ```

- 利用可能なGraphQL mutations:
  - `addSubIssue`: Sub-issue関係を追加
  - `removeSubIssue`: Sub-issue関係を削除
  - `reprioritizeSubIssue`: Sub-issueの順序を変更
- 利用可能なフィールド:
  - `Issue.subIssues`: 子Issueの一覧
  - `Issue.parent`: 親Issue
  - `Issue.subIssuesSummary`: 進捗サマリー（total, completed, percentCompleted）

**注意事項:**

- 旧来の`trackedIssues` / `trackedInIssues`フィールドは非推奨
- Tasklist方式（Issue本文に`- [ ] #123`形式）は動作が不確定なため非推奨
- 手動UI設定も可能だが、自動化スクリプトを推奨

**参考スクリプト:**

- `scripts/github/setup-all-epic-subissues.sh`: 全Epic-Subissue関係の一括設定

### Issue作成時のルール

- **Issue作成時は必ずGitHub Projectsに追加する**
  - プロジェクト名: `Account Book Development`
  - コマンド例: `gh project item-add 1 --owner kencom2400 --url <issue_url>`
- **追加時のステータスは必ず「📋 Backlog」にする**
- 適切なラベルを付与する（bug、feature、enhancement等）
- テンプレートに従って詳細を記載する
- 関連するissueやPRがあれば明記する

### Issue取得時のルール

- **GitHub Projectsから取得する際は必ず「📋 To Do」ステータスから取得する**
  - コマンド例: `gh project item-list 1 --owner kencom2400 --format json | jq '.items[] | select(.status.name == "To Do")'`
- 作業を開始する際は、ステータスを「🏗 In Progress」に変更する
- 完了後は「✅ Done」に変更する

### Issue管理のベストプラクティス

- 1つのissueは1つの問題または機能に集中させる
- ただし、複数の関連する問題が同じ原因の場合はまとめて記載可能
- 優先度や影響範囲を明確に記載する
- 実装方針や修正方針を具体的に記述する

### GitHub Projects フィールド管理

#### ステータスフィールドの更新時の注意

**重要な制約:**

- `ProjectV2SingleSelectField`の`options`配列を更新すると、**すべてのオプションIDが再生成される**
- 既存のIssueが参照している古いオプションIDとの紐付けが切れる
- 結果として、すべてのIssueのステータスが「No Status」になる可能性がある

**対策:**

1. **新しいステータスオプションを追加する前にバックアップを取る**

   ```bash
   gh project item-list 1 --owner @me --format json --limit 200 > backup-$(date +%Y%m%d).json
   ```

2. **ステータス追加後は必ず既存Issueのステータスを確認**

   ```bash
   gh project item-list 1 --owner @me --format json | jq '.items[] | select(.status == null)'
   ```

3. **Issue TimelineからステータスHistoryを復元**
   - `ProjectV2ItemStatusChangedEvent`を利用して履歴を取得
   - 各Issueの最終ステータスを特定
   - 新しいステータスIDにマッピングして復元
   - 参考スクリプト: `scripts/github/restore-all-statuses-pagination.sh`

**復元手順:**

```bash
# 1. Issue Timelineからステータス履歴を取得
gh api graphql -f query='
query {
  repository(owner: "kencom2400", name: "account-book") {
    issue(number: 23) {
      timelineItems(last: 100) {
        nodes {
          __typename
          ... on ProjectV2ItemStatusChangedEvent {
            status
            createdAt
          }
        }
      }
    }
  }
}'

# 2. 最終ステータスを抽出
last_status=$(jq -r '[.data.repository.issue.timelineItems.nodes[] | select(.__typename == "ProjectV2ItemStatusChangedEvent")] | .[-1].status')

# 3. 新しいステータスIDにマッピング
case "$last_status" in
  "🎯 Epic") status_id="9aa232cf" ;;
  "📋 Backlog") status_id="f908f688" ;;
  "📝 To Do") status_id="f36fcf60" ;;
  "🚧 In Progress") status_id="16defd77" ;;
  "👀 Review") status_id="0f0f2f26" ;;
  "✅ Done") status_id="2f722d70" ;;
esac

# 4. ステータスを更新
gh api graphql -f query="mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: \"$PROJECT_ID\"
    itemId: \"$PROJECT_ITEM_ID\"
    fieldId: \"$STATUS_FIELD_ID\"
    value: { singleSelectOptionId: \"$status_id\" }
  }) {
    projectV2Item { id }
  }
}"
```

**参考ドキュメント:**

- `docs/project-status-restoration-report.md`: ステータス復元作業の詳細記録
- `docs/api-epic-investigation.md`: GitHub GraphQL API調査結果
- `docs/sub-issue-investigation-result.md`: Sub-issue設定の調査結果

## 開発フロー

1. 要件定義 → ドメインモデル設計
2. ドメイン層から実装開始（内側→外側）
3. ユースケース実装
4. インフラ層実装（API接続、データ永続化）
5. プレゼンテーション層実装
6. テスト実装
7. レビュー・リファクタリング
