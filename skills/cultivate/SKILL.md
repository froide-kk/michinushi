---
name: cultivate
description: プロジェクト固有 config の育成セッションを起動する。reviewer Agent が PR レビュー対応時に蓄積した観点（review-feedback.yml）を、cultivator Agent に委譲してユーザーと対話形式で整理し、tech.yml / biz.yml への昇格を提案する。引数は取らない。
user-invocable: true
disable-model-invocation: true
---

# /cultivate — プロジェクト育成セッション

`.claude/config/review-feedback.yml` に蓄積された未処理 observation を、cultivator Agent に委譲してユーザーと対話的に処理する。

## 行動原則

1. **判断は委譲する** — `/cultivate` 自身は判断をせず、cultivator Agent に委譲する
2. **引数は取らない** — 蓄積された全 observation を処理対象とする
3. **対話を妨げない** — cultivator がユーザーに 4 択を問う際、`/cultivate` は出力をそのまま中継する

## 手順

### Step 1: 前提確認

`.claude/config/review-feedback.yml` の存在を確認する。

- ファイルが存在しない、または `observations` が空 → 「現在育成対象の observation はありません」と報告して終了
- ファイルが存在し observation がある → Step 2 へ

### Step 2: cultivator Agent への委譲

`cultivator` Agent を起動する。以下のコンテキストを渡す:

- 対象ファイル: `.claude/config/review-feedback.yml`
- 使う Skill: `cultivate-review`
- 出力先:
  - `docs/cultivation-log.md`（通史）
  - `.claude/config/review-feedback.yml`（処理済み observation の削除）
  - 昇格先 config（`tech.yml` / `biz.yml` 等）への更新 PR

cultivator が `cultivate-review` Skill の手順に従って各 observation について `promoted` / `rejected` / `deferred` / `delete` を確定する。

### Step 3: 結果の中継

cultivator Agent から返ってきた結果（処理件数・cultivation-log への追記件数・昇格 PR の URL 等）を **そのままユーザーに中継** する。

結果フォーマットの正本は `cultivate-review` Skill の Step 7 にある。`/cultivate` は二重定義を避けるため、ここではフォーマットを再掲しない。フォーマットの更新が必要になった場合は `cultivate-review/SKILL.md` を編集すれば本 Skill の出力にも反映される。

## 関連

- **委譲先**: `cultivator` Agent（`agents/cultivator/AGENT.md`）
- **使用 Skill**: `cultivate-review`（cultivator が呼び出す）
- **データソース**: `.claude/config/review-feedback.yml`（reviewer Agent が投入）
- **出力**: `docs/cultivation-log.md`（通史）+ 昇格先 config への更新 PR
