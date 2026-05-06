# cc-jumboly

masa 個人用の Claude Code ハーネス（スキル / コマンド / 共有 CLAUDE.md スニペット）の
**単一情報源リポジトリ**。`~/.claude/` 配下に展開して使う。

## 構成

```
.
├── README.md                  # このファイル（人間向け）
├── INSTALL.md                 # Claude 向け：repo → ~/.claude/ 展開手順
├── MERGE.md                   # Claude 向け：~/.claude/ → repo 吸い上げ手順
├── CLAUDE.snippet.md          # ユーザレベル CLAUDE.md に挿入するブロック
├── commands/                  # ~/.claude/commands/ にミラー
│   └── j-*.md
└── skills/                    # ~/.claude/skills/ にミラー
    └── j-*/
```

ディレクトリ構造は `~/.claude/` と 1:1 で対応する。

## 命名規則

- 個人作成のスラッシュコマンド／スキルは `j-` プレフィックス必須
  （組み込みやマーケットプレイス由来と区別、補完しやすさのため）。

## 内蔵コマンド

- **`/j-handoff`** — claude.ai (Web) で深掘り学習するためのプロンプトを生成。
  パブリック知識／外に出して良い題材向け。
- **`/j-study`** — Claude Code セッション内で完結する教科書を生成。
  プロプライエタリコード／社内事情／外に出せない題材向け。

詳細仕様はそれぞれ `commands/j-*.md` と `skills/j-*/SPEC.md` を参照。

## 使い方

すべて Claude Code 経由。シェルスクリプトや別ツールは使わない。

### 1. 新規マシンに展開（INSTALL）

```sh
git clone <this-repo> ~/src/cc-jumboly
cd ~/src/cc-jumboly
claude
```

起動した Claude に「インストールして」と依頼。Claude が `INSTALL.md` を読み、
`~/.claude/commands/`・`~/.claude/skills/` に展開し、`~/.claude/CLAUDE.md` に
マーカーブロックを差し込む。冪等なので何度走らせても安全。

### 2. ローカルの改良を取り込む（MERGE）

普段使いで `~/.claude/commands/j-foo.md` を直接編集／追加した場合：

```sh
cd ~/src/cc-jumboly
claude
```

「ローカルから取り込んで」と依頼。Claude が `MERGE.md` を読み、`~/.claude/` を
スキャン、リポジトリと diff、各候補について取り込み判断をユーザに尋ねながら反映。

### 3. 既存の更新を反映（UPDATE）

INSTALL と同じ。「アップデートして」「インストールして」どちらでも `INSTALL.md`
フローが走る。冪等。

## CLAUDE.md マージ方針

- マーカーブロック方式：`<!-- BEGIN cc-jumboly --> ... <!-- END cc-jumboly -->`
- 再展開時はブロック内のみ置換、マーカー外のユーザ自筆部分は不可侵。
- 詳細は `INSTALL.md` / `MERGE.md` の該当セクション参照。

## 現時点のスコープ外

- `~/.claude/hooks/*.sh`
- `~/.claude/settings.json`
- `~/.claude/statusline-command.sh`

これらが必要になったら MERGE フローで明示的に取り込む。
