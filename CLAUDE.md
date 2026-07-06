# CLAUDE.md — プロジェクト概要（Claude Code 向け）

## プロジェクト概要

CISAガイドラインに準拠したセキュリティ分析を行うClaude Codeマルチエージェントシステム。
外部SDKは使わず、Claude Code の `Agent` ツールでサブエージェントを起動する構成。

## ディレクトリ構成

```
security/
├── .claude/
│   └── commands/
│       └── security.md           # Orchestrator — /security スラッシュコマンドの実装
├── agents/
│   ├── reporter_skill.md         # Sub-agent 1a の指示書（CISA: clone・スキャン・HTMLレポート出力）
│   ├── owasp_reporter_skill.md   # Sub-agent 1b の指示書（OWASP Top 10: スキャン・HTMLレポート出力）
│   └── fixer_skill.md            # Sub-agent 2 の指示書（HTMLを読んでCRITICAL脆弱性を修復）
└── output/                       # 実行時に生成されるレポート置き場
```

## エージェント構成

### Orchestrator（`.claude/commands/security.md`）
- Claude Code のスラッシュコマンド `/security <URL>` として起動
- `Bash` ツールでタイムスタンプを取得し、出力パスを確定する
- **フレームワーク選択** をユーザーに提示（CISA / OWASP / 両方）
- 選択に応じて Sub-agent 1a（CISA Reporter）、Sub-agent 1b（OWASP Reporter）、または両方を順番に起動
- 各スキャン完了後にメタデータを `Read` して件数・パスを表示
- CRITICAL件数合計が1件以上ならユーザーに修復確認（y/n）
- 承認されたら `Agent` ツールで Sub-agent 2（Fixer）を起動
- Sub-agent 2が出力するJSONメタデータを `Read` ツールで読み、結果を表示

### Sub-agent 1a: CISA Reporter（`agents/reporter_skill.md`）
- 起動方法: Orchestratorが `Agent` ツールで起動し、スキルファイルを `Read` ツールで読み込むよう指示する
- 使用ツール: `Bash`（git clone）、`Read`・`Glob`（ファイル調査）、`Write`（レポート保存）
- 出力ファイル（Orchestratorへの完了通知）:
  - `output/security_report_TIMESTAMP.html` — CISAセキュリティ分析レポート（HTML）
  - `output/scan_metadata_TIMESTAMP.json` — 件数・パス情報

### Sub-agent 1b: OWASP Reporter（`agents/owasp_reporter_skill.md`）
- 起動方法: Orchestratorが `Agent` ツールで起動し、スキルファイルを `Read` ツールで読み込むよう指示する
- チェック基準: OWASP Top 10 2021（A01〜A10）
- REPO_PATH が既にクローン済みならクローンをスキップしてスキャンのみ実行
- 出力ファイル（Orchestratorへの完了通知）:
  - `output/owasp_report_TIMESTAMP.html` — OWASP Top 10 セキュリティ分析レポート（HTML）
  - `output/owasp_metadata_TIMESTAMP.json` — 件数・OWASPカテゴリ別カウント・パス情報

### Sub-agent 2: Fixer（`agents/fixer_skill.md`）
- 起動方法: Orchestratorが `Agent` ツールで起動し、スキルファイルを `Read` ツールで読み込むよう指示する
- 入力: OrchestratorからHTMLレポートパス（1つまたは2つ）・リポジトリパスを受け取る
- 両フレームワーク実行時は重複脆弱性（同一ファイル・同一行）を一度だけ修復する
- 使用ツール: `Read`（HTMLレポート・ソースファイル読み込み）、`Edit`・`Write`（修復適用）
- 出力ファイル（Orchestratorへの完了通知）:
  - `output/fix_report_TIMESTAMP.html` — 修復内容レポート（HTML）
  - `output/fix_metadata_TIMESTAMP.json` — 修復件数情報

## エージェント間のデータフロー

