# Security Analysis — Claude Code Multi-Agent System

CISAガイドラインおよびOWASP Top 10 for LLM Applications (2025) に基づいたセキュリティ分析を自動化するClaude Codeマルチエージェントシステムです。
GitリポジトリのURLを入力するだけで、セキュリティスキャンから脆弱性の自動修復まで実行します。

## 特徴

- **Claude Code完結** — 外部SDKやAPIキー不要。Claude Code の `Agent` ツールのみで動作
- **2つのフレームワークに対応** — CISAガイドライン（CRITICAL/HIGH/MEDIUM/LOW）と OWASP Top 10 for LLM Applications 2025（LLM01〜LLM10）を選択、または両方同時に実行
- **grep-firstスキャン** — 全ファイルを読む前にgrepでヒット箇所を絞り込み、トークン使用量を削減
- **マルチエージェント構成** — スキャン（CISA / OWASP）・修復を専用エージェントが分担
- **ユーザー確認フロー** — 自動修復の前に必ず確認プロンプトを表示
- **重複排除** — CISA + OWASP 両方実行時、同一ファイル・同一行の脆弱性は一度だけ修復
- **HTMLレポート出力** — ブラウザで確認できるレポートを生成

## エージェント構成

```
Orchestrator（.claude/commands/security.md — /security スラッシュコマンド）
│
├─► STEP 1: フレームワーク選択  [1] CISA / [2] OWASP / [3] 両方
│
├─► Sub-agent 1a: CISA Reporter     ← agents/reporter_skill.md を Read して実行（選択1 or 3）
│       git clone → CISAスキャン → HTMLレポート出力
│       完了通知（JSON）: report_path / 件数
│
├─► Sub-agent 1b: OWASP Reporter    ← agents/owasp_reporter_skill.md を Read して実行（選択2 or 3）
│       （クローン済みならスキップ）→ grep-firstスキャン → HTMLレポート出力
│       完了通知（JSON）: report_path / 件数 / LLMカテゴリ別件数
│
│   ⚠️ CRITICAL 合計 N件検出 → "修復しますか？ [y/n]"（ユーザー確認）
│
└─► Sub-agent 2: Fixer              ← agents/fixer_skill.md を Read して実行
        HTMLレポート（1つ or 2つ）を読む → CRITICAL脆弱性を修復（重複は一度だけ）
        → 修復レポート出力
        完了通知（JSON）: fix_report_path / 修復件数
```

## 前提条件

| 必要なもの | 確認方法 |
|-----------|---------|
| Claude Code | `claude --version` |
| Git | `git --version` |
| インターネット接続 | — |

## ファイル構成

```
security/
├── .claude/
│   └── commands/
│       └── security.md            # Orchestrator（スラッシュコマンド実装）
├── CLAUDE.md                      # Claude Code向け技術仕様
├── README.md                      # このファイル
├── agents/
│   ├── reporter_skill.md          # Sub-agent 1a の指示書（CISA）
│   ├── owasp_reporter_skill.md    # Sub-agent 1b の指示書（OWASP LLM Top 10）
│   └── fixer_skill.md             # Sub-agent 2 の指示書
└── output/                        # レポート出力先（実行時に生成）
```

## 使い方

このリポジトリをcloneし、**そのディレクトリ内で** `claude` を起動してください（`/security` は起動中のプロジェクト内の `.claude/commands/security.md` から自動的に認識されます。パスはすべて実行時に `pwd` から動的に解決するため、clone先のディレクトリがどこであっても動作します）。

```bash
git clone <このリポジトリのURL> my-security-check
cd my-security-check
claude
```

Claude Code のチャット上でスラッシュコマンドを実行します:

```
/security https://github.com/example/target-repo
```

URLを省略した場合はOrchestratorがURLの入力を求めます。実行するとまずフレームワーク選択を聞かれ、その後対話形式で進みます:

```
🔍 セキュリティスキャンのフレームワークを選択してください:

  [1] CISA        — CISA ガイドライン準拠（認証情報漏洩・インジェクション・RCE等）
  [2] OWASP       — OWASP LLM Top 10 2025 準拠（プロンプトインジェクション・過剰エージェント権限・機密情報漏洩等）
  [3] CISA + OWASP — 両方を順番に実行（より網羅的）

番号を入力してください [1/2/3]: 3

## [CISA] スキャン完了
- 🔴 CRITICAL : 3 件
- 🟠 HIGH     : 5 件
- 🟡 MEDIUM   : 2 件
- 🟢 LOW      : 8 件
レポート: output/security_report_20260704_120000.html

## [OWASP LLM Top 10] スキャン完了
- 🔴 CRITICAL : 1 件
- 🟠 HIGH     : 2 件
- 🟡 MEDIUM   : 1 件
- 🟢 LOW      : 0 件
レポート: output/owasp_report_20260704_120000.html

⚠️ CRITICAL 脆弱性が合計 4 件検出されました。Fixer サブエージェントで自動修復しますか？ [y/n]: y

## 修復完了
- ✅ 修復完了   : 4 件
- ⚠️ 要手動対応 : 0 件
修復レポート: output/fix_report_20260704_120000.html
```

## 出力ファイル

実行後、`output/` フォルダに以下のファイルが生成されます:

| ファイル | 生成者 | 内容 |
|---------|--------|------|
| `security_report_TIMESTAMP.html` | CISA Reporter | CISAセキュリティ分析レポート（ブラウザで表示） |
| `scan_metadata_TIMESTAMP.json` | CISA Reporter | 件数・パス情報（Orchestratorが参照） |
| `owasp_report_TIMESTAMP.html` | OWASP Reporter | OWASP LLM Top 10 分析レポート（ブラウザで表示） |
| `owasp_metadata_TIMESTAMP.json` | OWASP Reporter | 件数・LLMカテゴリ別件数（Orchestratorが参照） |
| `fix_report_TIMESTAMP.html` | Fixer | 修復内容の詳細レポート（修復した場合のみ） |
| `fix_metadata_TIMESTAMP.json` | Fixer | 修復件数情報（Orchestratorが参照） |

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

## OWASP Top 10 for LLM Applications (2025) チェック項目

| LLMカテゴリ | 内容 |
|------------|------|
| LLM01 | プロンプトインジェクション（直接・間接） |
| LLM02 | 機密情報漏洩（APIキーのハードコード、PII出力） |
| LLM03 | サプライチェーン（脆弱なLLM SDKバージョン） |
| LLM04 | データ・モデルポイズニング |
| LLM05 | 不適切な出力処理（LLM出力のeval/exec実行、XSS） |
| LLM06 | 過剰なエージェント権限（不可逆操作の自律実行） |
| LLM07 | システムプロンプト漏洩 |
| LLM08 | ベクトル・埋め込みの脆弱性（RAGポイズニング、アクセス制御なし） |
| LLM09 | 誤情報（ファクトチェックなしでの重要判断への利用） |
| LLM10 | 無制限のリソース消費（トークン上限・レート制限なし） |

CRITICAL（CVSS 9.0以上）〜LOW（CVSS 0.1〜3.9）の4段階で分類し、CISAと同様に自動修復対象はCRITICALのみです。

## スキルファイルのカスタマイズ

`agents/` 配下のMarkdownファイルを編集することで動作を変更できます:

- **`reporter_skill.md`** — CISAのチェック項目・レポートフォーマットの変更
- **`owasp_reporter_skill.md`** — OWASP LLM Top 10 のチェック項目・grepパターン・レポートフォーマットの変更
- **`fixer_skill.md`** — 修復ロジック・修復パターンの追加

## ライセンス

MIT
