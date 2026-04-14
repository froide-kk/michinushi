# Triage — 設計判断の記録

## 概要

外部監視サービスのエラーを分析し、GitHub Issue として起票するワークフロー。
手順の詳細は `triage/SKILL.md`、`analyst/AGENT.md`、`analyze-*/SKILL.md` を参照。

## コマンド体系における位置づけ

```
/setup      セットアップ（MCP 追加はオプション）
/triage     外部ソース分析 → Issue 作成（本仕様）
/todo       タスク一覧報告
/run #33    既存 Issue を実装 → PR 作成
```

`/triage` は Issue を**作る側**、`/run` は Issue を**実装する側**。入出力が異なるため別コマンドとした。

## アーキテクチャ

### 3層構造

```
/triage (Skill)          ← ソース非依存。MCP データ取得 + gh 操作を担当
  └→ analyst (Agent)     ← ソース非依存。渡されたデータを分析（MCP/gh 不使用）
       └→ analyze-* (Skill) ← ソース固有の分析手順を参照
```

### 責務分離

| コンポーネント | やること | やらないこと |
|--------------|---------|------------|
| `/triage` | ソース検出、MCP データ取得、重複チェック、analyst 起動、承認取得、gh issue create、Project 追加 | エラー分析（analyst に委譲） |
| `analyst` | 渡されたデータの分析、根本原因推定、修正方針提案、結果の構造化 | MCP 呼び出し、gh 操作、Issue 作成 |
| `analyze-*` | ソース固有の分析手順・body テンプレートの提供 | データ取得、gh 操作 |

### MCP 制約と設計判断

**制約**: Claude Code の subagent（Agent ツール）は隔離コンテキストで動作し、MCP 接続を継承しない。

**判断**: triage（メインコンテキスト）がデータ取得、analyst（隔離コンテキスト）が分析。
これは `/run` の Conductor パターン（Conductor が gh 操作、Agent は分析のみ）と同じ構造。

## 設計原則

- **ソース非依存**: `triage` / `analyst` は特定サービスのロジックを持たない
- **自動検出**: `project.yml` のキー × `analyze-*` Skill の存在で利用可能なソースを判定
- **オプション**: 外部連携は必須ではない。未設定時はフォールバック案内
- **Conductor パターン**: 分析は Agent に委譲、外部操作（MCP/gh）は Skill 自身が行う

## 設定・認証フロー

1. `/setup` が `analyze-*` Skill の `requires-mcp` を検出し、オプションとして MCP 追加を提案
2. MCP 追加後はセッション再起動が必要（MCP ツールの読み込みのため）
3. `/triage` 実行時に `.mcp.json` のファイルチェック → MCP ツールの認証状態を確認
4. `.mcp.json` のチェックを最初に行うことで、MCP が未設定なのに接続が残っている状態を排除

## Sentry → GitHub Issue クローズ連携

Sentry の GitHub Integration を利用。PR に `Fixes <SENTRY_ID>` を含めると、マージ時に Sentry Issue が自動 resolve される。

- `run/github-ops.md` の PR 作成フローで、`source:sentry` ラベルの Issue のみ body から Sentry ID を抽出して付与
- Issue body に Sentry ID が `analyze-sentry` の body テンプレートに従って記載されていることが前提

## 拡張方法

新しい外部ソースを追加する場合:

1. `project.yml` の `external-sources` にソースのセクションを追加（接続情報）
2. `.claude/skills/analyze-<source>/SKILL.md` を作成（分析手順）
3. MCP が必要なら `requires-mcp` を frontmatter に記載
4. `/setup` で MCP 追加 → 再起動
5. `/triage` が自動検出

### 想定される拡張例

| カテゴリ | ソース | analyze-* | 用途 |
|---------|--------|-----------|------|
| エラー監視 | Datadog | `analyze-datadog` | APM エラー・ログ異常の分析 |
| エラー監視 | Crashlytics | `analyze-crashlytics` | モバイルアプリのクラッシュ分析 |
| エラー監視 | CloudWatch | `analyze-cloudwatch` | AWS インフラ・Lambda エラー分析 |
| フィードバック | App Store / Google Play | `analyze-app-reviews` | ユーザーレビューからバグ・要望を抽出 |
| フィードバック | Intercom / Zendesk | `analyze-support` | サポートチケットからバグパターンを分析 |
| セキュリティ | Dependabot / Snyk | `analyze-vulnerabilities` | 脆弱性アラートの分析・優先度付け |
| CI/CD | GitHub Actions | `analyze-ci-failures` | CI 失敗パターン・フレーキーテスト検出 |

### プロジェクト特性に応じた例

| プロジェクト特性 | ソース | 用途 |
|----------------|--------|------|
| Firebase 利用 | Crashlytics / Performance | クラッシュ分析、パフォーマンス劣化検出 |
| モバイルアプリ | App Store / Google Play | ユーザーレビューからバグ・UX 問題を抽出 |
| SaaS / B2B | Intercom / Zendesk | サポートチケットのパターン分析 |
| AWS インフラ | CloudWatch | Lambda エラー・インフラ異常の検出 |

### MCP の有無

MCP が提供されているサービスは MCP 経由、それ以外は REST API や CLI で対応する。
`requires-mcp` を持たない analyze-* Skill の場合、triage の Step 2.5 MCP チェックはスキップされる。
