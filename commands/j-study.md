# /j-study — generate a textbook inside the current Claude Code session

Trigger: the user wants top-down deep learning of a topic that should
**not** leave the current Claude Code environment (proprietary code,
internal materials, anything not for claude.ai). For public-knowledge
topics that should go to claude.ai, use `/j-handoff` instead. Detailed
trigger phrases live in `~/.claude/skills/j-study/SPEC.md`.

1. Parse leading scope hint from `ARGUMENTS`, then resolve transcripts.

   Default = current session (most recent 1 JSONL).

   Match rules:

   - Scan `ARGUMENTS` left-to-right; only the **first** matching hint
     is used as scope, the rest stays as topic.
   - Hints must occupy **whole tokens** (whitespace boundaries) — so
     「今日の」 is a hint by itself, but inside 「今日のメモリ」 it is NOT.
   - 「N 件」 alone is NOT a hint (require 直近/最近/last prefix).
   - Date format is strict `YYYY-MM-DD`; non-conforming dates → topic.

   Hint vocabulary (Japanese / English mixed OK):

   - 「今日の」「本日の」「today」 → JSONLs modified today (local TZ)
   - 「昨日の」「yesterday」 → JSONLs from yesterday only
   - 「直近 N 件」「最近 N 件」「last N」 → most recent N JSONLs
   - 「N 日分」「過去 N 日」「past N days」 → JSONLs modified within last N days
   - 「YYYY-MM-DD 以降」「since YYYY-MM-DD」 → JSONLs with mtime ≥ that date
   - 「YYYY-MM-DD のみ」「on YYYY-MM-DD」 → JSONLs modified on that date only
   - (no hint / unrecognized) → most recent 1 = current session (default)

   Strip the recognized hint tokens from `ARGUMENTS`; the remainder is
   the topic (may be empty).

   Resolve the JSONL list. Compute date references **once** up front
   (don't re-call `date` per file):

       PROJECT_DIR=~/.claude/projects/$(pwd | tr / -)
       TODAY=$(date +%F)
       # macOS:  YESTERDAY=$(date -v-1d +%F)
       # Linux:  YESTERDAY=$(date -d yesterday +%F)

       # default / last N — ls -t is already mtime-desc, no resort needed
       ls -1t "$PROJECT_DIR"/*.jsonl 2>/dev/null | head -n N

       # since DATE / today / on DATE / N 日分 — find is unsorted; resort
       # via `-exec ls -1t {} +` (POSIX: not invoked when there are 0 matches,
       # avoiding the `xargs ls` empty-stdin → CWD listing trap).
       find "$PROJECT_DIR" -maxdepth 1 -name "*.jsonl" \
            -newermt "$DATE 00:00" -exec ls -1t {} + 2>/dev/null

       # 昨日の — exclude today
       find "$PROJECT_DIR" -maxdepth 1 -name "*.jsonl" \
            -newermt "$YESTERDAY 00:00" ! -newermt "$TODAY 00:00" \
            -exec ls -1t {} + 2>/dev/null

   Capture as `TRANSCRIPT_PATHS` (newline-separated, most-recent first).

   **Empty-result handling**: if a scope hint was given AND the resolved
   list is empty, warn the user — "該当セッションなし。topic だけで
   進めますか？" Only proceed to step 2 on confirmation. (No scope hint +
   empty list = brand-new project: proceed as default.)

2. Spawn an Agent (`subagent_type: general-purpose`) with this prompt:

       Generate a textbook per ~/.claude/skills/j-study/SPEC.md.
       ARGUMENTS: {{TOPIC}}
       TRANSCRIPT_PATHS:
       {{PATHS_NEWLINE_SEPARATED}}
       CWD: {{CWD}}

       Output: plain markdown, ready to display in chat. Do NOT wrap in
       a fenced code block.

3. Display the Agent's returned text verbatim in the chat.

4. After display, offer to save:

       「保存しますか？ `./j-study/<YYYY-MM-DD>-<topic-slug>.md` に書き出します」

   If the user agrees:
   - `mkdir -p ./j-study`
   - Slug derivation: source = `ARGUMENTS`, fallback = the textbook's first
     H1 heading, fallback = `study`. Transform: replace whitespace and path
     separators (`/`, `\`) with `-`, collapse runs of `-`, strip leading/
     trailing `-`. Unicode (kanji, kana) is preserved verbatim. If the
     result is empty or `.`/`..`, use `study`.
   - Target = `./j-study/$(date +%Y-%m-%d)-<slug>.md`. If the file already
     exists, show its first line and ask: overwrite / append `-2` (or next
     free integer) / cancel.
   - Issue the Write. On any I/O failure, report it in one line — the
     textbook is still on screen, nothing is lost.
   - If CWD is in a git repo and `.gitignore` does not already match
     `j-study/`, ask yes/no; on yes append `j-study/` to `.gitignore`.
     Skip the prompt if it already matches.
