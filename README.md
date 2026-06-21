# CISA Security Analysis — Claude Code Multi-Agent System

CISAガイドラインに基づいたセキュリティ分析を自動化するClaude Codeマルチエージェントシステムです。
GitリポジトリのURLを入力するだけで、セキュリティスキャンから脆弱性の自動修復まで実行します。

## 特徴

- **Claude Code完結** — 外部SDKやAPIキー不要。`claude` CLIのみで動作
- **CISAガイドライン準拠** — CRITICAL/HIGH/MEDIUM/LOW の4段階で脆弱性を分類
- **マルチエージェント構成** — スキャン・修復を専用エージェントが分担
- **ユーザー確認フロー** — 自動修復の前に必ず確認プロンプトを表示
- **MDファイルベース連携** — エージェント間はMarkdownレポートとJSONメタデータでデータを受け渡す

## エージェント構成

```
Orchestrator（main.py）
│
├─► Sub-agent 1: Reporter   ← agents/reporter_skill.md を参照
│       git clone → CISAスキャン → MDレポート出力
│       完了通知（JSON）: report_path / 件数
│
│   ⚠️ CRITICAL N件検出 → "修復しますか？ [y/n]"（ユーザー確認）
│
└─► Sub-agent 2: Fixer      ← agents/fixer_skill.md を参照
        MDレポートを読む → CRITICAL脆弱性を修復 → 修復レポート出力
        完了通知（JSON）: fix_report_path / 修復件数
```

## 前提条件

| 必要なもの | 確認方法 |
|-----------|---------|
| Claude Code CLI | `claude --version` |
| Python 3.x | `python3 --version` |
| Git | `git --version` |
| インターネット接続 | — |

## ファイル構成

```
security/
├── main.py                  # Orchestrator
├── run.sh                   # 起動スクリプト
├── CLAUDE.md                # Claude Code向け技術仕様
├── README.md                # このファイル
├── agents/
│   ├── reporter_skill.md    # Sub-agent 1 の指示書
│   └── fixer_skill.md       # Sub-agent 2 の指示書
└── output/                  # レポート出力先（実行時に生成）
```

## 使い方

```bash
cd /home/masaya/security
python3 main.py
```

実行すると対話形式で進みます:

```
╔══════════════════════════════════════════════════════════════╗
║      CISA Security Analysis — Claude Code Multi-Agent        ║
╚══════════════════════════════════════════════════════════════╝

分析したいリポジトリの URL を入力してください:
> https://github.com/example/target-repo

──────────────────────────────────────────────────────────────
  Phase 1 — Sub-agent 1: Reporter
──────────────────────────────────────────────────────────────
  [Reporter] git clone ...
  [Reporter] ファイル分析中...
  [Reporter] レポート出力中...

  🔴 CRITICAL : 3 件
  🟠 HIGH     : 5 件
  🟡 MEDIUM   : 2 件
  🟢 LOW      : 8 件

！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
  ⚠️  CRITICAL 脆弱性が 3 件検出されました
！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！

  3 件の CRITICAL 脆弱性を Sub-agent 2 (Fixer) で自動修復しますか？ [y/n]: y

──────────────────────────────────────────────────────────────
  Phase 3 — Sub-agent 2: Fixer  (3 件を修復)
──────────────────────────────────────────────────────────────
  [Fixer] MDレポートを読み込み中...
  [Fixer] 修復を適用中...
  [Fixer] 修復レポート出力中...

  修復完了    : 3 件
  修復レポート: output/fix_report_20260621_120000.md
```

## 出力ファイル

実行後、`output/` フォルダに以下のファイルが生成されます:

| ファイル | 内容 |
|---------|------|
| `security_report_TIMESTAMP.html` | CISAセキュリティ分析レポート（ブラウザで表示） |
| `scan_metadata_TIMESTAMP.json` | 件数・パス情報（Orchestratorが参照） |
| `fix_report_TIMESTAMP.html` | 修復内容の詳細レポート（修復した場合のみ） |
| `fix_metadata_TIMESTAMP.json` | 修復件数情報（Orchestratorが参照） |

クローンされたリポジトリは `repos/repo_TIMESTAMP/` に保存されます。

## CISAセキュリティチェック項目

### 🔴 CRITICAL（自動修復対象）
| 脆弱性 | CWE | 例 |
|--------|-----|----|
| ハードコード認証情報 | CWE-798 | `API_KEY = "sk-..."` |
| SQLインジェクション | CWE-89 | `f"SELECT * WHERE id={id}"` |
| コマンドインジェクション | CWE-78 | `os.system(f"ping {host}")` |
| リモートコード実行 | CWE-94 | `eval(user_input)` |
| 認証バイパス | CWE-287 | 脆弱な認証チェック |
| 安全でないデシリアライゼーション | CWE-502 | `pickle.loads(data)` |

### 🟠 HIGH
XSS（CWE-79）、パストラバーサル（CWE-22）、SSRF（CWE-918）

### 🟡 MEDIUM
弱い暗号化（MD5/SHA1）、セキュリティ設定ミス、不十分な入力バリデーション

### 🟢 LOW
セキュリティヘッダー欠如、デバッグ情報漏洩、過剰なエラー情報開示

## スキルファイルのカスタマイズ

`agents/` 配下のMarkdownファイルを編集することで動作を変更できます:

- **`reporter_skill.md`** — チェック項目・レポートフォーマットの変更
- **`fixer_skill.md`** — 修復ロジック・修復パターンの追加

## ライセンス

MIT
