# GitHub 操作リファレンス

Conductor が `gh` / `git` コマンドを実行する際の参照情報。

## Issue 操作

```bash
# 読み取り
gh issue view <番号> --repo <owner>/<repo>

# 作成
gh issue create --repo <owner>/<repo> --title "<タイトル>" --body "<本文>" --label "<ラベル>"

# Project にアイテム追加
gh project item-add <projectNumber> --owner <owner> --url <Issue URL>
```

## PR 操作

```bash
# 作成
gh pr create --base <baseBranch> --title "<タイトル>" --body "<本文>"

# レビューコメント取得
gh api repos/<owner>/<repo>/pulls/<PR番号>/comments
```

## レビューコメント対応

```bash
# スレッドID取得（ページネーション必須: hasNextPage が true なら after: endCursor で次ページ取得）
gh api graphql -f query='
{
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <PR番号>) {
      reviewThreads(first: 20) {
        nodes { id comments(first: 1) { nodes { body author { login } } } }
        pageInfo { hasNextPage endCursor }
      }
    }
  }
}'

# スレッドにリプライ
gh api graphql -f query='
mutation {
  addPullRequestReviewThreadReply(input: {
    pullRequestReviewThreadId: "<threadId>",
    body: "<リプライ本文>"
  }) { comment { id } }
}'
```

## Project ステータス取得

```bash
# owner が org の場合（user の場合は organization を user に置換）
gh api graphql -f query='
{
  organization(login: "<owner>") {
    projectV2(number: <projectNumber>) {
      items(first: 50) {
        nodes {
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
          content {
            ... on Issue { number title state }
          }
        }
        pageInfo { hasNextPage endCursor }
      }
    }
  }
}'
# 50件超の場合は pageInfo.hasNextPage を確認し、after: endCursor で次ページを取得する
```

## Issue / Project 運用ルール

### ステータスフロー

```
Triage → Todo → In Progress → In Review → Done
```

| ステータス | 意味 | 遷移タイミング |
|-----------|------|--------------|
| Triage | 要望・検討中 | 要望を受けた時点 |
| Todo | 具体化済み・着手待ち | 要件が明確になった時点 |
| In Progress | 作業中 | ブランチ作成・実装開始時 |
| In Review | PR作成済み・レビュー待ち | PR作成時 |
| Done | マージ済み | PRマージ時 |

### Issue 管理フロー

**ふわっとした要望が来た場合**:
1. Issue 作成（ラベル: `bug` / `enhancement` / `infra`）
2. Project に追加、ステータス: `Triage`

**要件が具体化された場合**:
1. 既存 Issue の本文を更新
2. 必要なら sub-issue を作成
3. ステータスを `Todo` に変更

**作業開始時**:
1. ステータスを `In Progress` に変更
2. ブランチ作成

**PR 作成時**:
1. PR 本文に `Closes #<Issue番号>` を含める
2. ステータスを `In Review` に変更

### Issue 作成の基準

- **作成する**: ユーザーの要望・バグ報告・改善提案・派生タスク
- **作成しない**: 単純な質問・一時的な相談（会話で完結するもの）
