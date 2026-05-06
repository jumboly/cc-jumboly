# INSTALL.md — repo → `~/.claude/` 展開手順

このファイルは「（cc-jumboly を）インストールして」「アップデートして」と
依頼された Claude が読む手順書。リポジトリ root を CWD として実行する前提。

## ゴール

このリポジトリの中身を `~/.claude/` 配下にミラーし、ユーザレベル CLAUDE.md に
マーカーブロックを設置／更新する。冪等：同じ手順を 2 回走らせて差分ゼロ。

## 手順

### 1. 前提確認

- `pwd` がこのリポジトリ root であることを確認（`README.md`, `INSTALL.md`,
  `commands/`, `skills/` が見えるはず）。
- `mkdir -p ~/.claude/commands ~/.claude/skills`（既存でも無害）。

### 2. ファイル展開

対象：

- `commands/j-*.md` → `~/.claude/commands/j-*.md`
- `skills/j-*/**`   → `~/.claude/skills/j-*/**`

手順：

1. `diff -rq commands ~/.claude/commands` と `diff -rq skills ~/.claude/skills`
   をまず実行し、状態を 3 つに分類する（バイト完全一致で判定。末尾改行差も差分扱い）：
   - **NEW** — 「Only in `commands`」「Only in `skills`」のように
     リポジトリ側にだけ存在 → そのままリポジトリ → ローカルに新規 Write。
   - **CHANGED** — 「Files ... differ」 → 後段 (3.) の確認フローへ。
   - **SAME** — 出力なし → 何もしない（冪等性確保）。
2. ローカル側にだけ存在するファイル（「Only in `~/.claude/...`」）は INSTALL では
   触らない（取り込みは `MERGE.md` の責務）。
3. CHANGED 各ファイルについて、両側を Read（並列可）して diff を提示し 3 択：
   1. リポジトリ版で上書き（推奨：単一情報源を維持）
   2. ローカル版を保持（リポジトリには反映しない）
   3. 一度 `MERGE.md` フローで吸い上げてからインストールし直す

### 3. CLAUDE.md マーカーマージ

1. `~/.claude/CLAUDE.md` を Read。なければ作成して以下の手順に進む。
2. `<!-- BEGIN cc-jumboly -->` と `<!-- END cc-jumboly -->` の有無を確認。
3. ブロック内に挿入する内容は **このリポジトリの `CLAUDE.snippet.md` 全文**。
4. 分岐：
   - **両マーカーあり** → ブロック内（マーカー行は除き、その間の本文）を
     `CLAUDE.snippet.md` の内容で置換。既に同一なら書き込まない。
   - **マーカー無し** → CLAUDE.md 末尾に空行 + `<!-- BEGIN cc-jumboly -->` +
     改行 + `CLAUDE.snippet.md` の内容 + 改行 + `<!-- END cc-jumboly -->` を追加。
   - **片方だけある／BEGIN が複数ある／順序が逆** → 不整合。ユーザに状況を
     提示し 3 択：(a) 既存マーカー類を全削除して再生成、(b) 手で直すので中断、
     (c) Claude が構造を整える（差分提示・承認後）。
5. **重複チェック**（マーカーを今回新規挿入した場合のみ実行）：
   `CLAUDE.snippet.md` 内の `##` 見出し本体（例: `## 自作コマンド (cc-jumboly 提供)`）
   から末尾の括弧注記を除いた prefix（例: `## 自作コマンド`）で始まる行が
   マーカー外に存在する場合、ユーザに「マーカー外の旧セクションを
   削除しますか？」と確認する。勝手には消さない。

### 4. 検証

以下を実行し、結果を要約報告：

- `diff -rq commands ~/.claude/commands` — `j-*` に関する差分行が無いこと
  （ローカル独自の `j-*` 以外ファイルは差分扱いで構わない）
- `diff -rq skills ~/.claude/skills` — 同上
- `grep -c "BEGIN cc-jumboly" ~/.claude/CLAUDE.md` — 値が **1** であること。
  2 以上なら §3 の不整合分岐にエスカレーションして復旧。

### 5. 報告

- 追加 / 更新 / スキップしたファイル数
- CLAUDE.md に対して行った操作（追加 / 置換 / 変更なし）
- ユーザに確認した項目とその結果

## 注意

- `~/.claude/hooks/*.sh`、`settings.json`、`statusline-command.sh` 等は
  このリポジトリの初期スコープ外。必要なら `MERGE.md` フローで取り込む。
- マーカー外のユーザ自筆部分は絶対に書き換えない。
- git コミットはユーザの明示指示が無い限り行わない。