```
Orchestrator（/security <URL>）
  │ repo_url（$ARGS または ユーザー入力）
  │ タイムスタンプ取得 → パス群を確定
  │ フレームワーク選択をユーザーに提示 [1] CISA / [2] OWASP / [3] 両方
  │
  ├─（選択1 or 3）──────────────────────────────────────────────┐
  │  CISA Reporter（Agent ツールで起動）                          │
  │  │ reporter_skill.md を Read で読み込み、手順を実行           │
  │  │ → output/security_report_TIMESTAMP.html                   │
  │  │ → output/scan_metadata_TIMESTAMP.json（完了通知）          │
  │  ▼                                                            │
  │  Orchestrator（CISA_META を Read し件数を表示）               │
  │                                                               │
  ├─（選択2 or 3）──────────────────────────────────────────────┘
  │  OWASP Reporter（Agent ツールで起動）
  │  │ owasp_reporter_skill.md を Read で読み込み、手順を実行
  │  │ → output/owasp_report_TIMESTAMP.html
  │  │ → output/owasp_metadata_TIMESTAMP.json（完了通知）
  │  ▼
  │  Orchestrator（OWASP_META を Read しカテゴリ別件数を表示）
  │
  │  CRITICAL合計を集計 → ユーザー確認（y/n）
  ▼
Fixer（Agent ツールで起動）
  │ fixer_skill.md を Read で読み込み、手順を実行
  │ → 1つまたは2つのHTMLレポートから CRITICAL脆弱性を把握
  │ → ソースファイルを直接修正（重複は一度だけ）
  │ → output/fix_report_TIMESTAMP.html
  │ → output/fix_metadata_TIMESTAMP.json     （完了通知）
  ▼
Orchestrator（FIX_META を Read で読む）
  │ fixed_count / skipped_count を取得して結果を表示
```

## スキルファイルの構成

スキルファイル（`.md`）はサブエージェントの「指示書」。
Orchestratorがサブエージェントを起動する際、`Read` ツールでスキルファイルを読み込んでから実行するよう指示する。

```
# security.md（Orchestratorの起動コード）での使用例
Agent ツールで以下のプロンプトをサブエージェントに渡す:

  まず Read ツールで AGENTS_DIR/reporter_skill.md を読み込み、（AGENTS_DIR は STEP 0 で pwd から求めたプロジェクトルート配下の agents/）
  そこに書かれた手順をすべて実行してください。
  
  ## 実行タスク
  リポジトリURL      : <URL>
  クローン先         : <REPO_PATH>
  HTMLレポート出力先 : <REPORT_PATH>
  メタデータ出力先   : <SCAN_META>
```

スキルファイルを編集することで、チェック基準・レポートフォーマット・修復ロジックを変更できる。

## 出力ファイル一覧

| ファイル名 | 生成者 | 内容 |
|-----------|--------|------|
| `security_report_TIMESTAMP.html` | CISA Reporter | CISAセキュリティ分析レポート（ブラウザで表示） |
| `scan_metadata_TIMESTAMP.json` | CISA Reporter | 件数・パス（Orchestratorが読む） |
| `owasp_report_TIMESTAMP.html` | OWASP Reporter | OWASP Top 10 分析レポート（ブラウザで表示） |
| `owasp_metadata_TIMESTAMP.json` | OWASP Reporter | 件数・カテゴリ別カウント（Orchestratorが読む） |
| `fix_report_TIMESTAMP.html` | Fixer | 修復内容レポート（ブラウザで表示） |
| `fix_metadata_TIMESTAMP.json` | Fixer | 修復件数（Orchestratorが読む） |

## 実行方法

Claude Code のチャット上で以下のスラッシュコマンドを実行する:

```
/security https://github.com/example/target-repo
```

URLを省略した場合はOrchestratorがURLの入力を求める。

## 注意事項

- Claude Code（`claude` CLI）が起動済みであること
- インターネット接続と `git` コマンドが利用可能なこと
- クローンされたリポジトリは `repos/repo_TIMESTAMP/` に保存される
