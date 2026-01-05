# データアクセスと配列操作

**優先度レベル**: `02-XX` - **優先（SHOULD）** - できる限り守るべきルール

## 2. データアクセスと配列操作

### ❌ 避けるべきパターン

#### 2-1. 配列の順序依存

```typescript
// ❌ 配列の順序に暗黙的に依存
const results = await checkMultiple(institutions);
const histories = results.map((result, index) => {
  const institution = institutions[index]; // 危険！
});
```

**✅ 正しい対応**:

```typescript
// ✅ IDベースでマッピング
const institutionMap = new Map(institutions.map((inst) => [inst.id, inst]));

const histories = results
  .map((result) => {
    const institution = institutionMap.get(result.institutionId);
    if (!institution) {
      logger.warn(`対応する金融機関が見つかりません: ${result.institutionId}`);
      return null;
    }
    return createHistory(result, institution);
  })
  .filter((h): h is History => h !== null);
```

#### 2-2. クエリパラメータの網羅性不足

```typescript
// ❌ 一部のパラメータ組み合わせのみ対応
if (query.institutionId) {
  return getLatest(query.institutionId); // limitが無視される！
}
```

**✅ 正しい対応**:

```typescript
// ✅ 全てのパラメータ組み合わせを考慮
if (query.latestOnly) {
  // 最新のみ
} else if (query.startDate && query.endDate) {
  histories = await getByDateRange(...);
  if (query.limit) {
    histories = histories.slice(0, query.limit);
  }
} else if (query.institutionId) {
  histories = await getAll().filter(h => h.id === query.institutionId);
  if (query.limit) {
    histories = histories.slice(0, query.limit);
  }
}
```

---

## 2-3. 環境変数の管理（NestJSベストプラクティス）

### ❌ 避けるべきパターン: `process.env`の直接参照

```typescript
// ❌ 悪い例: process.envを直接参照
@Injectable()
export class ConnectionCheckerService {
  private readonly TIMEOUT_MS = parseInt(process.env.HEALTH_CHECK_TIMEOUT_MS || '10000', 10);
  private readonly MAX_RESPONSE_TIME_MS = parseInt(
    process.env.HEALTH_CHECK_MAX_RESPONSE_TIME_MS || '5000',
    10
  );
}
```

**問題**:

- テスト容易性が低い（`process.env`を直接操作する必要がある）
- 型安全性がない（`parseInt`が`NaN`を返すリスク）
- バリデーションが実行時まで遅延される
- 設定の一元管理が困難

### ✅ 正しいパターン: `ConfigService`の使用

```typescript
// ✅ 良い例: ConfigServiceを使用
import { ConfigService } from '@nestjs/config';

@Injectable()
export class ConnectionCheckerService {
  private readonly TIMEOUT_MS: number;
  private readonly MAX_RESPONSE_TIME_MS: number;

  constructor(private readonly configService: ConfigService) {
    this.TIMEOUT_MS = this.configService.get<number>(
      'HEALTH_CHECK_TIMEOUT_MS',
      10000 // デフォルト値
    );
    this.MAX_RESPONSE_TIME_MS = this.configService.get<number>(
      'HEALTH_CHECK_MAX_RESPONSE_TIME_MS',
      5000 // デフォルト値
    );
  }
}
```

**改善点**:

1. **テスト容易性の向上**
   - `ConfigService`をモックすることで、テスト時に設定値を簡単に注入できる
   - `process.env`を直接操作する必要がない

2. **型安全性の向上**
   - `ConfigService`のジェネリクス型パラメータで型を指定
   - `class-validator`と連携することで、起動時にバリデーション可能

3. **設定の一元管理**
   - アプリケーション全体の設定を`ConfigModule`で一元管理
   - 可読性と保守性が向上

4. **実行時エラーの防止**
   - `parseInt`が`NaN`を返すリスクを低減
   - アプリケーション起動時に環境変数の型チェックと存在確認が可能

### 推奨実装パターン

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // 全モジュールでConfigServiceを使用可能に
      validationSchema: Joi.object({
        HEALTH_CHECK_TIMEOUT_MS: Joi.number().default(10000),
        HEALTH_CHECK_MAX_RESPONSE_TIME_MS: Joi.number().default(5000),
        SYNC_MAX_PARALLEL: Joi.number().default(5),
      }),
    }),
  ],
})
export class AppModule {}
```

**バリデーションスキーマ（オプション）**:

`class-validator`を使用することで、より厳密な型チェックとバリデーションが可能です：

```typescript
import { IsNumber, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class EnvironmentVariables {
  @Type(() => Number)
  @IsNumber()
  @Min(1000)
  @Max(60000)
  HEALTH_CHECK_TIMEOUT_MS: number = 10000;

  @Type(() => Number)
  @IsNumber()
  @Min(1000)
  @Max(30000)
  HEALTH_CHECK_MAX_RESPONSE_TIME_MS: number = 5000;

  @Type(() => Number)
  @IsNumber()
  @Min(1)
  @Max(20)
  HEALTH_CHECK_MAX_PARALLEL: number = 5;
}
```

### チェックリスト

- [ ] `process.env`を直接参照せず、`ConfigService`を使用している
- [ ] 環境変数の型が明示的に指定されている
- [ ] デフォルト値が適切に設定されている
- [ ] テストでは`ConfigService`をモックしている
- [ ] （オプション）バリデーションスキーマを定義している

**参考**: PR #282 - Gemini Code Assistレビュー指摘
