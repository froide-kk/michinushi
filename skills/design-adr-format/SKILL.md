---
name: design-adr-format
description: ADR (Architecture Decision Record) の記述形式。設計上の重要な分岐点で判断を残すためのテンプレート。architect Agent が判断系の決定を下した際に使う。
user-invocable: false
---

# Skill: design-adr-format

## 目的

設計上の重要な分岐点で **「なぜその判断をしたか」** を将来の自分・他者に伝えるためのドキュメント。

## ADR を作成すべきタイミング

- **複数の選択肢があり、トレードオフを伴う判断** をしたとき
- 採用しなかった選択肢がある程度妥当性を持つとき
- 将来「なぜこうなっているのか」を問われる可能性が高いとき

逆に以下は ADR 不要:
- 自明な選択（業界標準や唯一の合理的解）
- 一時的な対応・試験的実装

## ファイル配置

```
docs/adr/
  ADR-001-<短いタイトル>.md
  ADR-002-<短いタイトル>.md
  ...
```

連番は1から開始、ゼロパディング（3桁）。

## フォーマット

```markdown
# ADR-<番号>: <タイトル>

- **Status**: Proposed | Accepted | Deprecated | Superseded by ADR-XXX
- **Date**: YYYY-MM-DD
- **Deciders**: <意思決定者>

## Context

判断が必要となった背景。何の問題を解こうとしているか。

## Decision

採用した決定。明確に1つの選択を述べる。

## Alternatives Considered

検討した他の選択肢:

### Option A: <案名>
- 概要
- Pros
- Cons
- 不採用理由

### Option B: <案名>
- 概要
- Pros
- Cons
- 不採用理由

## Consequences

この決定の結果として何が起きるか:

- **Positive**: 得られるもの
- **Negative**: 諦めたもの・抱えるリスク
- **Neutral**: 副作用・付随する変更

## Related

- 関連する ADR、設計書、Issue
```

## ADR と設計書の関係

設計書の `metadata.relatedDecisions` に該当 ADR 番号を記録する:

```yaml
metadata:
  relatedDecisions:
    - ADR-003
    - ADR-007
```

これにより、後で設計書を変更しようとした際に過去の判断との整合性を確認できる。

## 注意

- **短く書く** — 1 ADR は1〜2画面で読める量
- **判断の根拠が将来検証可能であること** — 「みんなが言うから」ではなく、固有の理由を書く
- **Deprecated 時は理由と置き換え先を明記** — 古い ADR を消さず、Status を更新する
