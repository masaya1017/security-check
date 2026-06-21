# CLAUDE.md — プロジェクト概要（Claude Code 向け）

## プロジェクト概要

CISAガイドラインに準拠したセキュリティ分析を行うClaude Codeマルチエージェントシステム。
外部SDKは使わず、`claude -p` CLIでサブエージェントを起動する構成。

## ディレクトリ構成

```
security/
├── main.py                  # Orchestrator — ユーザー入力・エージェント起動・確認処理
├── run.sh                   # 起動スクリプト
├── agents/
│   ├── reporter_skill.md    # Sub-agent 1 の指示書（clone・スキャン・MDレポート出力）
│   └── fixer_skill.md       # Sub-agent 2 の指示書（MDを読んでCRITICAL脆弱性を修復）
└── output/                  # 実行時に生成されるレポート置き場
```

## エージェント構成

### Orchestrator（`main.py`）
- ユーザーからリポジトリURLを取得
- `claude -p <reporter_skill.md + タスク>` でSub-agent 1を起動
- Sub-agent 1が出力するJSONメタデータ（`scan_metadata_*.json`）を読み、件数・パスを把握
- CRITICAL件数が1件以上ならユーザーに修復確認（y/n）
- 承認されたら `claude -p <fixer_skill.md + タスク>` でSub-agent 2を起動
- Sub-agent 2が出力するJSONメタデータ（`fix_metadata_*.json`）を読み、結果を表示

### Sub-agent 1: Reporter（`agents/reporter_skill.md`）
- 起動方法: `claude -p <スキルファイル内容 + タスク> --dangerously-skip-permissions`
- 使用ツール: `Bash`（git clone）、`Read`・`Glob`（ファイル調査）、`Write`（レポート保存）
- 出力ファイル（Orchestratorへの完了通知）:
  - `output/security_report_TIMESTAMP.md` — Markdownセキュリティレポート
  - `output/scan_metadata_TIMESTAMP.json` — 件数・パス情報

### Sub-agent 2: Fixer（`agents/fixer_skill.md`）
- 起動方法: `claude -p <スキルファイル内容 + タスク> --dangerously-skip-permissions`
- 入力: OrchestratorからMDレポートパス・リポジトリパスを受け取る
- 使用ツール: `Read`（MDレポート・ソースファイル読み込み）、`Edit`・`Write`（修復適用）
- 出力ファイル（Orchestratorへの完了通知）:
  - `output/fix_report_TIMESTAMP.md` — 修復内容のMarkdownレポート
  - `output/fix_metadata_TIMESTAMP.json` — 修復件数情報

## エージェント間のデータフロー

```
Orchestrator
  │ repo_url（ユーザー入力）
  │ repo_path / report_path / scan_metadata_path（パスを生成）
  ▼
Reporter
  │ → output/security_report_TIMESTAMP.md  （MDレポート）
  │ → output/scan_metadata_TIMESTAMP.json  （完了通知）
  ▼
Orchestrator（JSONを読む）
  │ critical_count / repo_path / report_path を取得
  │ ユーザー確認
  ▼
Fixer
  │ report_path（MDレポートのパス）を受け取る
  │ → MDファイルを読んでCRITICAL脆弱性を把握
  │ → ソースファイルを直接修正
  │ → output/fix_report_TIMESTAMP.md
  │ → output/fix_metadata_TIMESTAMP.json （完了通知）
  ▼
Orchestrator（JSONを読む）
  │ fixed_count を取得して結果を表示
```

## スキルファイルの構成

スキルファイル（`.md`）はエージェントの「指示書」。
Orchestratorが内容を読み込み、タスク情報と結合してプロンプトとして `claude -p` に渡す。

```python
# main.py での使用例
skill = skill_path.read_text()
prompt = f"{skill}\n\n---\n## 実行タスク\n\n{task}"
subprocess.run(["claude", "-p", prompt, "--dangerously-skip-permissions"])
```

スキルファイルを編集することで、チェック基準・レポートフォーマット・修復ロジックを変更できる。

## 出力ファイル一覧

| ファイル名 | 生成者 | 内容 |
|-----------|--------|------|
| `security_report_TIMESTAMP.html` | Reporter | CISAセキュリティ分析レポート（ブラウザで表示） |
| `scan_metadata_TIMESTAMP.json` | Reporter | 件数・パス（Orchestratorが読む） |
| `fix_report_TIMESTAMP.html` | Fixer | 修復内容レポート（ブラウザで表示） |
| `fix_metadata_TIMESTAMP.json` | Fixer | 修復件数（Orchestratorが読む） |

## 実行方法

```bash
cd /home/masaya/security
python3 main.py
```

## 注意事項

- `claude` CLIが `PATH` に存在すること（Claude Code CLI）
- インターネット接続と `git` コマンドが利用可能なこと
- `--dangerously-skip-permissions` は非対話バッチ実行のために使用
- クローンされたリポジトリは `repos/repo_TIMESTAMP/` に保存される
