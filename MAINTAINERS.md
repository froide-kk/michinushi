# Maintainers Guide

Michinushi のメンテナンス手順。

## リポジトリ構成

```
froide-kk/michinushi           # Skills/Agents 公開リポジトリ（public）
<org>/<your-project>           # 開発元リポジトリ（private）
  └── .claude/skills/          # ← michinushi と subtree で同期（Skills + Agent ソース）
  └── .claude/agents/          # /setup が AGENT.md からコピー登録
  └── .claude/config/          # プロジェクト固有（同期対象外）
```

## ブランチ保護

main ブランチは保護されています。

| 操作 | admin | 一般メンバー | 外部 |
|------|:-----:|:----------:|:----:|
| main に直接 push | OK | NG | NG |
| ブランチ作成 | OK | OK | Fork |
| PR 作成 | OK | OK | OK |
| PR マージ（1 approval 必要） | OK | OK | - |

admin は `subtree push` で main に直接同期できます。

## Skills の同期

### 前提

開発元リポジトリにリモートが追加されていること:

```bash
# 確認
git remote -v | grep michinushi

# 未追加の場合
git remote add michinushi git@github.com:froide-kk/michinushi.git
```

### 開発元 → michinushi（公開）

開発元で Skills を変更・コミットした後:

```bash
git subtree push --prefix=.claude/skills michinushi main
```

**注意**:
- 開発元のコミットが `.claude/skills/` 配下を含んでいれば、その差分だけが michinushi に反映される
- `.claude/config/` や他のファイルは同期されない（prefix 指定で自動除外）
- コミット履歴は skills に関連するものだけが michinushi に渡る

### michinushi → 開発元（取り込み）

外部からの PR がマージされた場合など:

```bash
git subtree pull --prefix=.claude/skills michinushi main --squash
```

`--squash` により michinushi 側の履歴は 1 コミットにまとめられる。

## Skill / Agent の追加・変更

michinushi の機能は2種類で構成される:

- **Skill** (`SKILL.md`) — ピンポイントな手順・参照材料。Claude Code が `.claude/skills/` から自動認識
- **Agent** (`AGENT.md`) — 判断と委譲を担うオーケストレーター。`/setup` が `.claude/agents/` に登録

詳細な設計指針は `process.md` の「Skill と Agent の使い分け」を参照。

### 新規 Skill 追加時のチェックリスト

- [ ] `<name>/SKILL.md` を作成（リポジトリ直下）
- [ ] frontmatter に `name`, `description` を記載
- [ ] `user-invocable` / `disable-model-invocation` を適切に設定
- [ ] プロジェクト固有のロジックが含まれていないか確認
- [ ] プロジェクト固有の設定が必要な場合、`.claude/config/` 側に分離
- [ ] どの Agent が使う Skill かを明確化（必要なら Agent 側の `使う Skill` セクションを更新）
- [ ] README.md の参照 Skill 表を更新
- [ ] `process.md` の参照 Skill 一覧を更新
- [ ] 開発元でコミット後、`subtree push` で同期

### 新規 Agent 追加時のチェックリスト

- [ ] `<name>/AGENT.md` を作成（リポジトリ直下）
- [ ] frontmatter に `name`, `description`, `tools`（任意）, `model`（任意）を記載
- [ ] 判断ロジックを Agent に書く（手順は Skill に切り出す）
- [ ] 使う Skill を `## 使う Skill` セクションに列挙
- [ ] Conductor (`run/SKILL.md`) の委譲先 Agent 表を更新
- [ ] `process.md` のエージェント構成・開発フローを更新
- [ ] README.md の Agents 表を更新
- [ ] 開発元でコミット後、`subtree push` で同期

### Skill / Agent 設計の原則

**Skill に書くもの（手順・参照材料、汎用）:**
- テスト手法・記法、レビュー観点、設計アプローチ
- OWASP Top 10 等の一般的なベストプラクティス
- 出力フォーマット、ファイル形式の規約

**Agent に書くもの（判断・委譲、隔離コンテキスト）:**
- どの Skill をいつ使うかの判断ロジック
- タスクの分解、複数 Skill の組み合わせ方
- 上流・下流 Agent との連携方針

**config に分離するもの（プロジェクト固有）:**
- 具体的な関数名、モデル名、テーブル名
- チェック項目のリスト（severity 付き）
- テスト用フィクスチャの定義
- プロジェクト固有のファイルパス

## PR レビュー方針

外部やメンバーからの PR をレビューする際の観点:

1. **汎用性** — 特定プロジェクトに依存していないか
2. **既存 Skill / Agent との整合性** — 命名規則、frontmatter 形式、出力フォーマット
3. **Skill vs Agent の役割分担** — 判断は Agent に、手順・参照は Skill に分離されているか
4. **Skill vs Config の分離** — プロジェクト固有の内容が混入していないか
5. **README / process.md 更新** — 新規 Skill / Agent の場合、対応表が更新されているか

## トラブルシューティング

### subtree push が rejected される

```
! [rejected] ... -> main (non-fast-forward)
```

michinushi 側に開発元にない変更がある場合に発生。先に pull する:

```bash
git subtree pull --prefix=.claude/skills michinushi main --squash
# コンフリクトがあれば解決してコミット
git subtree push --prefix=.claude/skills michinushi main
```

### subtree push が遅い

subtree push は全コミット履歴をスキャンするため、リポジトリが大きくなると遅くなる。
現状は許容範囲だが、将来的に問題になる場合は `git subtree split --prefix=.claude/skills --rejoin` で高速化できる。
