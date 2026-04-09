---
name: todo
description: タスク一覧をPM視点で構造化して報告する。GitHub Projectまたはローカルファイルからタスクを取得し、要判断・進行中・次の候補・保留中に分類。Use when the user asks "what's the status?", "what should I work on?", or wants to see the task list.
argument-hint: "[filter: all|backlog|in-progress]"
context: fork
agent: Explore
allowed-tools: Bash(gh api *, gh issue list *, gh pr list *), Read
---

# /todo — タスク一覧報告

PM視点で状況を評価し、ビジネスオーナーが判断できる形で構造化して報告する。

## Step 1: プロジェクト設定の読み込み

`.claude/config/project.yml` を読み、管理モードを判定する:
- `mode.tasks` を確認（未定義なら `github` として扱う）
- `tasks: github` → Step 2A へ
- `tasks: local` → Step 2B へ

## Step 2A: 情報収集（GitHub モード）

`.claude/config/project.yml` から以下を取得:
- `github.owner`
- `github.repo`
- `github.projectNumber`

以下の3つを並列で取得する:

### Projectアイテム（ステータス付き）
ownerがorgの場合。userの場合は `organization` を `user` に置換。
```bash
gh api graphql -f query='
{
  organization(login: "<owner>") {
    projectV2(number: <projectNumber>) {
      title
      items(first: 50) {
        pageInfo { hasNextPage endCursor }
        nodes {
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
          content {
            ... on Issue {
              number
              title
              state
              labels(first: 5) { nodes { name } }
              comments(last: 3) { nodes { body author { login } } }
            }
          }
        }
      }
    }
  }
}'
```

### オープンIssue一覧
```bash
gh issue list --repo <owner>/<repo> --state open --limit 200 --json number,title,labels,assignees,createdAt,updatedAt
```

### オープンPR一覧
```bash
gh pr list --repo <owner>/<repo> --state open --limit 200 --json number,title,headRefName,body,reviewDecision
```

→ Step 3A へ

## Step 2B: 情報収集（ローカルモード）

`docs/tasks.md` を読み込み、各セクション（Triage / Todo / In Progress / In Review / Done）のタスクを解析する。

→ Step 3B へ

## Step 3A: PR↔Issue紐付け検出（GitHub モードのみ）

各PRについて関連Issueを特定:
1. PR本文に `Closes #XX`, `Fixes #XX`, `Resolves #XX` → Issue #XX と紐付け
2. ブランチ名にIssue番号が含まれる → 紐付け
3. 紐付けなし → 「Issue未紐付きPR」として報告

## Step 3B: タスク解析（ローカルモードのみ）

`docs/tasks.md` のセクション構造から各タスクのステータスを判定する。

## Step 4: 分類

両モード共通で、以下の分類に振り分ける:

| セクション | GitHub モード判定条件 | ローカルモード判定条件 |
|-----------|---------------------|---------------------|
| 🔴 要判断 | Projectステータスが「Triage」/ コメントに方針未決定の議論 | `## Triage` セクションのタスク |
| ▶ 進行中 | Projectステータスが「In Progress」または「In Review」 | `## In Progress` / `## In Review` セクションのタスク |
| 📥 次の候補 | Projectステータスが「Todo」 | `## Todo` セクションのタスク |
| 💤 保留中 | ラベルに「on-hold」/ コメントで明示的に保留 | タスク行に `[on-hold]` を含む |

優先度判断基準:
- [bug] → ユーザー影響があるため優先
- 他タスクのブロッカー → 優先
- [enhancement] → 機能追加は後回し可能

## Step 5: 報告

該当がないセクションは表示しない:

```
📋 <プロジェクト名>

🔴 要判断:
  #<番号> <タイトル> — <判断が必要な理由>

▶ 進行中 (<件数>):
  #<番号> <タイトル> — <状態の補足>

📥 次の候補 (優先度順):
  #<番号> <タイトル> — <優先する理由>

💤 保留中:
  #<番号> <タイトル> — <保留理由>
```

GitHub モードで Issue 未紐付き PR があれば末尾に:
```
⚠ Issue未紐付きPR:
  PR #<番号> <タイトル>
```

最後にオーナーへの判断依頼やアクション提案を簡潔に添える。
