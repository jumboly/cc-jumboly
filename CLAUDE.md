# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリについて

`cc-jumboly` は masa 個人用の Claude Code ハーネス（スラッシュコマンド・スキル・
`~/.claude/CLAUDE.md` スニペット）の**単一情報源リポジトリ**。アプリケーション
コードではなく、将来の Claude Code セッションが読んで実行する**手順書 (markdown)**
を集めている。ビルド・テスト・lint は存在しない。

代わりに 2 つの自然言語トリガで動作する：

- 「インストールして」「アップデートして」 → `INSTALL.md` を読んで実行。
  repo → `~/.claude/` を `diff -rq` 駆動で展開（冪等、SAME はスキップ）。
- 「ローカルから取り込んで」「マージして」 → `MERGE.md` を読んで実行。
  `~/.claude/` → repo 吸い上げ、NEW / CHANGED / SAME で提示を分岐。

**ユーザが明示しない限り、これらのフローを勝手に走らせない。** リポジトリの
ファイルを直接編集する作業（コマンド/スキル新規追加、INSTALL.md 改修など）は通常タスク。

## 横断的な約束ごと

- ディレクトリは `~/.claude/` と 1:1 ミラー（`commands/` と `skills/` のみ）。
- 個人ハーネスは **`j-` プレフィックス必須**。INSTALL/MERGE のスキャン対象を
  絞るキーになっている（`commands/j-*.md`, `skills/j-*/**`）。
- `~/.claude/CLAUDE.md` への追記は **マーカーブロック方式**：
  `<!-- BEGIN cc-jumboly -->` 〜 `<!-- END cc-jumboly -->` 間のみ
  `CLAUDE.snippet.md` で置換、マーカー外はユーザ自筆部分として不可侵。
- スコープ外（`~/.claude/hooks/*.sh`, `settings.json`,
  `statusline-command.sh` 等）は MERGE フローで**ユーザが発話で直接指定した場合のみ**
  取り込む。「全部」のような包括指示では拡張しない。
- `commands/j-*.md` や `skills/j-*/` を編集してもローカル `~/.claude/` には
  自動反映されない。反映には INSTALL フローを明示的に走らせる必要がある。
- コミットはユーザの明示指示が無ければ行わない（INSTALL.md / MERGE.md にも
  同じ方針が書かれている）。

## ファイル構成

- `README.md` — 人間向け概要・3 シナリオの使い方
- `INSTALL.md` — Claude 向け：repo → `~/.claude/` 展開手順（権威ある仕様）
- `MERGE.md` — Claude 向け：`~/.claude/` → repo 吸い上げ手順（権威ある仕様）
- `CLAUDE.snippet.md` — マーカーブロックに挿入される純内容
- `commands/j-*.md`, `skills/j-*/**` — 配信対象本体（現状: `j-handoff`,
  `j-study`）。それぞれの使い分けは `CLAUDE.snippet.md` および各 `SPEC.md` 参照。

CLAUDE.md マージや展開挙動の詳細は `INSTALL.md` / `MERGE.md` が単一情報源。
ここでは概観のみ。
