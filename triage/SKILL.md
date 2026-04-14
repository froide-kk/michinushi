---
name: triage
description: 外部監視サービスのエラーを分析し、GitHub Issue を作成・Project に追加する。analyst Agent に分析を委譲し、結果をもとに Issue 化・ステータス設定を行う。ソース非依存で、project.yml の設定から対象を自動検出する。
user-invocable: true
argument-hint: "[<source>|all] [<project-slug>] [--query <search>]"
disable-model-invocation: true
---

# /triage — 外部ソース分析 & Issue 化

外部監視サービスのエラーを分析し、GitHub Issue として起票する。
分析は analyst Agent に委譲し、GitHub 操作は triage 自身が行う。

## 行動原則

1. **分析は analyst に委譲** — 自分ではエラーを分析しない
2. **GitHub 操作は自分で行う** — Issue 作成・Project 追加・ステータス設定
3. **ユーザー承認を挟む** — 分析結果を提示し、Issue 化の承認を得てから作成
4. **ソース非依存** — 特定サービスのロジックを持たない。analyze-* Skill に委ねる

## Step 1: プロジェクト設定の読み込み

`.claude/config/project.yml` から以下を取得:
- `mode.source`, `mode.tasks`
- `github.owner`, `github.repo`, `github.projectNumber`

### モード別の動作

| mode.tasks | 動作 |
|-----------|------|
| github | Issue 作成 + Project 追加（本 Skill の標準フロー） |
| local | 分析結果を `docs/tasks.md` の Triage セクションに追記 |

ローカルモードの場合、Step 5 の Issue 作成を `docs/tasks.md` への追記に置き換える。
フォーマットは `run/local-ops.md` に従い、ラベルは `[source:sentry]` のようにインラインタグで表現する。

## Step 2: 外部ソースの検出と対象決定

### 自動検出

`project.yml` の `external-sources` セクションから、対応する `analyze-*` Skill が存在するソースを検出する。

検出ロジック:
1. `project.yml` の `external-sources` 配下のキーを走査（`sentry`, `datadog` 等）
2. 各キーに対応する `.claude/skills/analyze-<key>/SKILL.md` が存在するか確認
3. 両方存在する → 利用可能なソースとして登録

### 引数の解釈

| 引数 | 動作 |
|------|------|
| `<source>` | 指定ソースのみ分析（例: `sentry`） |
| `<source> <project-slug>` | 指定ソースの特定プロジェクトのみ |
| `<source> --query <search>` | 自然言語フィルタ付き |
| `all`（デフォルト） | 検出された全ソースを分析 |

### ソースが見つからない場合（フォールバック）

外部連携が未設定でも `/triage` は利用可能。以下の順で対応する:

1. **利用可能なソースを案内**:
   ```
   ℹ 外部監視サービスが未設定です。

   対応可能なソース:
     sentry — /setup で MCP を追加し、project.yml の external-sources に設定
     (他のソースが追加されれば自動で検出されます)

   設定例（project.yml）:
     external-sources:
       sentry:
         organization: <org-slug>
         projects:
           - slug: <project-slug>
             directory: <対応ディレクトリ>

   外部連携なしでも、手動で Issue を作成できます:
     /run に自然言語で指示してください（例: /run ログイン画面でエラーが出る）
   ```

2. **指定ソースが未設定の場合**（例: `/triage datadog` だが datadog が未設定）:
   ```
   ⚠ datadog は未設定です。
     1. project.yml に datadog セクションを追加
     2. 対応する analyze-datadog Skill を配置
     3. 必要な MCP サーバーがあれば /setup で追加
   ```

## Step 2.5: MCP 設定・認証の確認

検出されたソースの `analyze-*` Skill が `requires-mcp` を持つ場合、MCP の設定と認証を確認する。

### 確認手順

1. `analyze-*` Skill の frontmatter から `requires-mcp` を読み取る（`name`, `transport`, `url`）
2. `.mcp.json` に該当する MCP サーバーが登録されているか確認
3. 登録状況に応じて分岐:

| `.mcp.json` に登録 | MCP ツール利用可能 | 判定 | 動作 |
|:-:|:-:|--------|------|
| ✓ | ✓ | 認証済み | → Step 3 へ |
| ✓ | authenticate のみ | 未認証 | → 認証を案内 |
| ✓ | ✗ | 要再起動 | → 再起動を案内 |
| ✗ | — | 未設定 | → `/setup` を案内し該当ソースをスキップ |

