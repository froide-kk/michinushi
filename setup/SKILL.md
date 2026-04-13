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

## Step 3.5: Agent の登録

michinushi の各 Agent ディレクトリ（`AGENT.md` を含むもの）を Claude Code のサブエージェントとして `.claude/agents/<name>.md` に登録する。

### 対象の検出

`.claude/skills/` 配下を走査し、`AGENT.md` を含むディレクトリを検出する。

```bash
find .claude/skills -maxdepth 2 -name 'AGENT.md'
```

### 登録処理

各 `AGENT.md` について:

1. **新規登録**: `.claude/agents/<name>.md` が未存在 → そのままコピー
2. **既存・未編集**: 既存ファイルが直近のコピーと一致 → マスターで上書き更新
3. **既存・編集済み**: 既存ファイルがユーザー編集を含む可能性 → 差分を表示し、更新するか確認

```bash
mkdir -p .claude/agents
cp .claude/skills/<agent-dir>/AGENT.md .claude/agents/<name>.md
```

`<name>` は `AGENT.md` の frontmatter `name` フィールドの値を使用する（ディレクトリ名と一致するのが原則）。

### 削除対応

`.claude/agents/` に存在するが、michinushi 側で対応する `AGENT.md` が無い Agent については、ユーザー独自定義の可能性があるため削除しない。

### 確認表示

```
🤖 Agent 登録結果
  新規登録: 5件
    - architect, designer, implementer, tester, reviewer
  更新: 0件
  ユーザー編集を検出: 0件
  → .claude/agents/ に登録完了
```

## Step 4: 完了報告

```
✅ セットアップ完了

利用可能なコマンド:
  /run     — タスク実行（/run #33 or /run 自然言語指示）
  /todo    — タスク一覧（PM視点の構造化報告）
  /setup   — このセットアップ（再実行可能）

登録された Agent (.claude/agents/):
  architect    — 影響範囲分析・実装計画・工程要否判断
  designer     — UI/UX 設計
  implementer  — コード実装・設計書更新
  tester       — テスト全体の組み立て
  reviewer     — tech / biz 両観点レビュー

参照 Skill (.claude/skills/):
  design-*, impl-*, test-*, review-*, doc-yaml-schema
  各 Agent が必要に応じて参照する

michinushi の更新:
  /setup update — 最新版を取得し Agent を再登録
```

---

## Update フロー

`/setup update` で起動する。michinushi 本体（`.claude/skills/` 配下）を最新版に更新し、Agent 登録を再実行する。

### Step U1: インストール方式の判定

`.claude/config/project.yml` の `michinushi.installMethod` を確認する:

- **キャッシュあり**（`subtree` または `curl`）→ その方式を採用
- **キャッシュなし** → 自動判定:
  ```bash
  git log --grep="git-subtree-dir: \.claude/skills" --oneline | head -1
  ```
  - 結果あり → `subtree`
  - 結果なし → `curl`
- 判定結果を `project.yml` の `michinushi.installMethod` に保存（次回以降のキャッシュ）

### Step U2: 事前チェック

#### subtree の場合

1. ワーキングツリーが clean か確認（`git status --porcelain`）。汚れている場合は警告し、コミットまたはスタッシュを促してから再実行
2. 現在のブランチを記録（後の参考用）

#### curl の場合

1. `.claude/skills/` 配下にユーザーが追加した独自 Skill / Agent がないか確認
   - michinushi 既知のディレクトリ（`run`, `todo`, `setup`, `architect`, `designer`, ... など）以外があれば一覧表示
   - 上書きされない位置にあるか確認（`tar` 展開時に消える可能性を警告）
2. ユーザー独自のものがあれば、退避 or 続行を確認

### Step U3: 更新の実行

#### subtree の場合

```bash
git subtree pull --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
```

コンフリクト発生時:
- ユーザーがローカルで michinushi の Skill / Agent を編集しているケースが多い
- コンフリクト箇所を表示し、ユーザーに解決を依頼

#### curl の場合

```bash
curl -sL https://github.com/froide-kk/michinushi/archive/refs/heads/main.tar.gz \
  | tar xz --strip-components=1 -C .claude/skills
```

### Step U4: Agent の再登録

通常フローの [Step 3.5](#step-35-agent-の登録) と同じ処理を実行する。
更新で `AGENT.md` が変わった Agent は、ユーザー編集チェックの上で `.claude/agents/` を更新する。

### Step U5: 更新内容の報告

```
✅ michinushi 更新完了

更新前: <commit hash> (subtree の場合)
更新後: <commit hash>

変更:
  - <変更されたディレクトリ一覧>
  - 新規 Skill: <あれば>
  - 削除 Skill: <あれば>

Agent 再登録: 5件中 N件更新
  → ユーザー編集のため保留: <あれば一覧>
```

### 注意

- subtree 方式で history を rebase 等で書き換えた後は自動判定が失敗する場合がある。その場合は `project.yml` の `michinushi.installMethod` を手動で `subtree` に設定する
- curl 方式は基本上書きのため、michinushi 配布物（既知の Skill / Agent）を直接編集しているとロストする。`.claude/agents/` 側で編集すべき
