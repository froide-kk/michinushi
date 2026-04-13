# Michinushi (道主貴)

**Claude Code Skills & Agents for AI-driven development.**

> 道主貴（みちぬしのむち）— 宗像三女神の別称。道を統べる最高神。
> 開発の道を導き、各 Skill / Agent がその道を守る。

> ⚠️ **破壊的変更（2026-04 リリース）** — Skill と Agent の再構成を行いました。9つの Skill（architect / coder / docgen / ui-designer / unit-tester / e2e-tester / security-tester / tech-reviewer / biz-reviewer）が **5つの Agent**（architect / designer / implementer / tester / reviewer）+ **複数の参照 Skill** に再編されています。既存ユーザーは [Migration Guide](#migration-guide-from-previous-versions) を参照してください。

## Overview

Michinushi は [Claude Code](https://claude.ai/code) 向けの Skill コレクションです。
設計・実装・テスト・レビューまでを AI が一貫して遂行する開発フレームワークを提供します。

### 管理モード

プロジェクトの環境に応じて3つの管理モードを選択できます（`/setup` で設定）:

| モード | ソース管理 | タスク管理 | 用途 |
|-------|-----------|-----------|------|
| GitHub 完全連携 | GitHub | GitHub Projects + Issues | GitHub でソース・タスクを一元管理 |
| Git + ローカル | Git | `docs/tasks.md` | リモート無し、または GitHub 以外のホスティング |
| 完全ローカル | なし | `docs/tasks.md` | Git を使わないプロジェクト |

### タスク管理

**GitHub モード** — [GitHub Projects (V2)](https://docs.github.com/en/issues/planning-and-tracking-with-projects) をタスク管理の中心に据えます。

```
GitHub Project
  │
  ├─ Issue #33 (status: Todo)
  ├─ Issue #34 (status: In Progress)
  └─ Issue #35 (status: In Review)
        │
        └─ PR #91 (Closes #35)
```

**ローカルモード** — `docs/tasks.md` でタスクを管理します。GitHub 不要で動作します。

**共通のステータスフロー**:
```
Triage → Todo → In Progress → In Review → Done
```

- `/run #33` — タスクを読み取り、計画を立て、専門 Skill に作業を委譲
- `/run 自然言語指示` — タスクが存在しない場合は自動作成してから作業開始
- `/todo` — タスク全体を PM 視点で分析。要判断・進行中・次の候補・保留中に分類して報告

### 開発フロー

```
         /run #33
            │
   ┌────────┴──────────┐
   │   計画・設計       │  architect Agent → designer Agent (※)
   ├───────────────────┤
   │   実装             │  implementer Agent
   ├───────────────────┤
   │   テスト           │  tester Agent (unit / e2e / security)
   ├───────────────────┤
   │   自己レビュー      │  reviewer Agent (tech + biz 観点)
   ├───────────────────┤
   │   PR 作成          │  Closes #33, status → In Review
   └───────────────────┘
```

※ designer は UI 変更を含む場合のみ。architect が判断します。

各工程で必要な Agent だけが呼び出されます。バグ修正なら設計工程はスキップ、リファクタリングなら設計書更新は不要、といった判断を `architect` が行い、`/run` (Conductor) がそれに従って Agent を起動します。

### Skill と Agent の役割分担

- **Skill** = ピンポイントな手順・参照材料（テスト記法、レビュー観点、ADR 形式など）
- **Agent** = それらを組み合わせて目的を達成するオーケストレーター（判断 + 委譲）

例: `tester` Agent はタスクに応じて `test-design-principles` / `test-vitest` / `test-playwright` / `test-owasp-checklist` の中から必要な Skill を選んでテストを実装します。

### YAML 設計書

Michinushi は設計書を **JSON Schema 準拠の YAML** で管理します。要件定義からテスト仕様まで、22 種類のスキーマを用意しており、コードと同じリポジトリで設計書のバージョン管理・差分レビューが可能です。

生成された YAML 設計書は Web ビューアーで閲覧・PDF エクスポートできます。

- **ビューアー**: https://yaml-doc-renderer.web.app
- プロジェクトの `docs/` 配下の YAML ファイルをドラッグ＆ドロップで読み込み
- Mermaid 図（ER 図、フローチャート等）も自動レンダリング
- PDF エクスポート対応（右上のダウンロードボタン）

## Installation

### 1. Skills の導入

#### A. Git subtree（Git 管理プロジェクト向け・推奨）

```bash
cd your-project
git subtree add --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
```

> **初回コミットがない場合**: `git subtree add` は既存のコミット履歴が必要です。新規リポジトリでは先に初回コミットを作成してください:
> ```bash
> git init
> git commit --allow-empty -m "initial commit"
> ```
>
> **未コミットの変更がある場合**: ワーキングツリーがクリーンでないとエラーになります。先にコミットするかスタッシュしてください:
> ```bash
> git stash          # 一時退避
> git subtree add --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
> git stash pop      # 退避した変更を復元
> ```

#### B. curl でダウンロード（Git 未使用プロジェクト向け）

```bash
cd your-project
mkdir -p .claude/skills
curl -sL https://github.com/froide-kk/michinushi/archive/refs/heads/main.tar.gz \
  | tar xz --strip-components=1 -C .claude/skills
```

#### 更新（A / B 共通）

導入後の更新は `/setup update` で行います。インストール方式は自動判定され、適切なコマンドが実行されます（手動で `git subtree pull` / `curl` を実行することも可能）。

### 2. 初期セットアップ

Claude Code で `/setup` を実行。プロジェクトの状態を自動検出し、管理モードの選択から設定まで対話的にガイドします。

**GitHub モードの場合**は追加で GitHub CLI のセットアップが必要です:

```bash
# インストール
brew install gh          # macOS
# その他: https://cli.github.com/

# 認証
gh auth login

# スコープ追加
gh auth refresh -h github.com -s repo,read:project,project,read:org
```

## Skills と Agents

### User-invocable Skills (ユーザーが直接呼び出す)

| Skill | Command | Description |
|-------|---------|-------------|
| **run** | `/run #33` `/run 自然言語指示` | タスクの理解・計画・Agent への委譲・進捗管理 (Conductor) |
| **todo** | `/todo` | タスク一覧を PM 視点で構造化して報告（GitHub / ローカル両対応） |
| **setup** | `/setup` `/setup update` | AI 駆動開発の初期セットアップ・michinushi 本体の更新 |

### Agents (Conductor が状況に応じて委譲)

`/setup` 実行時に `.claude/agents/<name>.md` に登録される。

| Agent | Role | Description |
|-------|------|-------------|
| **architect** | 設計 | 影響範囲分析・実装計画・工程要否判断 |
| **designer** | UI/UX | コンポーネント構成・操作フロー・状態遷移の設計 |
| **implementer** | 実装 | 設計書・既存パターンに忠実なコード実装と設計書更新 |
| **tester** | テスト | unit / e2e / security テストを必要に応じて組み立て |
| **reviewer** | レビュー | tech 観点 + biz 観点を統合したコード・設計レビュー |

### Reference Skills (Agent が必要に応じて参照)

`.claude/skills/` 配下にディレクトリで配置される。各 Agent が状況に応じて読み込む。

| Category | Skill | Used by |
|----------|-------|---------|
| 設計 | `design-impact-analysis`, `design-impl-plan-format`, `design-adr-format` | architect |
| UI | `design-ui-patterns` | designer |
| 実装 | `impl-coding-conventions`, `doc-yaml-schema` | implementer |
| テスト | `test-design-principles`, `test-vitest`, `test-playwright`, `test-owasp-checklist` | tester |
| レビュー | `review-tech-checklist`, `review-biz-checklist` | reviewer |

## Architecture

```
.claude/
  skills/                    # <-- This repository (Michinushi)
    run/SKILL.md             # /run (Conductor)
    todo/SKILL.md            # /todo
    setup/SKILL.md           # /setup
    architect/AGENT.md       # Agent 定義（setup で .claude/agents/ にコピー）
    designer/AGENT.md
    implementer/AGENT.md
    tester/AGENT.md
    reviewer/AGENT.md
    design-*/SKILL.md        # 設計参照 Skill
    impl-*/SKILL.md          # 実装参照 Skill
    test-*/SKILL.md          # テスト参照 Skill
    review-*/SKILL.md        # レビュー参照 Skill
    doc-yaml-schema/         # 設計書スキーマ + 生成手順
  agents/                    # /setup で AGENT.md からコピーされる
    architect.md
    designer.md
    implementer.md
    tester.md
    reviewer.md
  config/                    # Project-specific settings (NOT included)
    project.yml              # 管理モード・連携情報
    tech.yml                 # 技術レビューチェック項目
    biz.yml                  # 業務レビューチェック項目
    security.yml             # セキュリティテストチェック項目
    ui.yml                   # UI方針
    design.yml               # 設計書規約
```

### Skill vs Agent

- **Skill** (`SKILL.md`): ピンポイントな手順・参照材料。`.claude/skills/` に配置すれば Claude Code が自動認識
- **Agent** (`AGENT.md`): 判断と委譲を担う。`/setup` が `.claude/agents/<name>.md` にコピー登録

ファイル名で識別するため、Skill と Agent が同名で衝突することはありません。

### Skill vs Config の分離

| | Skills (`skills/`) | Config (`config/`) |
|--|---|---|
| 内容 | 汎用的なテスト手法・レビュー観点 | プロジェクト固有のチェック項目 |
| 例 | 「OWASP Top 10 ベースでテストする」 | 「storeId フィルタ必須」 |
| リポジトリ | Michinushi (公開) | 各プロジェクト (非公開) |

Skills/Agents はどのプロジェクトでも使える汎用的な指示を記述し、プロジェクト固有の設定は `.claude/config/` に分離します。

### Config files

| File | Used by | Description |
|------|---------|-------------|
| `project.yml` | run, todo, setup | 管理モード・連携情報（必須） |
| `tech.yml` | reviewer Agent | 技術レビューのチェック項目 |
| `biz.yml` | reviewer Agent | 業務レビューのチェック項目 |
| `security.yml` | tester Agent | セキュリティテストのチェック項目 |
| `ui.yml` | designer Agent | UI 方針・デザインシステム設定 |
| `design.yml` | architect / implementer Agent | 設計書規約・スキーマ参照先 |

## Creating a New Skill or Agent

```
skills/
  your-feature/
    SKILL.md   # 参照 Skill にする場合
    AGENT.md   # Agent にする場合
    *.md       # Supporting docs (optional)
```

ファイル名で種別を区別する（同じディレクトリに両方を置かない）。

### SKILL.md Format

```yaml
---
name: your-skill
description: Short description. Used for auto-matching.
user-invocable: false          # true = user calls directly via /command
disable-model-invocation: true # true = only invoked by explicit /command
---

# Your Skill — Title

Instructions for Claude when this skill is invoked.
```

### AGENT.md Format

Claude Code のサブエージェント標準フォーマットに準拠する。

```yaml
---
name: your-agent
description: Agent の役割と起動条件。Conductor が委譲先を判断するために使う。
tools: Read, Grep, Glob, Bash  # 使用可能ツール（任意）
model: opus | sonnet | haiku    # 使用モデル（任意）
---

# Your Agent

責務、使う Skill、判断基準、出力フォーマットなどの指示。
```

`/setup` 実行時に `.claude/agents/<name>.md` にコピー登録される。

### Frontmatter Fields (SKILL.md)

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Skill identifier (matches directory name) |
| `description` | Yes | Used by Claude to decide when to invoke |
| `user-invocable` | No | `true` if user can call via `/name` |
| `disable-model-invocation` | No | `true` to prevent auto-invocation |
| `argument-hint` | No | Hint for command arguments |
| `allowed-tools` | No | Restrict available tools |

## Contributing

1. このリポジトリを Fork
2. Skill を追加・修正
3. Pull Request を作成

各 Skill は汎用的に設計してください。プロジェクト固有のロジックは `.claude/config/` に分離します。

## Requirements

- [Claude Code](https://claude.ai/code) CLI, desktop app, or IDE extension
- GitHub モードの場合のみ:
  - [GitHub CLI](https://cli.github.com/) (`gh`) with `repo`, `read:project`, `project` scopes
  - A [GitHub Project (V2)](https://docs.github.com/en/issues/planning-and-tracking-with-projects) for `/run` and `/todo`

## Migration Guide (from previous versions)

旧構成（9つの Skill）から新構成（5つの Agent + 参照 Skill 群）への移行手順:

### 1. Michinushi 本体を更新

```
/setup update
```

インストール方式（subtree / curl）は自動判定され、適切なコマンドが実行されます。手動で実行する場合は導入時のコマンドを再実行してください。

### 2. 旧 Skill の削除（subtree pull の場合は自動）

旧 Skill ディレクトリは削除されています。手動配置の場合は以下を削除:

```
.claude/skills/architect/SKILL.md  → architect/AGENT.md (中身も再構成)
.claude/skills/coder/               → implementer/AGENT.md
.claude/skills/docgen/              → doc-yaml-schema/ (Skill にリネーム)
.claude/skills/ui-designer/         → designer/AGENT.md
.claude/skills/unit-tester/         → 削除（tester Agent + test-* Skill に統合）
.claude/skills/e2e-tester/          → 削除
.claude/skills/security-tester/     → 削除
.claude/skills/tech-reviewer/       → 削除（reviewer Agent + review-* Skill に統合）
.claude/skills/biz-reviewer/        → 削除
```

### 3. `/setup` を再実行

```
/setup
```

Step 3.5 で AGENT.md が `.claude/agents/` に登録されます。

### 4. 設計書スキーマのパス更新

`docgen/schemas/` を参照していた箇所は `doc-yaml-schema/schemas/` に変更されています。プロジェクト内に `$schema` パスをハードコードしている設計書 YAML がある場合は確認してください。

### 5. プロジェクト固有 Config の確認

`.claude/config/` の各 YAML（tech.yml, biz.yml 等）の中身は変更不要です。新しい reviewer / tester Agent がそのまま読み込みます。

---

## License

MIT

## Origin

> 宗像三女神は「道主貴（みちぬしのむち）」= 道を司る最高神と呼ばれた。
> 福岡・宗像大社に祀られる三柱の女神は、古来より航海の安全を守り、
> 人々を正しい道へ導いてきた。
>
> このプロジェクトも同様に、AI 駆動開発という未知の海を渡る開発者を
> 正しい道へ導くことを目指している。
