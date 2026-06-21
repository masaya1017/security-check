CISAガイドラインに準拠したセキュリティ分析を実行します。
Claude Code がオーケストレーターとなり、Agent ツールでサブエージェントを順番に起動します。
各サブエージェントは自分のスキルファイルを Read ツールで読み込み、手順を実行します。

使い方: /security <git-clone-URL>

---

以下の手順をそのまま実行してください。

## STEP 0: 準備

$ARGS からリポジトリ URL を取得する。空なら「リポジトリ URL を入力してください」とユーザーに聞く。

Bash ツールで以下を実行し、タイムスタンプを取得してディレクトリを作成する:
```bash
mkdir -p /home/masaya/security/repos /home/masaya/security/output
date +%Y%m%d_%H%M%S
```

取得したタイムスタンプを TS とし、以下のパスを確定する:
- REPO_PATH   = /home/masaya/security/repos/repo_TS
- REPORT_PATH = /home/masaya/security/output/security_report_TS.html
- SCAN_META   = /home/masaya/security/output/scan_metadata_TS.json
- FIX_REPORT  = /home/masaya/security/output/fix_report_TS.html
- FIX_META    = /home/masaya/security/output/fix_metadata_TS.json

---

## STEP 1: Reporter サブエージェントを起動

Agent ツールで Reporter サブエージェントを起動する。
description = "CISA security reporter"

サブエージェントへ渡すプロンプト（REPO_URL・REPO_PATH・REPORT_PATH・SCAN_META を実際の値に置き換えて渡す）:

```
まず Read ツールで /home/masaya/security/agents/reporter_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

リポジトリURL      : REPO_URL
クローン先         : REPO_PATH
HTMLレポート出力先 : REPORT_PATH
メタデータ出力先   : SCAN_META

手順の最後まで完了し、メタデータ JSON を必ず出力してください。
```

---

## STEP 2: スキャン結果を表示

サブエージェントが完了したら Read ツールで SCAN_META を読み込み、ユーザーに以下の形式で表示する:

```
## スキャン完了
- 🔴 CRITICAL : N 件
- 🟠 HIGH     : N 件
- 🟡 MEDIUM   : N 件
- 🟢 LOW      : N 件
レポート: REPORT_PATH
```

---

## STEP 3: 修復の確認

SCAN_META の critical_count が 0 なら「CRITICAL 脆弱性は検出されませんでした」と伝えて終了。

critical_count が 1 以上なら、ユーザーに確認する:
「⚠️ CRITICAL 脆弱性が N 件検出されました。Fixer サブエージェントで自動修復しますか？ [y/n]」

ユーザーが n なら終了。

---

## STEP 4: Fixer サブエージェントを起動

Agent ツールで Fixer サブエージェントを起動する。
description = "CISA security fixer"

サブエージェントへ渡すプロンプト（各パスを実際の値に置き換えて渡す）:

```
まず Read ツールで /home/masaya/security/agents/fixer_skill.md を読み込み、
そこに書かれた手順をすべて実行してください。

## 実行タスク

HTMLレポートパス      : REPORT_PATH
リポジトリパス        : REPO_PATH
修復HTMLレポート出力先: FIX_REPORT
修復メタデータ出力先  : FIX_META

手順の最後まで完了し、修復メタデータ JSON を必ず出力してください。
```

---

## STEP 5: 修復結果を表示

サブエージェントが完了したら Read ツールで FIX_META を読み込み、ユーザーに表示する:

```
## 修復完了
- ✅ 修復完了   : N 件
- ⚠️ 要手動対応 : N 件
修復レポート: FIX_REPORT
```
