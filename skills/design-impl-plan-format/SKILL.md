---
name: design-impl-plan-format
description: 実装計画の記述フォーマット。タスクを Phase 分解し、依存関係・担当 Agent を明示するためのテンプレート。architect Agent が使う。
user-invocable: false
---

# Skill: design-impl-plan-format

## 目的

architect が分析結果を「実行可能な計画」に変換する際の標準フォーマット。

## 基本原則

1. **依存関係を可視化** — どの Phase が何の完了に依存するか明示
2. **担当 Agent を明示** — 各 Phase は designer / implementer / tester / reviewer のいずれかが担当
3. **粒度は Conductor が委譲できる単位** — 1 Phase = 1 Agent 呼び出しで完結する規模
4. **完了条件を書く** — 各 Phase の完了をどう判定するか明確に

## フォーマット

```yaml
implementation-plan:
  goal: <この実装で達成すること>
  complexity: low | medium | high

  phases:
    - id: 1
      name: <フェーズ名>
      agent: designer | implementer | tester | reviewer
      tasks:
        - <具体的な作業1>
        - <具体的な作業2>
      uses-skills:
        - <使用する Skill 名>
      dependencies: []  # 依存する phase id（複数可）
      completion-criteria: <完了条件>

    - id: 2
      name: <フェーズ名>
      agent: implementer
      tasks:
        - <作業>
      dependencies: [1]
      completion-criteria: <完了条件>

  # 全 Phase 完了後の最終確認
  final-checks:
    - <例: 全テストパス>
    - <例: 設計書と実装の整合性>
```

## Phase の典型例

### 設計フェーズ

```yaml
- id: 1
  name: UI設計
  agent: designer
  tasks:
    - 入力フォームのレイアウト決定
    - 状態遷移図作成
  uses-skills: [design-ui-patterns]
  dependencies: []
  completion-criteria: コンポーネント構成と操作フローが文書化されている
```

### 実装フェーズ

```yaml
- id: 2
  name: バックエンド実装
  agent: implementer
  tasks:
    - Prisma schema 更新
    - Server Action 追加
    - 設計書 YAML 更新
  uses-skills: [impl-coding-conventions, doc-yaml-schema]
  dependencies: [1]
  completion-criteria: 全ファイル変更完了、ローカルでビルドが通る
```

### テストフェーズ

```yaml
- id: 3
  name: 単体・E2E テスト
  agent: tester
  tasks:
    - 入力検証ケース
    - ハッピーパス E2E
  uses-skills: [test-design-principles, test-vitest, test-playwright]
  dependencies: [2]
  completion-criteria: 全テストパス、カバレッジ閾値クリア
```

### レビューフェーズ

```yaml
- id: 4
  name: コード・設計レビュー
  agent: reviewer
  tasks:
    - tech / biz 両観点でレビュー
  uses-skills: [review-tech-checklist, review-biz-checklist]
  dependencies: [2, 3]
  completion-criteria: MUST 指摘なし、または全て解消済み
```

## 注意

- **必要な Phase だけ書く** — 設計変更不要なら designer Phase は不要
- **依存関係は最小限に** — 並列実行可能な Phase は dependencies を空にする
- **過剰分割を避ける** — 1 Agent が一連で行える作業を無理に分けない
