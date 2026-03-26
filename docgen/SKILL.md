---
name: docgen
description: JSON Schema準拠のYAML設計書を生成・更新する。設計書の作成、スキーマバリデーションに関するタスクで使用。Use when creating or updating design documents, YAML specs, or schema validation.
user-invocable: false
license: MIT
compatibility: "Requires ajv-cli for schema validation (npm install -g ajv-cli ajv-formats)"
---

# 設計書ジェネレーター

JSON スキーマに準拠した YAML 設計書を生成する。

スキーマディレクトリ: `docs/schemas/`（プロジェクトルートからの相対パス）

## 生成手順

### Step 1: 対象フェーズの特定

ユーザーの要求から、以下のどのフェーズの設計書が必要かを特定する。

| フェーズ | スキーマパス | 必須/任意 |
|----------|------------|-----------|
| 要件定義 | `docs/schemas/requirements_definition_schema.json` | 必須 |
| ペルソナ定義 | `docs/schemas/persona_schema.json` | 任意 |
| ユーザージャーニー | `docs/schemas/user_journey_schema.json` | 任意 |
| 業務フロー | `docs/schemas/business_process_flow_schema.json` | 任意 |
| ドメインモデル | `docs/schemas/domain_model_schema.json` | 必須 |
| システム構成 | `docs/schemas/system_architecture_schema.json` | 必須 |
| DB設計（テーブル定義） | `docs/schemas/database_design_schema.json` | 必須 |
| DB設計（全体概要） | `docs/schemas/database_overview_schema.json` | 必須 |
| API仕様 | `docs/schemas/api_specification_schema.json` | 必須 |
| コンポーネント設計 | `docs/schemas/component_design_schema.json` | 必須 |
| レイアウト設計 | `docs/schemas/layout_template_schema.json` | 必須 |
| 画面設計 | `docs/schemas/screen_design_schema.json` | 必須 |
| 通知設計 | `docs/schemas/notification_schema.json` | 任意 |
| 機能一覧 | `docs/schemas/feature_list_schema.json` | 必須 |
| 機能設計 | `docs/schemas/feature_design_schema.json` | 必須 |
| 認証フロー | `docs/schemas/auth_flow_schema.json` | 任意 |
| バッチ・イベント処理 | `docs/schemas/batch_event_process_schema.json` | 任意 |
| 環境定義 | `docs/schemas/environment_definition_schema.json` | 任意 |
| テスト計画 | `docs/schemas/test_plan_schema.json` | 必須 |
| テスト仕様 | `docs/schemas/test_specification_schema.json` | 必須 |
| 共通メタデータ | `docs/schemas/common_metadata_schema.json` | （全スキーマから参照） |

### Step 2: 依存関係の確認

設計書には前工程からの依存がある。以下の順序を厳守すること。

```
要件定義
├── ドメインモデル ← 要件定義 + ペルソナ + ユーザージャーニー
├── システム構成 ← 要件定義
├── DB設計 ← ドメインモデル
├── API仕様 ← ドメインモデル + システム構成
├── コンポーネント設計 ← システム構成 + API仕様
├── 機能一覧 ← 要件定義
├── レイアウト設計 ← システム構成
├── 画面設計 ← ドメインモデル + システム構成 + 機能一覧 + レイアウト設計
├── 通知設計 ← 要件定義 + システム構成
├── 機能設計 ← 上流設計書すべて
├── テスト計画 ← 要件定義 + 機能設計
└── テスト仕様 ← テスト計画 + 機能設計
```

上流の設計書が未作成の場合、先にそちらの生成を促すこと。途中から生成する場合は、既存の設計書を必ず読み込んでから着手する。

### Step 3: スキーマの読み込みと設計書生成

1. `docs/schemas/` から対象スキーマを読み込み、`required` フィールドと構造を把握する
2. `docs/schemas/common_metadata_schema.json` を読み込み、メタデータ構造を把握する
3. ユーザーの要求内容をスキーマの各フィールドにマッピングする
4. 先頭に `$schema` リンクを付与した YAML を生成する

出力テンプレート:

```yaml
$schema: docs/schemas/<schema_name>.json

metadata:
  version: "1.0.0"
  createdAt: "<ISO 8601形式>"
  updatedAt: "<ISO 8601形式>"

# 以降、スキーマの properties に定義された全フィールドを記述（required / optional 問わず）
```

### Step 4: セルフチェック

生成した YAML を以下の観点で検証する。

- [ ] `required` フィールドがすべて含まれているか
- [ ] `description` の記載に曖昧な表現（「〜など」「適宜」「必要に応じて」）がないか
- [ ] ID の命名規則が一貫しているか（例: `FR-001`, `API-USER-001`）
- [ ] 上流設計書との ID 参照が整合しているか
- [ ] Mermaid 図がある場合、構文が正しいか

### Step 5: バリデーション

```bash
ajv validate -s docs/schemas/<schema_name>.json -r docs/schemas/common_metadata_schema.json -d <output_file>.yaml
```

バリデーションエラーが出た場合は、エラー内容を確認し修正してから再実行する。

## 出力規約

- ファイル名（単一）: `<schema_name から _schema を除去>_<プロジェクト名>.yaml`（例: `requirements_definition_classroom.yaml`）
- ファイル名（分割）: `<schema_name から _schema を除去>_<プロジェクト名>_<識別子>.yaml`（例: `feature_design_classroom_user-registration.yaml`）
- エンコーディング: UTF-8
- インデント: スペース2つ
- 文字列値に `:` や `#` を含む場合はクォートで囲む
- 配列要素が1つでも配列記法（`- item`）を使用する

## エッジケース

- **複数スキーマを一度に生成する場合**: Step 2 の依存順序を厳守し、上流から順に生成する。並行生成はしない
- **既存設計書の更新**: 既存ファイルを読み込み差分のみ更新する。`metadata.updatedAt` と `metadata.history` を必ず更新する
- **要件が不明確な場合**: 想定で埋めず、ユーザーに具体的な質問を行う。「〜でよいですか？」ではなく「〜は A / B / C のどれですか？」のように選択肢を提示する
- **スキーマに存在しないフィールドを追加したい場合**: スキーマ側の拡張を先に検討する。YAML に独自フィールドを追加するとバリデーションが失敗する
