# E2E Tester Skill 設計書

## 概要

E2Eテスト・インテグレーションテスト・ビジュアルリグレッションテストを担う専門Skill。
ユーザーの業務フロー全体をブラウザ上で再現し、システム全体が正しく動作することを検証する。

## ペルソナ: QAエンジニア / テスト自動化エンジニア

ユーザーの業務フローを理解し、実際の操作シナリオでテストを設計する。
「関数が正しい」ではなく「ユーザーが業務を完了できる」ことを検証する。

### 行動原則

| 原則 | 具体的な行動 |
|------|-------------|
| **ユーザー視点** | API単位ではなく業務フロー単位でテストを設計 |
| **安定性重視** | Flakyなテストを作らない。待機・リトライを適切に設計 |
| **独立性** | テスト間でデータを共有しない。各テストがセットアップから行う |
| **設計書駆動** | ユースケース・業務フローからテストシナリオを導出 |
| **失敗時の診断性** | スクリーンショット・ログで失敗原因が即座に分かるようにする |

## 責務

### やること
- E2Eテストの設計・実装（Playwright）
- インテグレーションテストの設計・実装（実DB接続）
- ビジュアルリグレッションテスト
- テスト環境のセットアップ（Docker Compose）
- テスト実行・結果分析・スクリーンショット管理

### やらないこと
- ユニットテスト（→ unit-tester）
- 実装コードの修正（→ coder）
- テスト戦略の策定（→ architect）

## テスト種別

### 1. E2Eテスト（Playwright）

ブラウザを操作してユーザーの業務フローを再現する。

```
ツール: Playwright
環境: Docker Compose（Next.js + PostgreSQL + Firebase Emulator）
ブラウザ: Chromium（ヘッドレス）
```

#### テストシナリオ例

**アンケート〜本登録フロー:**
```
1. スタッフがアンケートURL発行
2. 見込みがアンケートに回答（5ステップ全て）
3. 見込み一覧に表示されることを確認
4. 見込み詳細でアンケート回答を確認
5. 本登録ボタンを押下
6. お客様一覧に表示されることを確認
7. カルテにヒアリングデータが引き継がれていることを確認
```

**カルテ編集フロー:**
```
1. お客様詳細 > カルテタブを開く
2. 編集ボタンをクリック
3. ヒアリング内容を編集
4. 初回測定値を編集
5. 保存ボタンをクリック
6. 閲覧モードに戻り、変更が反映されていることを確認
7. 再度編集 > キャンセル > 変更が破棄されていることを確認
```

**権限制御:**
```
1. STORE_OWNERでログイン → スタッフ管理画面にアクセス可能
2. STAFFでログイン → スタッフ管理画面にアクセス不可
3. STAFFでログイン → 自分の施術記録のみ編集可能
```

### 2. インテグレーションテスト（実DB）

Server Action + Prisma + PostgreSQL の一気通貫テスト。

```
ツール: Vitest + 実Prisma Client
環境: テスト用PostgreSQL（Docker / tmpfs で高速化）
```

#### テスト対象
- トランザクション: 見込み→本登録の変換（customer作成 + medicalRecord作成 + prospect更新 + survey更新）
- 制約: unique制約違反時のエラーハンドリング
- カスケード: customer削除時の関連データ削除
- JSONB: counselingData/initialMeasurementの保存・読み込み
- マイグレーション: 全マイグレーションが空DBに正常適用されるか

### 3. ビジュアルリグレッションテスト

UIの意図しない変更を検出する。

```
ツール: Playwright のスクリーンショット比較
対象: 主要画面のスナップショット
```

#### 対象画面
- ログイン画面
- お客様一覧
- お客様詳細（各タブ）
- 見込み一覧・詳細
- アンケートフォーム（各ステップ）
- 設定画面

## テスト環境

### Docker Compose 構成
```yaml
# docker-compose.test.yml（将来作成）
services:
  db-test:
    image: postgres:15
    environment:
      POSTGRES_DB: humanlink_karte_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    tmpfs: /var/lib/postgresql/data  # メモリ上で高速化

  firebase-test:
    # Firebase Auth Emulator
    ports: ["9099:9099"]

  store-test:
    # Next.js store app
    depends_on: [db-test, firebase-test]
    environment:
      DATABASE_URL: postgresql://test:test@db-test:5432/humanlink_karte_test
      FIREBASE_AUTH_EMULATOR_HOST: firebase-test:9099
```

### テストデータ
- E2E: テストごとにAPIでデータをセットアップ（seed.tsは使わない）
- インテグレーション: テストごとにトランザクションでロールバック or truncate

## CI実行戦略

| テスト種別 | タイミング | 所要時間目安 |
|-----------|-----------|------------|
| E2E | developマージ時 / 手動実行 | 3〜5分 |
| インテグレーション | developマージ時 | 1〜2分 |
| ビジュアルリグレッション | 手動実行 / リリース前 | 2〜3分 |

### GitHub Actions ジョブ構成（将来）
```yaml
e2e-test:
  needs: [build-images]
  services:
    postgres: ...
    firebase: ...
  steps:
    - npx playwright install
    - npx playwright test
    - Upload screenshots as artifacts (on failure)
```

## Conductorとの連携インターフェース

### 入力
```
type: 'create-e2e' | 'create-integration' | 'run-e2e' | 'run-integration' | 'visual-regression'
target:
  feature: 対象機能名
  scenarios: [テストシナリオ（architect/設計書から）]
```

### 出力
```
status: 'all-passed' | 'some-failed'
results:
  total, passed, failed
  failureDetails:
    - scenario, step, screenshot, error
  screenshots: [スクリーンショットパス]
artifacts:
  - trace files (Playwright trace)
  - video (失敗時)
```

## 検討事項

### 1. Playwright の導入タイミング
- Phase 4 で導入予定
- 先に docker-compose.test.yml とテスト用seedの設計が必要
- architect が環境設計 → e2e-tester がテスト実装の流れ

### 2. テストアカウント管理
- Firebase Emulator のテストユーザー（seed.ts で定義済み）
- E2Eテスト時はEmulatorに直接ユーザーを作成
- 本番Firebase は絶対に使わない

### 3. スクリーンショットのベースライン管理
- git にコミット？外部ストレージ？
- → git LFS or 専用ディレクトリ（.gitignore で制御）
- 更新時はPRで差分をレビュー
