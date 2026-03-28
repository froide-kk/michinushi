# Michinushi (道主貴)

**Claude Code Skills for AI-driven development.**

> 道主貴（みちぬしのむち）— 宗像三女神の別称。道を統べる最高神。
> 開発の道を導き、各Skillがその道を守る。

## Overview

Michinushi は [Claude Code](https://claude.ai/code) 向けの Skill コレクションです。
`/run` が GitHub Issue からタスクを読み取り、状況に応じて専門 Skill に作業を委譲する AI 駆動開発フレームワークを提供します。

```
User ─── /run #33 ─┬─→ architect     (設計)
                    ├─→ coder         (実装)
                    ├─→ unit-tester   (テスト)
                    ├─→ tech-reviewer (レビュー)
                    └─→ ...
```

## Skills

### User-invocable (ユーザーが直接呼び出す)

| Skill | Command | Description |
|-------|---------|-------------|
| **run** | `/run #33` `/run 自然言語指示` | タスクの理解・計画・専門Skillへの委譲・進捗管理 |
| **todo** | `/todo` | GitHub Project のタスク一覧を PM 視点で構造化して報告 |
| **setup** | `/setup` | AI 駆動開発の初期セットアップ。リポジトリ状態を自動検出 |

### Auto-invoked (/run が状況に応じて委譲)

| Skill | Role | Description |
|-------|------|-------------|
| **architect** | 設計 | 影響範囲分析・技術選定・実装計画 |
| **docgen** | 設計書 | JSON Schema 準拠の YAML 設計書を生成・更新 |
| **coder** | 実装 | 設計書・既存パターンに忠実なコード生成 |
| **ui-designer** | UI/UX | コンポーネント設計・スタイリング・インタラクション設計 |
| **unit-tester** | テスト | Server Action の入力検証・権限制御・ビジネスロジックのテスト |
| **e2e-tester** | E2Eテスト | ユーザーの業務フロー全体をブラウザ上で検証 |
| **security-tester** | セキュリティ | OWASP Top 10 ベースのセキュリティテスト |
| **tech-reviewer** | 技術レビュー | セキュリティ・型安全・エラーハンドリング・パフォーマンスをチェック |
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

## Installation

### 既存プロジェクトに導入（推奨）

```bash
cd your-project
git subtree add --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
```

これで `.claude/skills/` に全 Skill が配置されます。`/setup` を実行して初期セットアップを進めてください。

### Skills の更新

```bash
git subtree pull --prefix=.claude/skills https://github.com/froide-kk/michinushi.git main --squash
```

### 開発元リポジトリからの同期（メンテナ向け）

サービス開発と Skills 開発を同時に行うリポジトリから、Skills の変更を Michinushi に push する場合:

```bash
# リモート追加（初回のみ）
git remote add michinushi git@github.com:froide-kk/michinushi.git

# Skills の変更を push
git subtree push --prefix=.claude/skills michinushi main
```

## Contributing

1. このリポジトリを Fork
2. Skill を追加・修正
3. Pull Request を作成

各 Skill は汎用的に設計してください。プロジェクト固有のロジックは `.claude/config/` に分離します。

## Configuration

Skills が参照するプロジェクト固有の設定ファイルを `.claude/config/` に配置します。

### project.yml (必須)

```yaml
github:
  owner: your-org
  repo: your-repo
  projectNumber: 1
  baseBranch: main
```

### Optional config files

| File | Used by | Description |
|------|---------|-------------|
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
description: Short description in Japanese + English. Used for auto-matching.
user-invocable: false          # true = user calls directly via /command
disable-model-invocation: true # true = only invoked by explicit /command
---

# Your Skill — タイトル

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

## Requirements

- [Claude Code](https://claude.ai/code) CLI or IDE extension
- [GitHub CLI](https://cli.github.com/) (`gh`) with `repo`, `read:project`, `project` scopes
- A GitHub Project (for `/run` and `/todo`)

## License

MIT

## Origin

> 宗像三女神は「道主貴（みちぬしのむち）」= 道を司る最高神と呼ばれた。
> 福岡・宗像大社に祀られる三柱の女神は、古来より航海の安全を守り、
> 人々を正しい道へ導いてきた。
>
> このプロジェクトも同様に、AI 駆動開発という未知の海を渡る開発者を
> 正しい道へ導くことを目指している。
