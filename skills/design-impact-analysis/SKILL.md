---
name: design-impact-analysis
description: 影響範囲分析の手順と出力フォーマット。変更がコードベース・設計書・テストにどう波及するかを系統的に洗い出すための参照材料。architect Agent が使う。
user-invocable: false
---

# Skill: design-impact-analysis

## 目的

要件・変更に対して、影響を受ける対象を漏れなく特定する。

## 分析手順

### 1. 直接影響の特定

要件から直接変更される対象を列挙する:

- **データモデル**: 追加・変更・削除されるエンティティ・フィールド
- **ビジネスロジック**: 影響する Server Action / API / ドメイン関数
- **UI**: 影響する画面・コンポーネント
- **設計書**: 更新が必要な YAML / ADR
- **テスト**: 影響する既存テスト

### 2. 連鎖影響の追跡

直接影響対象から伝播する影響を辿る:

- データモデル変更 → それを使う Server Action / Component
- Server Action 変更 → それを呼ぶ UI / 他 Action
- UI 変更 → ナビゲーション / 共通レイアウト

### 3. 過去判断との整合性確認

影響する設計書の `metadata.relatedDecisions` を確認:

- ADR 番号があれば該当ファイル（例: `docs/adr/ADR-1.md`）を読む
- 今回の変更が過去判断に影響する場合、警告を上げる

### 4. 横断関心事の確認

以下が影響を受けるかチェック:

- マイグレーション要否
- 認証・認可（権限定義の変更）
- 監査ログ・通知
- 環境変数・設定ファイル
- CI/CD パイプライン

## 出力フォーマット

```yaml
affected:
  models:
    - name: <モデル名>
      change: added | modified | removed
      details: <変更内容>
  actions:
    - path: <ファイルパス>
      change: added | modified | removed
  components:
    - path: <ファイルパス>
      change: added | modified | removed
  docs:
    - path: <ファイルパス>
      change: added | modified | removed
      reason: <更新理由>
  tests:
    - path: <ファイルパス>
      change: added | modified | removed
  migrations:
    required: true | false
    description: <マイグレーション内容>
  cross-cutting:
    auth: <影響内容 or none>
    audit: <影響内容 or none>
    config: <影響内容 or none>

related-decisions:
  - adr: ADR-<番号>
    impact: <この変更が判断に与える影響>
    action-required: <必要な対応>
```

## 注意

- **不確実な部分は推測ではなく確認** — コードを読んで確認できることは確認する
- **網羅性 > 詳細度** — 各項目は短くて良いので漏れなく挙げる
- **連鎖の打ち切り** — 影響が薄くなったら追跡を打ち切る（過剰調査を避ける）
