---
name: test-playwright
description: playwright を使った E2E テストの記法とパターン。ブラウザ自動化による業務フローテスト、ビジュアルリグレッションを担当する tester Agent が使う。
user-invocable: false
---

# Skill: test-playwright

## 目的

playwright 固有の記法・セレクタ戦略・実行方法をまとめた参照 Skill。
テストの「何を書くか」は `test-design-principles` Skill を参照する。

## ファイル配置

プロジェクトの慣習に従う。一般的には:

- `e2e/<feature>.spec.ts`
- `tests/e2e/<feature>.spec.ts`

## 基本構造

```typescript
import { test, expect } from '@playwright/test';

test.describe('feature name', () => {
  test.beforeEach(async ({ page }) => {
    // 共通セットアップ（ログイン等）
    await page.goto('/login');
    await page.fill('[name=email]', 'test@example.com');
    await page.fill('[name=password]', 'password');
    await page.click('button[type=submit]');
  });

  test('user can complete the workflow', async ({ page }) => {
    await page.goto('/some-path');
    await expect(page.locator('h1')).toHaveText('Expected Title');
    // ...
  });
});
```

## セレクタ戦略

優先順:

1. **role + accessible name** — `page.getByRole('button', { name: '保存' })`
2. **label** — `page.getByLabel('メールアドレス')`
3. **placeholder** — `page.getByPlaceholder('email@example.com')`
4. **text** — `page.getByText('注文を確定する')`
5. **test-id** — `page.getByTestId('submit-button')`（最終手段）

CSS セレクタ・XPath は壊れやすいので避ける。

## 待機

playwright は自動待機があるが、明示が必要な場合:

```typescript
// 要素が表示されるまで
await page.waitForSelector('[data-loaded]');

// URL 遷移を待つ
await page.waitForURL('/dashboard');

// レスポンスを待つ
await page.waitForResponse(resp => resp.url().includes('/api/data'));

// 非表示まで（ローディング消えるまで等）
await page.waitForSelector('.spinner', { state: 'hidden' });
```

## アサーション

```typescript
await expect(page.locator('h1')).toHaveText('期待値');
await expect(page.locator('input')).toHaveValue('val');
await expect(page.locator('.error')).toBeVisible();
await expect(page.locator('.spinner')).toBeHidden();
await expect(page.locator('button')).toBeEnabled();
await expect(page.locator('button')).toBeDisabled();
await expect(page).toHaveURL(/\/dashboard/);
await expect(page).toHaveTitle('期待タイトル');

// 件数
await expect(page.locator('.item')).toHaveCount(5);
```

## ビジュアルリグレッション

```typescript
// ページ全体
await expect(page).toHaveScreenshot('home.png');

// 特定要素
await expect(page.locator('.card')).toHaveScreenshot('card.png');

// オプション
await expect(page).toHaveScreenshot({
  fullPage: true,
  mask: [page.locator('.timestamp')],  // 動的部分はマスク
});
```

初回実行で baseline が生成される。差分検出時は `--update-snapshots` で更新。

## 安定性のコツ

Flaky なテストを避けるための原則:

- **固定待機（`waitForTimeout`）を使わない** — 状態待機（`waitForSelector` 等）を使う
- **アニメーション中のスクリーンショットを避ける** — `animations: 'disabled'` オプション or 明示待機
- **テストごとに独立したデータを使う** — 他テストの影響を受けない
- **テスト前にセッションをリセット** — `context.clearCookies()` 等

## 実行

```bash
# 全テスト
npx playwright test

# 特定ファイル
npx playwright test e2e/login.spec.ts

# UI モード（デバッグ向け）
npx playwright test --ui

# ヘッドフル（ブラウザ表示）
npx playwright test --headed

# トレース付き（失敗解析用）
npx playwright test --trace on

# レポート表示
npx playwright show-report
```

## 失敗時の診断

playwright は失敗時に自動でスクリーンショット・トレースを保存する設定にしておく:

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
    video: 'retain-on-failure',
  },
});
```

失敗報告時はこれらの artifacts のパスを添付する。
