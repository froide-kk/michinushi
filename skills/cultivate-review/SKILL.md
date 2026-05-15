---
name: cultivate-review
description: review-feedback.yml に蓄積された未処理 observation を起点に、reviewer 領域の観点を育成する手順。cultivator Agent が使う。ユーザーと対話しながら promoted / rejected / deferred / delete を確定し、cultivation-log への通史追記と昇格先 config への更新 PR 提案を行う。
user-invocable: false
---

# Skill: cultivate-review

## 目的

各 PR で reviewer Agent が `.claude/config/review-feedback.yml` に投入した observation を、ユーザーとの対話を通じて整理し、`tech.yml` / `biz.yml` への昇格判断を行う。reviewer がプロジェクト固有のドメインに育っていくループの中核を担う。

## このSkillの位置づけ

cultivate-review は参照 Skill である。cultivator Agent が呼び出し、本 Skill の手順に従って育成セッションを進める。判断はすべてユーザーに委ね、cultivator はファシリテーションに徹する。

## 前提条件

- `.claude/config/review-feedback.yml` が存在する（無ければ「育成対象なし」を報告して終了）
- `docs/cultivation-log.md` が存在する（無ければ初回として新規作成）
- 昇格先 config（`.claude/config/tech.yml` / `biz.yml` 等）が存在する（無ければユーザーに作成可否を確認）

## データモデル

### review-feedback.yml の構造

```yaml
schema-version: 1
observations:
  - id: <category>-<slug>-<hash6>
    summary: <観点の内容>
    category: security | type-safety | docs | consistency | perf | test | other
    severity: MUST | SHOULD | NIT
    occurrence-count: <回数>
    first-seen: <ISO 8601 日付>
    last-seen: <ISO 8601 日付>
    detections:
      - pr: <PR 番号>
        path: <ファイルパス>
        line: <行番号>
        date: <ISO 8601 日付>
        snippet: <該当箇所の抜粋>
        comment-url: <PR コメント URL（あれば）>
    related: [<related observation id>, ...]
```

#### id 採番ルール

- **形式**: `<category>-<slug>-<hash6>`（例: `docs-tree-syntax-a3f2k1`）
- **slug**: summary を英数小文字 + ハイフンに正規化（観点が一意に識別できる短いフレーズ）
- **hash6**: ランダムな 6 文字（base32 小文字、`a-z2-7`）を生成。並行 PR で同じ `category-slug` の新規 observation が衝突しないよう、連番ではなく ID 内に独立した識別子を持たせる

##### 既存判定の順序

新規検出時、reviewer Agent は以下の順で判定する:

1. **同じ `category-slug` の組み合わせが `review-feedback.yml` にあるか?**
   - **Yes** → 既存 observation の `detections` に append し `occurrence-count` を +1。新規 id は採番しない
   - **No** → 次へ
2. **同じ `category-slug` が `cultivation-log.md` にあるか?**（過去に `promoted` / `rejected` 済み）
   - **Yes** → **新しい hash6 を生成して新規 observation として yml に追加**。過去の判断は古いコンテキストでなされたものなので再議論する。元 log エントリは書き換えない（append-only）
   - **No** → 次へ
3. **新規 observation を作成**: `<category>-<slug>-<新規 hash6>` で yml に追加

##### 衝突回避

- hash6 はランダム生成のため、同じ `category-slug` で複数 PR が並行投入しても **異なる id が振られる**
- 結果として cultivate 実行時に「**同じ category-slug を持つ複数 observation が yml に並んでいる**」状態が起こり得る。これは Step 2 のグルーピングで自然に検出されるため、cultivator がユーザーに「これらは同じ観点ですか?」と統合判断を問う
- 統合時は新しい hash6 を 1 つ選び、他方の detections をマージして occurrence-count を合算、余った id は削除

##### 削除済み id の不再利用

- `promoted` / `rejected` / `delete` で yml から削除された hash6 は **再利用しない**（cultivation-log に参照として残るため）
- hash6 はランダム生成なので、削除済み id を意識せず新規生成すれば事実上衝突しない（base32 6 文字 = 約 10 億通り）

### cultivation-log.md の構造

時系列降順。`Promoted` と `Rejected` のエントリを記録する（`deferred` / `delete` は記録しない）。フォーマットは本ファイル下部の「cultivation-log エントリテンプレート」を参照。

## 手順

### Step 1: review-feedback.yml の読み込み

`.claude/config/review-feedback.yml` を読み、`observations[]` を取得する。

- ファイル不在 or `observations` 空 → 「今回育成対象なし」を報告して終了

### Step 2: 全体像の提示

observation を以下の軸で集計し、ユーザーに概要を見せる:

```
📊 蓄積された observation: N件
  カテゴリ別:
    - docs: 4件
    - consistency: 2件
    - type-safety: 1件
    - test: 1件
  severity 別:
    - MUST: 1件 / SHOULD: 3件 / NIT: 4件
  occurrence-count 別:
    - 3回以上: 2件（昇格候補）
    - 2回: 3件
    - 1回: 3件（判断保留候補）
```

### Step 3: 観点ごとの対話

observations を **occurrence-count の降順** で順番に処理する。各 observation について:

**3-1. observation の内容を提示**:

```
🔍 観点 (N/M):
  id: docs-tree-syntax-a3f2k1
  summary: ツリー表記で同階層に複数要素ある場合、中間は ├── 最後は └── を使う
  category: docs
  severity: NIT
  occurrence-count: 2
  detections:
    - PR #6 MAINTAINERS.md:14 (2026-05-13)
      comment: https://github.com/.../discussion_r3231203723
    - PR #11 README.md:24 (2026-05-15)
```

**3-2. ユーザーに 4 択で確認**:

```
この観点をどうしますか?
  1. promoted — 昇格先 config に追加（次に昇格先を聞きます）
  2. rejected — 却下（次に理由を聞きます）
  3. deferred — 次回の cultivate で再議論（yml に残す）
  4. delete   — 誤検出 / もう価値が無い（yml から削除、log には残さない）
```

**3-3. 選択別の追加ヒアリング**:

| 選択 | 追加ヒアリング |
|---|---|
| `promoted` | 昇格先（`tech.yml` / `biz.yml` / 他 config）と rationale（採用根拠） |
| `rejected` | rationale（却下理由） |
| `deferred` | 不要（yml に残すだけ） |
| `delete` | 不要（yml から削除、log にも残さない） |

**昇格先の提示と新規 config 作成**:

`promoted` 選択時は、`.claude/config/` 配下に存在する **すべての yml ファイルを動的に列挙** してユーザーに提示する（`tech.yml` / `biz.yml` だけでなく `security.yml` / `ui.yml` / `design.yml` など、リポジトリの状態に応じてその場で列挙される）。

```bash
ls .claude/config/*.yml 2>/dev/null | grep -v '^\.claude/config/project\.yml$' | grep -v 'review-feedback\.yml$'
```

> `project.yml`（管理モード設定）と `review-feedback.yml`（cultivator 自身が読み書きするバッファ）は昇格先から除外する。

提示例（実行時のディレクトリ状態に応じて自動列挙）:

```
昇格先を選んでください:
  1. .claude/config/tech.yml (既存) — 技術レビューのチェック項目
  2. .claude/config/biz.yml (既存) — 業務レビューのチェック項目
  3. .claude/config/security.yml (既存) — セキュリティテストのチェック項目
  4. .claude/config/ui.yml (既存) — UI 方針
  5. .claude/config/design.yml (既存) — 設計書規約
  6. その他（新規 config を作成）
```

observation の `category` から **推奨昇格先** を提示しても良い（例: `category: security` なら security.yml を推奨）。ただし最終判断はユーザー。

「その他」が選ばれた場合、新規 config ファイル名（例: `.claude/config/coding-style.yml`）をユーザーから聞き取り、以下の最小テンプレートで新規作成する:

```yaml
# .claude/config/<file-name>.yml
# プロジェクト固有のチェック項目。cultivator Agent が育成提案で追記する。

schema-version: 1
items:
  - id: <category>-<slug>-<hash6>     # review-feedback.yml の id と整合させる
    summary: <観点>
    category: <カテゴリ>
    severity: MUST | SHOULD | NIT
    rationale: <なぜ採用したか>
    promoted-from: <observation id>
    promoted-at: <ISO 8601 日付>
```

既存の config に追記する場合も、上記 `items[]` のフォーマットに従って末尾に append する。

**3-4. 中断オプション**:

各 observation の処理後、ユーザーに「続けますか? / 一旦止めますか?」を確認する選択肢を提供する。`stop` を選択された場合、未処理の observation はすべて `deferred` 扱いで Step 4 に進む。

### Step 4: cultivation-log.md への追記

`promoted` / `rejected` になった observation について `docs/cultivation-log.md` に追記する。

時系列降順を維持するため、**ファイルの先頭近く**（タイトルとイントロの直下）に追記する。具体的なルール:

- **同日の既存セクション (`## YYYY-MM-DD`) が既にあれば、そのセクションの末尾に追記** する（新規セクションを作らない）
- **同日のセクションが無ければ**、タイトル / イントロ直下に **新規セクションを挿入** する
- 結果として、同じ日に複数回 `/cultivate` が実行されても `## YYYY-MM-DD` セクションは 1 つだけ存在し、その中にエントリが時系列に並ぶ
- 異なる日同士は降順（新しい日付が上）

