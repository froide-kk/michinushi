---
name: test-vitest
description: vitest を使ったユニット・インテグレーションテストの記法とパターン。TypeScript/JavaScript プロジェクトで vitest を採用している場合に tester Agent が使う。
user-invocable: false
---

# Skill: test-vitest

## 目的

vitest 固有の記法・モック・実行方法をまとめた参照 Skill。
テストの「何を書くか」は `test-design-principles` Skill を参照する。

## ファイル配置

プロジェクトのテスト配置パターンを踏襲する。一般的には:

- `<source>.test.ts` — 同階層
- `__tests__/<source>.test.ts` — 専用ディレクトリ

## 基本構造

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';

describe('functionName', () => {
  beforeEach(() => {
    // セットアップ（必要に応じて）
  });

  it('does the expected thing', async () => {
    // Arrange
    const input = { ... };

    // Act
    const result = await functionName(input);

    // Assert
    expect(result).toEqual({ ... });
  });
});
```

## モック

### 関数のモック

```typescript
import { vi } from 'vitest';

const mockFn = vi.fn();
mockFn.mockReturnValue(value);
mockFn.mockResolvedValue(value);  // async
mockFn.mockRejectedValue(error);  // async error
```

### モジュールのモック

```typescript
vi.mock('@/lib/db', () => ({
  db: { user: { findUnique: vi.fn() } },
}));
```

### モックのリセット

```typescript
import { resetAllMocks } from './helpers';

beforeEach(() => {
  resetAllMocks();
});
```

## Server Action テストの定型

Next.js Server Action のテスト構成例:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { mockPrisma, mockCurrentUser, testStoreOwner, resetAllMocks } from './helpers';
import { actionName } from './actionFile';

describe('actionName', () => {
  beforeEach(() => resetAllMocks());

  it('does the expected thing', async () => {
    mockCurrentUser(testStoreOwner);
    mockPrisma.entity.findUnique.mockResolvedValue({ ... });

    const result = await actionName({ ... });

    expect(result).toEqual({ ... });
  });

  it('returns error for invalid input', async () => { ... });
  it('returns error for non-owner', async () => { ... });
  it('returns error when record not found', async () => { ... });
  it('returns error for other store data', async () => { ... });
  it('throws when not authenticated', async () => { ... });
});
```

## アサーション一覧

よく使うもの:

```typescript
expect(value).toBe(expected);            // ===
expect(value).toEqual(expected);          // deep equal
expect(value).toContain(item);
expect(value).toHaveLength(n);
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toMatch(/regex/);
expect(value).toThrow();
expect(value).toThrow('message');

// Promise
await expect(promise).resolves.toEqual(value);
await expect(promise).rejects.toThrow();

// Mock
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(n);
expect(mockFn).toHaveBeenCalledWith(args);
expect(mockFn).toHaveBeenLastCalledWith(args);
```

## テストデータ規約

- UUID: `00000000-0000-4000-8000-000000000001` 形式（Zod の uuid バリデーションを通る形式）
- ID 系の固定値はヘルパーファイル（`helpers.ts`）で定義し、再利用する

## 実行

```bash
# 全テスト実行
pnpm test          # またはプロジェクトの test スクリプト
npx vitest run

# 特定ファイル
npx vitest run path/to/file.test.ts

# watch モード
npx vitest

# カバレッジ
npx vitest run --coverage
```

## トラブルシューティング

- **モックが効かない**: `vi.mock()` はファイルトップで宣言する必要がある（ホイストされる）
- **タイマー系テストが flaky**: `vi.useFakeTimers()` で時刻を固定する
- **非同期処理が完了しない**: `await` の付け忘れがないか確認
