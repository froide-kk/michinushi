# 開発プロセス定義

全エージェントが参照する共通ルール。
各エージェントは自分の担当工程だけでなく、前後の工程を理解した上で作業する。

## Skill と Agent の使い分け

michinushi の機能は **Skill** と **Agent** の2種類で構成される。

| 種別 | 役割 | ファイル名 | 配置場所（ユーザー側） |
|------|-----|-----------|--------------------|
| **Skill** | ピンポイントな手順・参照材料 | `SKILL.md` | `.claude/skills/<name>/SKILL.md` |
| **Agent** | 判断・委譲・複数 Skill の組み合わせ | `AGENT.md` | `.claude/agents/<name>.md` |

### 設計原則

- **Skill** = 「どうやるか」を書く（テスト記法、レビュー観点、ADR形式、コーディング規約など）
  - 単機能・再利用可能・言語/ライブラリ依存物はライブラリごとに分離（例: `test-vitest`, `test-playwright`）
- **Agent** = 「何をすべきか」を判断して Skill を組み合わせる
  - 隔離コンテキストで動作し、メイン会話を判断ロジックの情報汚染から守る

### AGENT.md フォーマット

Claude Code のサブエージェント標準フォーマットに準拠する:

```yaml
---
name: agent-name
description: Agent の役割と起動条件（Conductor が委譲先を判断するために使う）
tools: Read, Grep, Glob, Bash  # 使用可能ツール（任意）
model: opus | sonnet | haiku    # 使用モデル（任意）
---

# Agent Name

（指示内容: 責務、使う Skill、判断基準、出力フォーマット）
```

### SKILL.md フォーマット

```yaml
---
name: skill-name
description: Skill の内容（Agent が読み込み判断に使う）
user-invocable: false           # true ならユーザーが /<name> で直接呼び出せる
disable-model-invocation: true  # true ならモデルの自動呼び出しを無効化
---

# Skill Name

（手順・参照材料）
```

### Agent への登録方法

`AGENT.md` は Claude Code が自動認識しないため、`/setup` 実行時に `.claude/agents/<name>.md` へコピー登録する。michinushi 本体を更新した後は `/setup` を再実行することで Agent 定義が更新される。

## 管理モード

プロジェクトの `.claude/config/project.yml` で管理モードを設定する。

```yaml
mode:
  source: github | git | none   # ソース管理方式
  tasks: github | local         # タスク管理方式
```

| パターン | source | tasks | 用途 |
|---------|--------|-------|------|
| GitHub 完全連携 | `github` | `github` | GitHub でソース・タスクを一元管理 |
| Git + ローカルタスク | `git` | `local` | Git 管理だがリモートが GitHub 以外、またはリモート無し |
| 完全ローカル | `none` | `local` | Git を使わないプロジェクト |

**制約**: `tasks: github` は `source: github` を必要とする。

**後方互換**: `mode` が未定義の場合は `source: github`, `tasks: github` として扱う。

各エージェントの動作はモードによって以下が変わる:
- **タスク参照先**: GitHub Issues / `docs/tasks.md`
- **完了時の操作**: PR 作成・Project 更新 / `docs/tasks.md` のステータス移動 / サマリー報告のみ
- **参照ドキュメント**: `run/github-ops.md` / `run/local-ops.md`

## エージェント構成

| 種別 | 名称 | 担当 |
|------|-----|------|
| Skill | `run` (Conductor) | 工程管理・委譲・完了判定 |
| Skill | `todo` | タスク一覧の構造化報告 |
| Skill | `setup` | 初期セットアップ |
| Agent | `architect` | 要件分析・影響範囲・実装計画・工程要否判断 |
| Agent | `designer` | UI レイアウト・UX 設計・インタラクション設計 |
| Agent | `implementer` | コード実装・設計書更新 |
| Agent | `tester` | テスト全体の組み立て（unit / e2e / security） |
| Agent | `reviewer` | コード・設計レビュー（tech / biz 両観点） |

各 Agent は `.claude/agents/<name>.md` に登録される（`/setup` で配置）。

参照 Skill（Agent が必要に応じて使う）:

- 設計系: `design-impact-analysis`, `design-impl-plan-format`, `design-adr-format`, `design-ui-patterns`
- 実装系: `impl-coding-conventions`, `doc-yaml-schema`
- テスト系: `test-design-principles`, `test-vitest`, `test-playwright`, `test-owasp-checklist`
- レビュー系: `review-tech-checklist`, `review-biz-checklist`

## 開発フロー