> **「append-only」の意味**: 既存エントリ（過去の `Promoted` / `Rejected` 記述）を **書き換えない・消さない** という意味。新エントリの挿入位置は時系列順を保つために先頭側になるが、これは「既存を書き換える」操作ではないため append-only 原則と矛盾しない。

#### cultivation-log エントリテンプレート

```markdown
## YYYY-MM-DD

### Promoted: <短いタイトル>
- **Target**: `.claude/config/tech.yml` (category: <カテゴリ>)
- **Severity**: <重大度>
- **Trigger**: <occurrence-count>回出現
- **Rationale**: <ユーザーから聞いた採用根拠>
- **Detections**:
  - PR #6 MAINTAINERS.md:14 (YYYY-MM-DD) — [comment](<comment-url>)
  - PR #11 README.md:24 (YYYY-MM-DD) — [comment](<comment-url>)
- **Observation**: `<observation id>`

### Rejected: <短いタイトル>
- **Source PR(s)**: #8
- **Reason**: <ユーザーから聞いた却下理由>
- **Observation**: `<observation id>`
```

cultivation-log.md が存在しない場合、以下のヘッダで新規作成する:

```markdown
# Cultivation Log

このプロジェクトの reviewer / 各 Agent が学習し進化してきた通史。
cultivator Agent によって観点が昇格・却下されるたびに追記される。
詳細データは `.claude/config/review-feedback.yml` を参照。

---

```

### Step 5: review-feedback.yml の更新

処理が確定した observation を yml から削除する:

| decision | yml の扱い |
|---|---|
| `promoted` | 削除 |
| `rejected` | 削除 |
| `delete` | 削除 |
| `deferred` | **残す**（次回の cultivate で再議論される） |

未処理（Step 3 で中断された）observation も `deferred` 扱いで残す。

### Step 6: 昇格 PR の提案

`promoted` になった observation について、昇格先 config の更新 PR を別途作成する。

複数の `promoted` を 1 つの PR にまとめても良い（カテゴリ別や昇格先別にグルーピングする）。

PR フォーマット:
- **branch**: `cultivate/<YYYY-MM-DD>-<summary-slug>`
- **title**: `cultivate: <昇格内容のサマリ>`
- **body**:
  ```
  ## Summary
  <観点の追加内容>

  ## Promoted observations
  - `<observation id>`: <summary>
    - Trigger: <occurrence-count>回出現 (PR #6, #11)
    - Rationale: <採用根拠>

  ## Changes
  - `.claude/config/tech.yml` に <カテゴリ> カテゴリの新規項目を追加

  Co-authored-by: cultivator Agent
  ```

PR 作成は `gh pr create` を使う。base は `main`。

### Step 7: 結果報告

```
🌱 育成セッション完了

処理した observation: N件
  - promoted: <数>件
  - rejected: <数>件
  - deferred: <数>件（次回 cultivate で再議論）
  - delete: <数>件

cultivation-log への追記: <数>件
review-feedback.yml からの削除: <数>件（deferred <数>件は残存）

提案 PR:
  - <昇格先>: https://github.com/.../pull/<番号>
```

## 重複検出

新規の observation が、既存の observation と意味的に重複していると Step 2 のグルーピングで気づくことがある。その場合、ユーザーに「これは同じ観点ですか?」と問い、統合するか別物として扱うか判断する。統合する場合、`detections` を片方の observation にマージし、occurrence-count を合算する。

## ファイル不在時の対応

| 不在ファイル | 対応 |
|---|---|
| `.claude/config/review-feedback.yml` | 「今回育成対象なし」を報告して終了（エラーではない） |
| `docs/cultivation-log.md` | 上記テンプレートで新規作成 |
| 昇格先 config (`tech.yml` / `biz.yml` 等) | ユーザーに「作成して良いか」を確認。承認後に新規作成 |

## 注意

- **判断を持たない**: cultivator が「私はこれを promoted すべきだと思います」と言わない。観点と detections を提示し、ユーザーが決める
- **append-only**: cultivation-log の既存エントリは書き換えない。判断が変わった場合は新しいエントリとして追記する
- **rationale は必ず聞く**: `promoted` / `rejected` の場合、根拠（採用根拠 / 却下理由）を必ずユーザーから引き出す（空のままにしない）
- **deferred の蓄積**: deferred ばかりが yml に溜まる場合、ユーザーに「これらを定期的に見直しますか?」と問う。永久に判断保留が続くのは健全ではない