### 未認証の場合
```
⚠ <source> の MCP が未認証です。以下のコマンドをターミナルで実行してください:
$ claude mcp auth <name>
表示された URL をブラウザで開いて認証してください。
```
認証完了後、Step 3 へ。

### 未設定の場合（`.mcp.json` に未登録）
```
⚠ <source> の MCP サーバーが未設定です。
  /setup を実行して MCP を追加してください。
```
該当ソースをスキップし、他のソースがあれば続行する。

## Step 2.7: MCP でデータ取得

Agent の隔離コンテキストでは MCP ツールが利用できないため、triage 自身が MCP でデータを取得する。

対象ソースの `analyze-*` Skill のデータ取得手順に従い:
1. 未解決 Issue/エラー一覧を取得（MCP ツールは analyze-* Skill に記載）
2. 各 Issue の詳細を取得
3. 既存 GitHub Issue との重複チェック（`gh issue list --label source:<name>`）
4. 取得したデータと重複情報を構造化して Step 3 の analyst に渡す

### データ取得のエラーハンドリング

| エラー | 対応 |
|--------|------|
| 認証期限切れ（401/403） | 再認証を案内（`claude mcp auth <name>`） |
| レート制限（429） | 待機時間を表示し、後で再実行を提案 |
| タイムアウト | リトライ1回。再失敗なら該当ソースをスキップし他ソースを続行 |
| MCP サーバー接続不可 | サーバーの状態を確認し、`/setup` での再設定を案内 |
| データ取得成功だが 0 件 | 正常終了として「未解決エラーはありません」を報告 |

複数ソースがある場合、1つのソースの失敗で全体を中断しない。
失敗したソースをスキップし、成功したソースの結果を続行する。

## Step 3: analyst Agent に分析を委譲

取得済みデータを analyst Agent に渡し、分析のみを依頼する。
analyst は渡されたデータをもとに根本原因推定・影響範囲分析・修正方針の提案を行い、結果を返す。
analyst が MCP ツールや gh コマンドを実行する必要はない。

## Step 4: 分析結果の提示と承認

analyst の結果をユーザーに提示する:

```
分析結果

ソース: <source_name>
対象: <project_slugs>
未解決: <total>件
  新規（Issue 化対象）: <new>件
  既存と重複: <skipped>件

新規:
  1. <title> (<severity>) — <root_cause の要約>
  2. <title> (<severity>) — <root_cause の要約>

Issue を作成しますか？ (全て / 個別選択 / スキップ)
```

複数ソースがある場合は、ソースごとに提示・承認する。

## Step 5: タスク登録

承認された Issue について、モードに応じた登録を行う。

### GitHub モード（`tasks: github`）

1. ラベル作成（未存在の場合）: `source:<source_name>`, `scope:<label>`
2. Issue 作成:
   ```bash
   gh issue create --repo <owner>/<repo> \
     --title "<title>" \
     --label "bug,source:<source_name>,scope:<project_label>" \
     --body "<analyze-* Skill の body テンプレートに従う>"
   ```
3. Project に追加
4. ステータス設定:
   - `Triage` の選択肢が存在する → `Triage` に設定
   - `Triage` が存在しない → `Todo` にフォールバック

### ローカルモード（`tasks: local`）

`docs/tasks.md` の `## Triage` セクションにタスクを追記する。
フォーマットは `run/local-ops.md` に従う。

```markdown
## Triage

- [ ] <title> [bug] [source:<source_name>] [scope:<project_label>]
  - Sentry ID: `<source_id>`
  - 根本原因: <root_cause の要約>
  - 修正方針: <recommended_fix の要約>
```

## Step 6: 完了報告

### GitHub モード

```
Triage 完了

<ソースごとの結果>
  対象: <project_slugs>
  新規 GitHub Issue 作成: <created>件
  既存と重複（スキップ）: <skipped>件

作成した Issue:
  - #<number>: <title> (<severity>) → Triage
  - #<number>: <title> (<severity>) → Triage

次のステップ:
  /run #<number> で修正を開始できます
```

### ローカルモード

```
Triage 完了

<ソースごとの結果>
  対象: <project_slugs>
  新規タスク追記: <created>件
  既存と重複（スキップ）: <skipped>件

追記先: docs/tasks.md (Triage セクション)

次のステップ:
  /run <タスク名> で修正を開始できます
```

## 参照情報

- GitHub API・Issue/Project 運用: `.claude/skills/run/github-ops.md`
- プロジェクト設定: `.claude/config/project.yml`
