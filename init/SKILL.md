---
name: init
description: AI駆動開発の初期セットアップ。リポジトリの状態を自動検出し、新規/既存に応じたガイダンスを提供。GitHub Project作成、スコープ確認、Skill配置を行う。
disable-model-invocation: true
---

# /init — AI駆動開発 初期セットアップ

リポジトリの状態を自動検出し、状況に応じたセットアップを実行する。
「新規ですか？」とは聞かず、見ればわかることは見て判断する。

## Step 1: リポジトリ状態の自動検出

以下を **すべて** 確認し、結果を一覧で表示する:

1. **Git状態**: `git log --oneline | wc -l` でcommit数
2. **CLAUDE.md**: ファイルの有無と内容の鮮度
3. **既存Skills**: `.claude/skills/` 配下のディレクトリ一覧
4. **プロジェクト設定**: `.claude/config/` 配下のファイル一覧
5. **GitHub Project**: `.claude/config/project.yml` を読み、`gh api graphql` でProject存在確認
6. **CI/CD**: `.github/workflows/` 配下のファイル一覧
7. **テスト**: 各 `package.json` から `vitest`, `jest`, `playwright` 等を検出
8. **パッケージ構成**: `packages/` 配下のディレクトリ一覧（モノレポの場合）
9. **gh auth スコープ**: `gh auth status` で必要スコープを満たすか確認
10. **Copilot Code Review**: 自動レビュー有効化の確認ガイダンス

## Step 2: 状況判定と提案

**既存リポジトリの場合**（commit数 > 0 かつ CLAUDE.md あり）:

検出結果を表示し、以下の選択肢を提示:
1. AI駆動開発の初期設定（Skill配置・設定確認）
2. 現状の棚卸し（コード分析→Issue化）
3. CLAUDE.md の改善（検出結果と現状の差分を反映）
4. 全て

**新規リポジトリの場合**（commit数 = 0 または CLAUDE.md なし）:

以下の順でセットアップを提案:
1. CLAUDE.md の作成（技術スタック・プロジェクト概要の定義）
2. `.claude/config/project.yml` の作成（owner/repo/project情報）
3. GitHub Projectの作成
4. `.claude/config/` にプロジェクト設定ファイルを配置
5. 要件定義から始めるか確認

## Step 3: 不足項目の対応

**gh auth スコープ不足**:
```
⚠ gh auth スコープ不足: project が必要です
以下のコマンドをターミナルで実行してください:
$ gh auth refresh -h github.com -s repo,read:project,project,read:org  # project.yml の requiredScopes 参照
```

**project.yml 未作成**:
→ ユーザーにowner/repo/projectNumberを確認し、作成を提案

**GitHub Project未作成**:
→ 承認後: `gh project create --owner <owner> --title "<プロジェクト名>"`

**CLAUDE.md が現状と乖離**:
→ 検出情報（パッケージ構成、コマンド等）とCLAUDE.md記載を比較し、差分の更新を提案

## Step 4: 完了報告

```
✅ セットアップ完了

利用可能なコマンド:
  /run     — タスク実行（/run #33 or /run 自然言語指示）
  /todo    — タスク一覧（PM視点の構造化報告）
  /init    — このセットアップ（再実行可能）

専門Skill:
  docgen — YAML設計書の生成・更新（Claude自動呼び出し）
```
