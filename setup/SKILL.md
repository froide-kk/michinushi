---
name: setup
description: AI駆動開発の初期セットアップと michinushi 本体の更新。リポジトリの状態を自動検出し、新規/既存に応じたガイダンスを提供。GitHub Project作成、スコープ確認、Skill配置、Agent 登録、michinushi 自体のアップデートを行う。
argument-hint: "[update]"
disable-model-invocation: true
---

# /setup — AI駆動開発 初期セットアップ・更新

リポジトリの状態を自動検出し、状況に応じたセットアップを実行する。
「新規ですか？」とは聞かず、見ればわかることは見て判断する。

## 引数の解析

`$ARGUMENTS` を確認し、フローを分岐する:

- **`update`** → [Update フロー](#update-フロー) を実行（michinushi 本体の更新）
- **引数なし** → 通常のセットアップフロー（Step 1 から）

## 事前準備

環境セットアップの詳細手順は [setup-guide.md](setup-guide.md) を参照。
ユーザーの環境に不足があれば、該当セクションを案内する。

## Step 1: リポジトリ状態の自動検出

以下を **すべて** 確認し、結果を一覧で表示する:

1. **Git状態**: `git log --oneline | wc -l` でcommit数（Git 未使用の場合はスキップ）
2. **CLAUDE.md**: ファイルの有無と内容の鮮度
3. **既存Skills**: `.claude/skills/` 配下のディレクトリ一覧
4. **プロジェクト設定**: `.claude/config/` 配下のファイル一覧
5. **GitHub Project**: `.claude/config/project.yml` を読み、`gh api graphql` でProject存在確認（GitHub モードの場合のみ）
6. **CI/CD**: `.github/workflows/` 配下のファイル一覧
7. **テスト**: 各 `package.json` から `vitest`, `jest`, `playwright` 等を検出
8. **パッケージ構成**: `packages/` 配下のディレクトリ一覧（モノレポの場合）
9. **gh auth スコープ**: `gh auth status` で必要スコープを満たすか確認（GitHub モードの場合のみ）
10. **Copilot Code Review**: 自動レビュー有効化の確認ガイダンス（GitHub モードの場合のみ）
11. **MCP サーバー**: `.claude/skills/` 配下の全 SKILL.md から `requires-mcp` を収集し、`.mcp.json` の設定状況と突合

## Step 1.5: 管理モードの選択

`.claude/config/project.yml` に `mode` が未定義の場合、管理モードを確認する。
（既に `mode` が定義済みなら、このステップはスキップする）

検出結果を踏まえて以下を提示し、選択してもらう:

```
管理モードを選択してください:

1. GitHub 完全連携 — GitHub でソース管理・タスク管理を一元化（GitHub Project + Issues）
2. Git + ローカルタスク — Git でソース管理、タスクはローカルファイル（docs/tasks.md）で管理
3. 完全ローカル — Git を使わず、タスクもローカルファイルで管理
```

選択に応じて `project.yml` の `mode` セクションを設定する:

| 選択 | source | tasks |
|------|--------|-------|
| 1 | `github` | `github` |
| 2 | `git` | `local` |
| 3 | `none` | `local` |

**判断の補助**: Step 1 の検出結果から推奨を示す。
- Git リポジトリ + `gh auth status` 成功 → 1 を推奨
- Git リポジトリ + `gh` 未インストールまたは認証失敗 → 2 を推奨
- Git リポジトリでない → 3 を推奨

## Step 2: 状況判定と提案

**既存リポジトリの場合**（commit数 > 0 かつ CLAUDE.md あり）:

検出結果を表示し、以下の選択肢を提示:
1. AI駆動開発の初期設定（Skill配置・設定確認）
2. 現状の棚卸し（コード分析→タスク化）
3. CLAUDE.md の改善（検出結果と現状の差分を反映）
4. 全て

**新規リポジトリの場合**（commit数 = 0 または CLAUDE.md なし）:

以下の順でセットアップを提案:
1. CLAUDE.md の作成（技術スタック・プロジェクト概要の定義）
2. `.claude/config/project.yml` の作成（mode + 連携情報）
3. タスク管理の初期化（GitHub モード: GitHub Project 作成 / ローカルモード: `docs/tasks.md` 作成）
4. `.claude/config/` にプロジェクト設定ファイルを配置
5. 要件定義から始めるか確認

## Step 2.5: 設計書スキーマの配置

`.claude/skills/doc-yaml-schema/schemas/` にマスタースキーマが存在する場合、`docs/schemas/` にコピーする。

**`docs/schemas/` が未作成の場合**:
→ ディレクトリ作成し、全スキーマをコピー
```bash
mkdir -p docs/schemas
cp .claude/skills/doc-yaml-schema/schemas/*.json docs/schemas/
```

**`docs/schemas/` が既存の場合**:
→ マスターに存在するが `docs/schemas/` にないファイルのみコピー（新規追加分）
→ 既存ファイルはプロジェクト固有のカスタマイズがある可能性があるため上書きしない
→ 差分がある場合は一覧を表示し、更新するか確認を挟む

```
📋 設計書スキーマ
  新規: 2ファイル追加
  差分あり: 3ファイル（マスターが更新済み）
    - common_metadata_schema.json
    - feature_design_schema.json
    - test_specification_schema.json
  → 差分ファイルを更新しますか？ (個別に確認 / 全更新 / スキップ)
```

## Step 3: 不足項目の対応

### GitHub モード（`source: github`, `tasks: github`）の場合

**gh auth スコープ不足**:
```
⚠ gh auth スコープ不足: project が必要です
以下のコマンドをターミナルで実行してください:
$ gh auth refresh -h github.com -s repo,read:project,project,read:org  # project.yml の requiredScopes 参照
```

**project.yml に GitHub 情報が不足**:
→ ユーザーにowner/repo/projectNumberを確認し、設定を追加

**GitHub Project未作成**:
→ 承認後: `gh project create --owner <owner> --title "<プロジェクト名>"`

### ローカルタスクモード（`tasks: local`）の場合

**`docs/tasks.md` 未作成**:
→ テンプレートから初期ファイルを作成（フォーマットは `.claude/skills/run/local-ops.md` を参照）

### 共通

**CLAUDE.md が現状と乖離**:
→ 検出情報（パッケージ構成、コマンド等）とCLAUDE.md記載を比較し、差分の更新を提案

**MCP サーバー未設定（オプション）**:
Skills の `requires-mcp` で必要な MCP と `.mcp.json` の設定を突合し、不足分を表示:
```
ℹ 外部連携（オプション）:
  sentry (required by: analyze-sentry)
    → 追加しますか？ (はい / スキップ)
    実行コマンド: claude mcp add --transport http sentry https://mcp.sentry.dev/mcp --scope project
```
ユーザーがスキップした場合はそのまま次に進む。外部連携はセットアップの必須項目ではない。
承認された MCP のみ `claude mcp add` を実行する。

MCP 追加後、`project.yml` の `external-sources` にもソース設定が必要であることを案内する:
```
ℹ MCP 追加後、project.yml に外部ソースの設定を追加してください:
  external-sources:
    sentry:
      organization: <org-slug>
      projects:
        - slug: <project-slug>
          directory: <対応ディレクトリ>
```

## Step 3.5: Agent の登録

michinushi の各 Agent ディレクトリ（`AGENT.md` を含むもの）を Claude Code のサブエージェントとして `.claude/agents/<name>.md` に登録する。

### 対象の検出

`.claude/skills/` 配下を走査し、`AGENT.md` を含むディレクトリを検出する。

```bash
find .claude/skills -maxdepth 2 -name 'AGENT.md'
```

### 登録処理

各 `AGENT.md` について以下の判定を行う:

| 状態 | 判定方法 | 動作 |
|------|---------|------|
| 新規 | `.claude/agents/<name>.md` が未存在 | そのままコピー |
| 同一 | 既存ファイルがマスター（michinushi 側 AGENT.md）と完全一致 | 何もしない（最新） |
| 差分あり | 既存ファイルとマスターが不一致 | 差分を表示し、ユーザーに対応を確認（後述） |

> **注意**: 「ユーザーが編集した」のか「michinushi の更新でマスターが変わった」のかは判定できない（過去のコピー時点のハッシュを保持していないため）。差分がある場合は常にユーザーに判断を委ねる。

#### 差分発見時の対応

差分が発生したファイルに対して、3択で確認:

```
⚠ .claude/agents/<name>.md にマスターとの差分があります
  ファイル: .claude/agents/architect.md
  マスター: .claude/skills/architect/AGENT.md

  差分:
  - <差分内容>

  対応を選んでください:
  1. マスターで上書き（ユーザー編集があれば失われる）
  2. 既存ファイルを保持（マスターの更新を取り込まない）
  3. 個別に確認して merge（手動編集後に再実行）
```

#### コマンド例

```bash
mkdir -p .claude/agents

# 新規 or 上書きの場合
cp .claude/skills/<agent-dir>/AGENT.md .claude/agents/<name>.md

# 差分確認
diff .claude/skills/<agent-dir>/AGENT.md .claude/agents/<name>.md
```

`<name>` は `AGENT.md` の frontmatter `name` フィールドの値を使用する（ディレクトリ名と一致するのが原則）。

### 削除対応

`.claude/agents/` に存在するが、michinushi 側で対応する `AGENT.md` が無い Agent については、ユーザー独自定義の可能性があるため削除しない。

### 確認表示

```
🤖 Agent 登録結果
  新規登録: 5件
    - architect, designer, implementer, tester, reviewer
  最新（同一）: 0件
  差分あり: 0件（ユーザー判断待ち）
  → .claude/agents/ に登録完了
```

新規登録または上書きが 1件以上ある場合は、続けて以下を案内する:

```
⚠ Claude Code を再起動してください
  新しい Agent / Skill 定義は起動時にのみ読み込まれます。
  再起動するまで今回の変更は反映されません。
  （/agents コマンドは一覧表示のみで、再読込は行いません）
```

## Step 4: 完了報告

```
✅ セットアップ完了

利用可能なコマンド:
  /run     — タスク実行（/run #33 or /run 自然言語指示）
  /triage  — 外部ソース分析 & Issue 化
  /todo    — タスク一覧（PM視点の構造化報告）
  /setup   — このセットアップ（再実行可能）

登録された Agent (.claude/agents/):
  architect    — 影響範囲分析・実装計画・工程要否判断
  designer     — UI/UX 設計
  implementer  — コード実装・設計書更新
  tester       — テスト全体の組み立て
  reviewer     — tech / biz 両観点レビュー
  analyst      — 外部エラーデータの分析

参照 Skill (.claude/skills/):
  design-*, impl-*, test-*, review-*, doc-yaml-schema
  各 Agent が必要に応じて参照する

michinushi の更新:
  /setup update — 最新版を取得し Agent を再登録
```

**MCP サーバーを新規追加した場合**:
MCP ツールの読み込みにはセッション再起動が必要。完了報告の末尾に以下を表示:
```
⚠ MCP サーバーを新規追加しました。ツールを有効にするためセッションの再起動が必要です。
  /exit で終了し、再度 claude を起動してください。
  ※ OAuth 認証は各スキルの初回実行時に自動で行われます。
```

---

## Update フロー

`/setup update` で起動する。michinushi 本体（`.claude/skills/` 配下）を最新版に取得し、ユーザーと対話しながら個別に反映する。

### Step U1: 一時ディレクトリへの取得

新版を一時ディレクトリにダウンロードする:

```bash
TMPDIR=".claude/.tmp/michinushi-update-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$TMPDIR"
curl -sL https://github.com/froide-kk/michinushi/archive/refs/heads/main.tar.gz \
  | tar xz --strip-components=1 -C "$TMPDIR"
```

> 中断・エラー時に一時ディレクトリが残らないよう、Step U4 で必ず削除する（成功・失敗どちらでも）。

### Step U2: 差分の集計

```bash
diff -rq .claude/skills "$TMPDIR" | sort
```

結果を以下に分類して一覧表示する:

| 分類 | 判定 | 例 |
|------|------|-----|
| **変更** | 両方に存在し内容が異なる | `architect/AGENT.md` の更新 |
| **新規** | 一時ディレクトリ側のみに存在 | michinushi に追加された新 Skill |
| **削除候補** | ローカル側のみに存在し、michinushi 既知のディレクトリ | 廃止された旧 Skill |
| **保持（ユーザー独自）** | ローカル側のみに存在し、michinushi 既知に含まれない | プロジェクト独自の Skill / Agent → 確認不要、そのまま残す |

### Step U3: 個別確認と反映

**変更があるファイル** ごとに 3択で確認する:

```
🔍 .claude/skills/architect/AGENT.md に変更があります
  ローカル: .claude/skills/architect/AGENT.md
  新版    : <TMPDIR>/architect/AGENT.md

  対応を選んでください:
  1. 上書き（新版で更新）
  2. スキップ（既存を保持）
  3. 差分を表示してから判断（diff を見て 1 / 2 を選び直す）
```

**新規ファイル** は追加するかを確認:

```
➕ 新規: .claude/skills/<name>/SKILL.md
  追加しますか？ (はい / スキップ)
```

**削除候補** はローカルから消すかを確認:

```
🗑 削除候補: .claude/skills/<old-name>/
  michinushi の最新版には存在しません。ローカルから削除しますか？ (はい / 保持)
```

承認されたものだけを `cp` / `rm -rf` で `.claude/skills/` 配下に反映する。

### Step U4: 一時ディレクトリの削除

成功・失敗いずれの場合も、最後に一時ディレクトリを削除する:

```bash
rm -rf "$TMPDIR"
```

> シェルスクリプトで実装する場合は `trap 'rm -rf "$TMPDIR"' EXIT` で保証する。

### Step U5: Agent の再登録

通常フローの [Step 3.5](#step-35-agent-の登録) と同じ処理を実行する。
更新で `AGENT.md` が変わった Agent は、ユーザー編集チェックの上で `.claude/agents/` を更新する。

### Step U6: 更新内容の報告

```
✅ michinushi 更新完了

反映した変更:
  - 上書き: <ファイル数>件
  - 新規追加: <ファイル数>件
  - 削除: <ファイル数>件
  - スキップ: <ファイル数>件（ユーザー判断で保留）

Agent 再登録: 5件中 N件更新
  → ユーザー編集のため保留: <あれば一覧>
```

Skill / Agent に変更があった場合は、続けて以下を案内する:

```
⚠ Claude Code を再起動してください
  更新された Skill / Agent 定義は起動時にのみ読み込まれます。
  再起動するまで /setup update の変更は反映されません。
  （/agents コマンドは一覧表示のみで、再読込は行いません）
```

> Skill / Agent の定義は Claude Code 起動時に読み込まれるため、`/setup update` 後は再起動が必須。再起動を忘れると、古い定義のまま動作してしまう。

### 注意

- 一時ディレクトリ `.claude/.tmp/` は `.gitignore` に追加することを推奨（誤コミット防止）
- michinushi 配布物（既知の Skill / Agent）を直接編集していると、Step U3 の上書き選択で変更が失われる。プロジェクト固有のカスタマイズは `.claude/agents/` 側で行うこと
