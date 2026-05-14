---
name: design-figma-fetch
description: Figma URL から画面・コンポーネント・スタイル・トークンを取得し、Michinushi の設計書 YAML（screen_design / ui_component / layout_template）にマッピングする手順。designer / implementer Agent が参照する。
requires-mcp:
  - name: figma
    transport: http
    url: http://127.0.0.1:3845/sse
---

# design-figma-fetch — Figma デザイン取り込み手順

## このSkillの位置づけ

design-figma-fetch は参照 Skill である。自身がツールを実行するのではなく、以下の場面で参照される:

- **designer Agent** が画面・コンポーネント設計を行う際に、入力に Figma URL があれば最初に呼び出して設計書 YAML に取り込む
- **implementer Agent** が設計書から実装する際、既存の設計書 YAML に Figma 由来の情報があればその構造を尊重する

implementer は原則 Figma を直接見ない。designer が Figma → YAML へ変換した結果のみを参照する。

## 前提条件

- Figma Dev Mode が利用可能なプラン（Professional / Organization / Enterprise）であること
- Figma Desktop アプリで Dev Mode MCP Server が有効化され、`http://127.0.0.1:3845/sse` で待ち受けていること
- `/setup` の MCP 設定検出で `figma` が `.mcp.json` に登録済みであること
- 取得対象の Figma ファイルにユーザーが閲覧権限を持っていること

> Dev Mode を持たないプロジェクトでも、サードパーティ MCP（Framelink Figma MCP Server 等）で代替できる。setup-guide.md の「Figma MCP のセットアップ」を参照。

---

## 取得手順

### Step 1: URL のパース

Figma URL（例: `https://www.figma.com/design/<file_key>/<file_name>?node-id=<node>`）から以下を抽出する:

- `file_key`: ファイル識別子
- `node-id`: 対象ノード ID（指定がなければファイル全体）

URL に `node-id` が含まれない場合は、ユーザーに「ファイル全体を取得するか、特定フレームに絞るか」を確認する。

### Step 2: デザイン情報の取得

Figma MCP のツールを使用する:

| MCP ツール | 用途 |
|-----------|------|
| `mcp__figma__get_code` | フレーム/コンポーネントの構造とコード相当の情報 |
| `mcp__figma__get_image` | プレビュー画像（参考用） |
| `mcp__figma__get_variable_defs` | デザイントークン（Variables）の定義 |
| `mcp__figma__get_code_connect_map` | Code Connect が設定されていれば既存実装との対応 |

ノード ID 指定時は対象フレームに絞り、未指定時はファイル全体のトップレベルフレームを列挙する。

### Step 3: 設計書 YAML へのマッピング

取得した情報を Michinushi の設計書スキーマにマッピングする:

| Figma 要素 | マッピング先 | スキーマ |
|-----------|------------|---------|
| Top-level Frame（画面相当） | `screens[]` | `screen_design_schema.json` |
| Component / Component Set | `components[]` | `ui_component_schema.json` |
| Auto Layout | `layout` | `layout_template_schema.json` |
| Variables（カラー/タイポ/スペース） | `tokens` | `ui_component_schema.json` の `tokens` |

具体的なフィールド対応:

- Frame name → `screens[].name`（kebab-case に正規化）
- Component instance の階層 → `screens[].components[]`
- Auto Layout の `direction` / `gap` / `padding` → `layout.direction` / `layout.gap` / `layout.padding`
- Fill / Stroke → `tokens.colors.*`
- Text Style → `tokens.typography.*`
- Variable の `mode` → `tokens.modes[]`（ライト/ダーク等）

### Step 4: 出力先と既存ファイルとの統合

出力先:

- 画面単位: `docs/screens/<screen-name>.yml`
- コンポーネント単位: `docs/components/<component-name>.yml`
- トークン: `docs/tokens.yml`（プロジェクト共通）

既存ファイルがある場合:

1. 差分を計算（取得情報 vs 既存 YAML）
2. ユーザーに差分を提示
3. 3択で確認:
   - 上書き（Figma が真実とみなす）
   - スキップ（既存を保持）
   - 個別フィールドだけマージ（手動で選択）

> ブランドカラーや既存トークンとの衝突は `.claude/config/design.yml` の規約に従って解決する。設計書側に独自の命名がある場合は Figma 側の名前ではなくプロジェクト命名を優先することも検討する。

### Step 5: 画像・アイコンアセット

Figma 上のラスタ画像 / SVG アイコンをエクスポートする必要がある場合:

1. `mcp__figma__get_image` で画像 URL（または base64）を取得
2. プロジェクトのアセット配置規約（`design.yml` の `assets-directory` を参照、未定義なら `public/images/`）に従って保存
3. 設計書 YAML 上のコンポーネント仕様に `asset: <相対パス>` として記録

## designer / implementer への引き渡し

- **designer**: 抽出した YAML を読み、Figma にない要素（操作フロー、状態遷移、エラー時の表示、レスポンシブ挙動）を補完する。Figma はあくまで「静的なデザインの一断面」であり、振る舞いは別途設計する
- **implementer**: 完成した設計書 YAML を仕様として実装する。Figma を直接参照しない（情報源は設計書 YAML に一元化）

## 注意

- Figma の Variables が未整備のプロジェクトでは、トークン抽出ではなくスタイル値（HEX、px）をそのまま記録する。これは設計書上で `note: "Figma 上に Variables 未定義"` を付して、後でリファクタしやすくする
- 同一 Frame 内に複数の状態（hover / disabled 等）が並列に配置されている場合、各状態を `components[].states[]` に展開する
- Figma の Auto Layout は CSS Flexbox と概念が近いが完全には一致しない（例: `space-between` 相当の表現）。生成 YAML を見て、必要なら implementer が `note` を残す
- ファイル名・ノード名に日本語や絵文字が含まれる場合、出力 YAML のキーは英数 + ハイフンに正規化する（元の名前は `display-name` フィールドに保持）
