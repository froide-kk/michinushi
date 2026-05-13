---
name: analyze-sentry
description: Sentry 分析の手順・Issue フォーマット・ステータスルール。triage Skill と analyst Agent が参照する。
requires-mcp:
  - name: sentry
    transport: http
    url: https://mcp.sentry.dev/mcp
---

# analyze-sentry — Sentry Issue 分析手順

## このSkillの位置づけ

analyze-sentry は参照 Skill である。自身が MCP ツールや gh コマンドを実行するのではなく、
以下の2つの場面で参照される:
- **triage Skill** がデータ取得する際の手順ガイド（Step 1〜4）
- **analyst Agent** が分析する際の分析基準・出力テンプレート（Step 5〜6）

## 前提条件

- Sentry MCP サーバーが設定・認証済みであること（`/triage` の Step 2.5 で確認される）
- `.claude/config/project.yml` に `sentry` セクションが定義されていること

---

## データ取得手順（triage が実行）

### Step 1: 設定の読み込み

`.claude/config/project.yml` から以下を取得:

```yaml
external-sources:
  sentry:
    organization: <org-slug>
    projects:
      - slug: <project-slug>
        directory: <対応ディレクトリ>  # 任意
        label: <scope ラベル名>       # 任意（未定義なら slug を使用）
```

### Step 2: 対象プロジェクトの決定

| 引数 | 動作 |
|------|------|
| `<project-slug>` | `sentry.projects` から slug が一致するものを選択 |
| `all`（デフォルト） | `sentry.projects` の全プロジェクトを対象 |
| `--query <search>` | 自然言語でフィルタ（`mcp__sentry__search_issues` の naturalLanguageQuery 使用） |

フォールバック:
1. `sentry.projects` が定義されている → その一覧を使用
2. `sentry.organization` のみ → `mcp__sentry__find_projects` で一覧取得し選択
3. `sentry` セクションなし → エラー（triage のソース検出で除外されるため通常到達しない）

### Step 3: Sentry Issue の取得

MCP ツールを使用:

1. `mcp__sentry__search_issues` で未解決 Issue を取得
   - `organizationSlug`, `projectSlugOrId`, `regionUrl` を指定
   - `naturalLanguageQuery`: `"all unresolved issues"` または引数のクエリ
2. 各 Issue に対して `mcp__sentry__get_sentry_resource` で詳細取得
   - `organizationSlug`, `resourceType: "issue"`, `resourceId: "<source-id>"` を指定

### Step 4: 既存 GitHub Issue との重複チェック

```bash
gh issue list --repo <owner>/<repo> --label source:sentry --state open --json title,body,number
```

Sentry Issue ID（SHORT_ID）が既存 Issue の body に含まれていれば重複とみなしスキップする。

---

## 分析手順（analyst が実行）

### Step 5: 分析

triage から渡されたデータをもとに、各 Issue について以下を分析:

1. **根本原因の推定** — スタックトレース、イベントデータから原因を推定
2. **直近変更との相関** — Sentry の `release` タグ（コミットハッシュ）から、エラーが発生したリリースを特定する。triage が渡すデータに `release` が含まれるので、そのコミット前後の変更を分析に活用する
3. **影響範囲** — 影響を受けるファイル・機能を特定（`directory` マッピングがあれば活用）
4. **再現手順** — イベントデータから再現条件を推定
5. **修正方針** — 推奨する修正アプローチを提案

優先度の判定基準:

| 優先度 | 条件 |
|--------|------|
| critical | ユーザー影響大、発生頻度高、クラッシュ |
| high | 頻度高 or 主要機能に影響 |
| medium | 散発的、限定的な影響 |
| low | エッジケース、軽微 |

### Step 6: 分析結果の出力

analyst は以下の情報を含む構造化された結果を返す。

出力フィールド（Issue ごと）:
- `source-id`: Sentry の SHORT_ID
- `source-url`: Sentry Issue の URL
- `project-slug`: 対象プロジェクト
- `title`: エラータイトル
- `severity`: critical / high / medium / low
- `first-seen`, `last-seen`, `count`, `user-count`
- `root-cause`: 根本原因の推定
- `affected-files`: 影響を受けるファイル・機能
- `reproduction-steps`: 再現手順（推定）
- `recommended-fix`: 修正方針
- `duplicate-of`: 重複する GitHub Issue 番号（triage が重複情報を渡した場合）

---

## テンプレート（triage が Issue 作成時に使用）

### ラベル

| ラベル | 用途 |
|--------|------|
| `bug` | Issue 種別 |
| `source:sentry` | Sentry 起因であることを示す（固定） |
| `scope:<label>` | 対象プロジェクト（`sentry.projects[].label` または slug） |

### GitHub Issue body テンプレート

```markdown
## Sentry Issue

| 項目 | 値 |
|------|-----|
| Issue | [<source-id>](<sentry_url>) |
| Project | `<project_slug>` |
| 初回発生 | <first_seen> |
| 最終発生 | <last_seen> |
| 発生回数 | <count> |
| 影響ユーザー | <user_count> |

## エラー分析

### 根本原因（推定）
<root_cause_analysis>

### 影響範囲
- ファイル: `<file_path>`
- 機能: <affected_feature>

### 再現手順（推定）
1. <step>

## 修正方針

<recommended_fix>

### 関連ファイル
- `<file_path>:<line>`

## メタデータ
- Sentry ID: `<source-id>`
```

## データサニタイゼーション

Issue body に記載する前に、以下のデータをマスクまたは除去する:
- ユーザー識別情報（メールアドレス、ユーザーID、IPアドレス）
- リクエストボディに含まれる可能性のある個人データ
- セッショントークン、API キー等のシークレット
- 子供の入力データ（感情表現テキスト等）がスタックトレースの変数に含まれる場合

analyst は分析時にこれらを参照できるが、出力の body-template には含めない。
