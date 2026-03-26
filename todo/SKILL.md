---
name: todo
description: GitHub Projectのタスク一覧をPM視点で構造化して報告する。要判断・進行中・次の候補・保留中に分類。Use when the user asks "what's the status?", "what should I work on?", or wants to see the task list.
argument-hint: "[filter: all|backlog|in-progress]"
context: fork
agent: Explore
allowed-tools: Bash(gh *), Read
---

# /todo — タスク一覧報告

PM視点で状況を評価し、ビジネスオーナーが判断できる形で構造化して報告する。

## Step 1: プロジェクト設定の読み込み

`.claude/config/project.yml` を読み、以下を取得:
- `github.owner`
- `github.repo`
- `github.projectNumber`

## Step 2: 情報収集

以下の3つを並列で取得する:

### Projectアイテム（ステータス付き）
```bash
gh api graphql -f query='
{
  organization(login: "<owner>") {
    projectV2(number: <projectNumber>) {
      title
      items(first: 50) {
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
gh issue list --repo <owner>/<repo> --state open --json number,title,labels,assignees,createdAt,updatedAt
```

### オープンPR一覧
```bash
gh pr list --repo <owner>/<repo> --state open --json number,title,headRefName,body,reviewDecision
```

## Step 3: PR↔Issue紐付け検出

各PRについて関連Issueを特定:
1. PR本文に `Closes #XX`, `Fixes #XX`, `Resolves #XX` → Issue #XX と紐付け
2. ブランチ名にIssue番号が含まれる → 紐付け
3. 紐付けなし → 「Issue未紐付きPR」として報告

## Step 4: 分類

| セクション | 判定条件 |
|-----------|---------|
| 🔴 要判断 | Issueコメントに質問・方針未決定の議論 / PRレビューで判断待ち |
| ▶ 進行中 | Projectステータスが「In Progress」/ 関連PRがオープン |
| 📥 次の候補 | Projectステータスが「Todo」 |
| 💤 保留中 | ラベルに「on-hold」/ コメントで明示的に保留 |

優先度判断基準:
- [bug] → ユーザー影響があるため優先
- 他Issueのブロッカー → 優先
- [enhancement] → 機能追加は後回し可能

## Step 5: 報告

該当がないセクションは表示しない:

```
📋 <プロジェクト名>

🔴 要判断:
  #<番号> <タイトル> — <判断が必要な理由>

▶ 進行中 (<件数>):
  #<番号> <タイトル> — <PR状態・CI結果等>

📥 次の候補 (優先度順):
  #<番号> <タイトル> [<ラベル>] — <優先する理由>

💤 保留中:
  #<番号> <タイトル> — <保留理由>
```

Issue未紐付きPRがあれば末尾に:
```
⚠ Issue未紐付きPR:
  PR #<番号> <タイトル>
```

最後にオーナーへの判断依頼やアクション提案を簡潔に添える。
