CISAまたはOWASP Top 10（あるいは両方）に準拠したセキュリティ分析を実行します。
Claude Code がオーケストレーターとなり、Agent ツールでサブエージェントを順番に起動します。
各サブエージェントは自分のスキルファイルを Read ツールで読み込み、手順を実行します。

使い方: /security <git-clone-URL>

---

以下の手順をそのまま実行してください。

## STEP 0: 準備

$ARGS からリポジトリ URL を取得する。空なら「リポジトリ URL を入力してください」とユーザーに聞く。

Bash ツールで以下を実行し、このプロジェクトのルート絶対パスとタイムスタンプを取得してディレクトリを作成する:
```bash
pwd
date +%Y%m%d_%H%M%S
mkdir -p "$(pwd)/repos" "$(pwd)/output"
```

`pwd` の結果を PROJECT_ROOT（絶対パス。cloneされた場所に依存せず動的に取得すること）、取得したタイムスタンプを TS とし、以下のパスを確定する:
- AGENTS_DIR        = PROJECT_ROOT/agents
- REPO_PATH        = PROJECT_ROOT/repos/repo_TS
- CISA_REPORT      = PROJECT_ROOT/output/security_report_TS.html
- CISA_META        = PROJECT_ROOT/output/scan_metadata_TS.json
- OWASP_REPORT     = PROJECT_ROOT/output/owasp_report_TS.html
- OWASP_META       = PROJECT_ROOT/output/owasp_metadata_TS.json
- FIX_REPORT       = PROJECT_ROOT/output/fix_report_TS.html
- FIX_META         = PROJECT_ROOT/output/fix_metadata_TS.json

> **注意:** パスを `/home/masaya/security/...` のように決め打ちしないこと。このリポジトリが
> どのディレクトリにcloneされても動作するよう、必ず STEP 0 で取得した PROJECT_ROOT を基準にすること。

---

## STEP 1: フレームワーク選択

ユーザーに以下のメッセージを表示し、実行するセキュリティフレームワークを選んでもらう:

```
🔍 セキュリティスキャンのフレームワークを選択してください:

  [1] CISA        — CISA ガイドライン準拠（認証情報漏洩・インジェクション・RCE等）
  [2] OWASP       — OWASP LLM Top 10 2025 準拠（プロンプトインジェクション・過剰エージェント権限・機密情報漏洩等）
  [3] CISA + OWASP — 両方を順番に実行（より網羅的）

番号を入力してください [1/2/3]:
```

ユーザーの入力を FRAMEWORK_CHOICE として記録する。
1 → CISA のみ
2 → OWASP のみ
3 → CISA → OWASP の順に実行

---

## STEP 2: CISAスキャン（FRAMEWORK_CHOICE が 1 または 3 の場合のみ実行）

FRAMEWORK_CHOICE が 2 の場合はこのステップをスキップして STEP 3 へ進む。

Agent ツールで CISA Reporter サブエージェントを起動する。
description = "CISA security reporter"

サブエージェントへ渡すプロンプト（REPO_URL・REPO_PATH・CISA_REPORT・CISA_META を実際の値に置き換えて渡す）:

```
まず Read ツールで AGENTS_DIR/reporter_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

リポジトリURL      : REPO_URL
クローン先         : REPO_PATH
HTMLレポート出力先 : CISA_REPORT
メタデータ出力先   : CISA_META

手順の最後まで完了し、メタデータ JSON を必ず出力してください。
```

サブエージェントが完了したら Read ツールで CISA_META を読み込み、以下の形式で表示する:

```
## [CISA] スキャン完了
- 🔴 CRITICAL : N 件
- 🟠 HIGH     : N 件
- 🟡 MEDIUM   : N 件
- 🟢 LOW      : N 件
レポート: CISA_REPORT
```

---

## STEP 3: OWASPスキャン（FRAMEWORK_CHOICE が 2 または 3 の場合のみ実行）

FRAMEWORK_CHOICE が 1 の場合はこのステップをスキップして STEP 4 へ進む。

Agent ツールで OWASP Reporter サブエージェントを起動する。
description = "OWASP LLM Top 10 security reporter"

