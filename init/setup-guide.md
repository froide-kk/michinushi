# 事前準備ガイド

AI駆動開発フレームワークを利用するための事前準備。
新規セットアップ・既存リポジトリへの参加どちらでも共通。

## 1. 必須ツール

### GitHub CLI (gh)

```bash
# macOS
brew install gh

# その他: https://cli.github.com/
```

### gh 認証

```bash
gh auth login
```

プロンプトに従い、GitHub アカウントで認証する。
- Protocol: HTTPS 推奨
- Authentication: ブラウザ認証 推奨

### gh スコープ追加

AI駆動開発では GitHub Projects API を使用するため、追加スコープが必要:

```bash
gh auth refresh -h github.com -s read:project,project
```

ブラウザが開くので承認する。

### スコープの確認

```bash
gh auth status
```

`Token scopes` に以下が含まれていることを確認:
- `repo` — Issue/PR操作
- `read:org` — Organization情報の読み取り
- `project` — GitHub Projects操作

不足がある場合:
```bash
gh auth refresh -h github.com -s repo,read:org,read:project,project
```

## 2. Copilot Code Review の有効化（リポジトリ管理者のみ）

PR作成時に自動でCopilotレビューを実行するために設定が必要。

1. リポジトリの Settings → Copilot → Code review に移動
2. 「Use custom instructions when reviewing pull requests」を ON
3. 「Create ruleset for default branch」をクリック
4. デフォルト設定のまま保存

※ この設定はリポジトリ管理者のみ実行可能。チームメンバーは不要。

## 3. Claude Code の権限設定（推奨）

頻繁に使うコマンドの許可設定。`.claude/settings.local.json` に追加:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh api *)",
      "Bash(gh issue *)",
      "Bash(gh pr *)",
      "Bash(gh project *)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git status *)",
      "Bash(ls *)"
    ]
  }
}
```

※ `settings.local.json` は `.gitignore` に含まれるため、個人設定として管理。
※ 破壊的コマンド（`git push`, `git reset`, `rm`）は含めない。承認を求められる方が安全。

## 4. プロジェクト固有の準備（該当する場合）

プロジェクトの `CLAUDE.md` に記載された開発環境セットアップを実行する。
一般的な項目:

- パッケージマネージャのインストール（pnpm, yarn 等）
- Docker Desktop のインストール・起動
- 環境変数ファイルのコピー（`.env.example` → `.env.local`）
- データベースのセットアップ
- 開発サーバーの起動確認

## 確認チェックリスト

セットアップ完了後、以下を確認:

- [ ] `gh auth status` でスコープが正しいこと
- [ ] `gh project list --owner <owner>` でProjectが見えること
- [ ] `/init` を実行して全項目が ✓ になること
