# Michinushi (道主貴)

**Claude Code Skills for AI-driven development.**

> 道主貴（みちぬしのむち）— 宗像三女神の別称。道を統べる最高神。
> 開発の道を導き、各Skillがその道を守る。

## Overview

Michinushi は [Claude Code](https://claude.ai/code) 向けの Skill コレクションです。
GitHub Project と連携し、Issue ベースのタスク管理から設計・実装・テスト・レビューまでを AI が一貫して遂行する開発フレームワークを提供します。

### GitHub Project 連携

Michinushi は [GitHub Projects (V2)](https://docs.github.com/en/issues/planning-and-tracking-with-projects) をタスク管理の中心に据えています。

```
GitHub Project
  │
  ├─ Issue #33 (status: Todo)
  ├─ Issue #34 (status: In Progress)
  └─ Issue #35 (status: In Review)
        │
        └─ PR #91 (Closes #35)
```

**ステータスフロー**:
```
Triage → Todo → In Progress → In Review → Done
```

- `/run #33` — Issue を読み取り、計画を立て、専門 Skill に作業を委譲。完了後に PR を作成し、ステータスを `In Review` に更新
- `/run 自然言語指示` — Issue が存在しない場合は自動作成し、Project に追加してから作業開始
- `/todo` — Project 全体を PM 視点で分析。要判断・進行中・次の候補・保留中に分類して報告

### 開発フロー

```
         /run #33
            │
   ┌────────┴────────┐
   │   計画・設計     │  architect, docgen
   ├─────────────────┤
   │   実装          │  coder, ui-designer
   ├─────────────────┤
   │   テスト        │  unit-tester, e2e-tester, security-tester
   ├─────────────────┤
   │   自己レビュー   │  tech-reviewer, biz-reviewer
   ├─────────────────┤
   │   PR 作成       │  Closes #33, status → In Review
   └─────────────────┘
```

各工程で必要な Skill だけが呼び出されます。バグ修正なら設計工程はスキップ、リファクタリングなら設計書更新は不要、といった判断を `/run` が行います。

### YAML 設計書

Michinushi は設計書を **JSON Schema 準拠の YAML** で管理します。要件定義からテスト仕様まで、22 種類のスキーマを用意しており、コードと同じリポジトリで設計書のバージョン管理・差分レビューが可能です。

生成された YAML 設計書は Web ビューアーで閲覧・PDF エクスポートできます。

- **ビューアー**: https://yaml-doc-renderer.web.app
- プロジェクトの `docs/` 配下の YAML ファイルをドラッグ＆ドロップで読み込み
- Mermaid 図（ER 図、フローチャート等）も自動レンダリング
- PDF エクスポート対応（右上のダウンロードボタン）

## Installation

### 1. Skills の導入

```bash
cd your-project
git subtree add --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
```

### 2. プロジェクト設定

`.claude/config/project.yml` を作成:

```yaml
github:
  owner: your-org
  repo: your-repo
  projectNumber: 1    # GitHub Project の番号
  baseBranch: main     # PR のベースブランチ
```

### 3. GitHub CLI のセットアップ

```bash
# インストール
brew install gh          # macOS
# その他: https://cli.github.com/

# 認証
gh auth login

# スコープ確認
gh auth status
# 必要なスコープ: repo, read:project, project, read:org

# 不足している場合
gh auth refresh -h github.com -s repo,read:project,project,read:org
```

### 4. 初期セットアップ

Claude Code で `/setup` を実行。リポジトリの状態を自動検出し、不足している設定をガイドします。

### Skills の更新

```bash
git subtree pull --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
```

## Skills

### User-invocable (ユーザーが直接呼び出す)

| Skill | Command | Description |
|-------|---------|-------------|
| **run** | `/run #33` `/run 自然言語指示` | タスクの理解・計画・専門 Skill への委譲・進捗管理 |
| **todo** | `/todo` | GitHub Project のタスク一覧を PM 視点で構造化して報告 |
| **setup** | `/setup` | AI 駆動開発の初期セットアップ。リポジトリ状態を自動検出 |

### Auto-invoked (/run が状況に応じて委譲)

| Skill | Role | Description |
|-------|------|-------------|
| **architect** | 設計 | 影響範囲分析・技術選定・実装計画 |
| **docgen** | 設計書 | JSON Schema 準拠の YAML 設計書を生成・更新 |
| **coder** | 実装 | 設計書・既存パターンに忠実なコード生成 |
| **ui-designer** | UI/UX | コンポーネント設計・スタイリング・インタラクション設計 |
| **unit-tester** | テスト | 入力検証・権限制御・ビジネスロジックのユニットテスト |
| **e2e-tester** | E2Eテスト | ユーザーの業務フロー全体をブラウザ上で検証 |
| **security-tester** | セキュリティ | OWASP Top 10 ベースのセキュリティテスト |
| **tech-reviewer** | 技術レビュー | セキュリティ・型安全・パフォーマンスをチェック |
| **biz-reviewer** | 業務レビュー | ビジネスロジックの正しさ・要件との整合性をチェック |

## Architecture

```
.claude/
  skills/          # <-- This repository (Michinushi)
    run/           # Task orchestration (/run)
    architect/     # Design & planning
    coder/         # Implementation
    ...
  config/          # Project-specific settings (NOT included)
    project.yml    # GitHub owner/repo/project info
    tech.yml       # Tech review checklist
    biz.yml        # Business review checklist
    security.yml   # Security test checklist
    ...
```

### Skill vs Config の分離

| | Skills (`skills/`) | Config (`config/`) |
|--|---|---|
| 内容 | 汎用的なテスト手法・レビュー観点 | プロジェクト固有のチェック項目 |
| 例 | 「OWASP Top 10 ベースでテストする」 | 「storeId フィルタ必須」 |
| リポジトリ | Michinushi (公開) | 各プロジェクト (非公開) |

Skills はどのプロジェクトでも使える汎用的な指示を記述し、プロジェクト固有の設定は `.claude/config/` に分離します。

### Config files

| File | Used by | Description |
|------|---------|-------------|
| `project.yml` | run, todo | GitHub 連携情報（必須） |
| `tech.yml` | tech-reviewer | 技術レビューのチェック項目 |
| `biz.yml` | biz-reviewer | 業務レビューのチェック項目 |
| `security.yml` | security-tester | セキュリティテストのチェック項目 |
| `ui.yml` | ui-designer | UI 方針・デザインシステム設定 |
| `design.yml` | docgen | 設計書規約・スキーマ参照先 |

## Creating a New Skill

```
skills/
  your-skill/
    SKILL.md       # Skill definition (required)
    *.md           # Supporting docs (optional)
```

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

### Frontmatter Fields

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
- [GitHub CLI](https://cli.github.com/) (`gh`) with `repo`, `read:project`, `project` scopes
- A [GitHub Project (V2)](https://docs.github.com/en/issues/planning-and-tracking-with-projects) for `/run` and `/todo`

## License

MIT

## Origin

> 宗像三女神は「道主貴（みちぬしのむち）」= 道を司る最高神と呼ばれた。
> 福岡・宗像大社に祀られる三柱の女神は、古来より航海の安全を守り、
> 人々を正しい道へ導いてきた。
>
> このプロジェクトも同様に、AI 駆動開発という未知の海を渡る開発者を
> 正しい道へ導くことを目指している。