サブエージェントへ渡すプロンプト（REPO_URL・REPO_PATH・OWASP_REPORT・OWASP_META を実際の値に置き換えて渡す）:

```
まず Read ツールで AGENTS_DIR/owasp_reporter_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

リポジトリURL      : REPO_URL
クローン先         : REPO_PATH
HTMLレポート出力先 : OWASP_REPORT
メタデータ出力先   : OWASP_META

なお、REPO_PATH が既にクローン済みの場合はクローンをスキップしてスキャンのみ実行してください。
手順の最後まで完了し、メタデータ JSON を必ず出力してください。
```

サブエージェントが完了したら Read ツールで OWASP_META を読み込み、以下の形式で表示する:

```
## [OWASP LLM Top 10] スキャン完了
- 🔴 CRITICAL : N 件
- 🟠 HIGH     : N 件
- 🟡 MEDIUM   : N 件
- 🟢 LOW      : N 件
レポート: OWASP_REPORT

LLMカテゴリ別:
  LLM01 プロンプトインジェクション  : N 件
  LLM02 機密情報漏洩               : N 件
  LLM03 サプライチェーン            : N 件
  LLM04 データ・モデルポイズニング  : N 件
  LLM05 不適切な出力処理            : N 件
  LLM06 過剰なエージェント権限      : N 件
  LLM07 システムプロンプト漏洩      : N 件
  LLM08 ベクトル・埋め込みの脆弱性  : N 件
  LLM09 誤情報                      : N 件
  LLM10 無制限のリソース消費        : N 件
```

---

## STEP 4: 修復の確認

実行したスキャンのメタデータから **CRITICAL件数の合計** を計算する:
- FRAMEWORK_CHOICE = 1: cisa_critical のみ
- FRAMEWORK_CHOICE = 2: owasp_critical のみ
- FRAMEWORK_CHOICE = 3: cisa_critical + owasp_critical の合計

CRITICAL合計が 0 なら「CRITICAL 脆弱性は検出されませんでした」と伝えて終了。

CRITICAL合計が 1 以上なら、ユーザーに確認する:
「⚠️ CRITICAL 脆弱性が合計 N 件検出されました。Fixer サブエージェントで自動修復しますか？ [y/n]」

ユーザーが n なら終了。

---

## STEP 5: Fixer サブエージェントを起動

Agent ツールで Fixer サブエージェントを起動する。
description = "security fixer"

サブエージェントへ渡すプロンプトは FRAMEWORK_CHOICE によって変える:

**FRAMEWORK_CHOICE = 1（CISAのみ）の場合:**
```
まず Read ツールで AGENTS_DIR/fixer_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

HTMLレポートパス      : CISA_REPORT
リポジトリパス        : REPO_PATH
修復HTMLレポート出力先: FIX_REPORT
修復メタデータ出力先  : FIX_META

手順の最後まで完了し、修復メタデータ JSON を必ず出力してください。
```

**FRAMEWORK_CHOICE = 2（OWASPのみ）の場合:**
```
まず Read ツールで AGENTS_DIR/fixer_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

HTMLレポートパス      : OWASP_REPORT
リポジトリパス        : REPO_PATH
修復HTMLレポート出力先: FIX_REPORT
修復メタデータ出力先  : FIX_META

手順の最後まで完了し、修復メタデータ JSON を必ず出力してください。
```

**FRAMEWORK_CHOICE = 3（両方）の場合:**
```
まず Read ツールで AGENTS_DIR/fixer_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

CISAレポートパス      : CISA_REPORT
OWASPレポートパス     : OWASP_REPORT
リポジトリパス        : REPO_PATH
修復HTMLレポート出力先: FIX_REPORT
修復メタデータ出力先  : FIX_META

両方のレポートに記載された CRITICAL 脆弱性をすべて修復してください。
重複している脆弱性（同一ファイル・同一行）は一度だけ修復すること。
手順の最後まで完了し、修復メタデータ JSON を必ず出力してください。
```

---

## STEP 6: 修復結果を表示

サブエージェントが完了したら Read ツールで FIX_META を読み込み、ユーザーに表示する:

```
## 修復完了
- ✅ 修復完了   : N 件
- ⚠️ 要手動対応 : N 件
修復レポート: FIX_REPORT
```