```
        ┌─────────────────────────────────────────────────┐
        │               設計フェーズ                        │
        │                                                 │
        │  architect ─→ (designer) ─→ reviewer (設計)     │
        │                  (※1)                            │
        │                                                 │
        │  reviewer が MUST 指摘 → 該当 Agent に差し戻し    │
        └────────────────────┬────────────────────────────┘
                             │ 設計レビュー通過
        ┌────────────────────▼────────────────────────────┐
        │               実装フェーズ                        │
        │                                                 │
        │  implementer ─→ tester ─→ reviewer (コード)      │
        │                                                 │
        │  reviewer が MUST 指摘 → 該当 Agent に差し戻し    │
        └────────────────────┬────────────────────────────┘
                             │ コードレビュー通過
                             ▼
                      コミット・PR 作成
```

※1 designer は UI/UX 変更を含む場合のみ。architect が判断する。
※ 設計書の作成・更新は implementer が `doc-yaml-schema` Skill を使って行う。

## 各 Agent の責務と後続への配慮

### architect

**担当**: 要件分析・影響範囲分析・実装計画の策定・工程要否判断

**後続のために**:
- 必要な工程を `required-phases` で明示（designer / implementer / tester / reviewer / 設計書更新の要否）
- implementer が迷わないよう、変更対象ファイル・実装順序・依存関係を具体的に示す
- 設計上の重要な分岐点で判断した場合は ADR 作成を提案

### designer

**担当**: UI レイアウト・UX 設計・インタラクション設計

**後続のために**:
- implementer が実装できるよう、具体的な UI 仕様（コンポーネント構成・入力形式・バリデーション・状態遷移）を示す
- 設計書（画面設計）の更新が必要な場合は更新内容を明示

### implementer

**担当**: 実装計画に基づくコード実装、必要に応じて設計書（YAML）の更新

**後続のために**:
- tester がテストを書けるよう、変更した関数・エンドポイント・コンポーネントを報告する
- reviewer がレビューしやすいよう、変更ファイル一覧と設計からの逸脱（あれば）を報告する

### tester

**担当**: テスト全体の組み立て・実行（unit / e2e / security）

**後続のために**:
- reviewer がテスト結果を確認できるよう、パス/フェイル・カバレッジを報告する
- テスト失敗時は原因（テスト側 or 実装側）を切り分けて報告する

### reviewer

**担当**: コード・設計のレビュー（tech 観点 + biz 観点を統合）

**差し戻し時のために**:
- 修正が必要な Agent を `delegated-to` で明示する（implementer / designer / architect）
- MUST / SHOULD / NIT の重大度を分類し、修正内容を具体的に示す

## Conductor の役割

Conductor は上記フローの **進行管理のみ** を行う。

- architect の分析結果と判断を読み取り、実行すべき工程を抽出する
- 各工程で該当 Agent を起動し、結果を受け取る
- reviewer の判定に基づき、次の工程に進むか差し戻すかを決める
- コード・設計書・テストを自分で書かない。レビューを自分でしない

## 工程の省略

全タスクで全工程を実行する必要はない。**architect が分析結果でどの工程が必要かを判断する。** Conductor は architect の判断に従い、指定された工程のみを実行する。

実装フェーズの基本フローは **implementer → tester → reviewer** とする。誤字脱字や文言修正などの軽微変更に限り、architect が影響範囲が限定的と判断した場合のみ reviewer を省略できる。

典型的なパターン:

- **新機能**: 全工程（architect → designer → implementer → tester → reviewer）
- **バグ修正**: architect → reviewer（設計）→ implementer → tester → reviewer（設計書更新不要なら設計 reviewer を省略）
- **リファクタリング**: implementer → tester → reviewer（設計フェーズ不要）
- **設計変更のみ**: architect → implementer（設計書更新）→ reviewer（実装フェーズ不要）
- **typo 修正**: implementer → tester（architect が軽微変更と判断した場合のみ reviewer を省略可）

これらはあくまで典型例であり、最終的には architect が個別のタスク内容に基づいて判断する。

## ユーザーへの質問ルール

全エージェント共通のルールとして、ユーザーに質問する際は以下の手順を厳守する。

### 手順

1. **質問の全体像を先に開示する** — 最初に「N点確認させてください」と宣言し、全質問の概要を番号付きリストで提示する
2. **一問一答で進める** — 概要提示後、1番目の質問から1つずつ聞く。回答を得てから次の質問に進む

### 例

```
3点確認させてください:
1. 認証方式について
2. データの保持期間について
3. エラー時の挙動について

まず1点目です。認証方式は OAuth と API キーのどちらを想定していますか？
```

### 注意

- 概要提示の段階では各質問の詳細には踏み込まない（見出しレベルの概要のみ）
- 前の回答によって後の質問が不要になった場合はスキップし、その旨を伝える
- 質問が1つだけの場合は概要提示を省略してよい
