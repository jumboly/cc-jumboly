# /j-study specification

Read by the Agent spawned by `/j-study`. The slash-command body is kept
tiny so the caller's context isn't bloated by this file.

## Goal

Produce a self-contained textbook (markdown) that the user reads inside
the current Claude Code session for top-down, textbook-grade learning.
Unlike `/j-handoff`, this does NOT generate a prompt for claude.ai — the
output IS the textbook itself, rendered directly into the conversation.
Suitable for proprietary code, internal materials, or anything that
cannot be shared outside the user's environment.

## Trigger phrases (offered by the caller, not the agent)

The caller (Claude Code) should offer to invoke `/j-study` when the user
says any of these (Japanese-leaning):

- 「ここで（claude.ai に出さずに）腹落ちさせたい」
- 「コード見せたまま教科書化したい」
- 「自社コードで／プロプライエタリ題材で整理したい」
- 「外に出せないけど整理して理解したい」
- 「メンタルモデル構築から、ここで」
- 「腹落ちさせたい」「教科書化したい」（claude.ai への言及なし）

`/j-handoff` 寄りシグナル（claude.ai／持っていく／プロンプト 等）との混在判定は
caller の責務であり、その方針は `~/.claude/CLAUDE.md` のマーカーブロックに
記載されている（cc-jumboly 提供）。SPEC では繰り返さない。

The agent itself never evaluates these — by the time the agent runs, the
trigger has already fired.

## Inputs

The bootstrap caller passes these in the agent prompt:

- `ARGUMENTS` — user-supplied focus topic (may be empty; scope hints
  already stripped by the caller).
- `TRANSCRIPT_PATHS` — newline-separated list of session transcript JSONL
  paths (most recent first). Default = single current session; the caller
  may extend via scope hints. May be empty.
- `CWD` — caller's current working directory.

The agent reads, in addition:

- `~/.claude/CLAUDE.md` — global preferences (language, etc.). **Treat as
  the source of truth**; do not re-state defaults inside the textbook.
- The project memory dir at `~/.claude/projects/<encoded-cwd>/memory/`,
  where `<encoded-cwd>` is `CWD` with `/` replaced by `-`. Read MEMORY.md
  and any individual `*.md` files relevant to the topic.
- Cross-project memories at `~/.claude/projects/*/memory/*.md`. Grep for
  topic keywords. Pull only relevant.
- Session transcripts in `TRANSCRIPT_PATHS` to extract concrete stumbling
  blocks. **Multi-session handling**: when the list has more than one
  entry, treat each transcript as an independent session (do not blend
  turn boundaries). For each, grep for `role:"user"` lines and topic
  keywords; Read only `offset±50`-line windows around matches.
  **Independent sessions parallelize** — issue greps and Reads across
  sessions in a single parallel batch. When citing a stumble, attribute
  it to the session it came from (date from the JSONL filename or
  `stat`). Most-recent session weighted highest if you must pick.
  **Role attribution is critical** — see "Identifying genuine user
  stumbles" below.
- **Proprietary inputs allowed.** Since output stays inside Claude Code,
  the agent may freely Read and embed code from the user's project
  (file contents, identifiers, internal API names, in-repo doc snippets)
  without any abstraction or self-censorship. **Concrete is mandatory.**

## Fallbacks

- `TRANSCRIPT_PATHS` empty → treat `ARGUMENTS` as the user's own summary
  of the session(s). If `ARGUMENTS` is also empty, return a one-line
  apology in Japanese asking the user to re-invoke with a topic argument
  or scope hint, or run from the project directory.
- Project memory dir missing → proceed with global CLAUDE.md and
  ARGUMENTS only.

## Identifying genuine user stumbles (CRITICAL)

Governs how to read each entry of `TRANSCRIPT_PATHS`. Treat `role: "user"`
and `role: "assistant"` lines as adversarial witnesses; mis-attribution
silently inverts the conversation. Apply per session — do not cross
attribution between sessions.

**Highest-priority exclusion — user-taught content.** When a user-role
message corrects, supplements, or out-knows the assistant
(e.g., 「そうじゃなくて X」「実は Y が使える」「最初気づかなかったが Z」),
the user **taught** — the opposite of a stumble. Embed as user
confusion **never**. If a single exchange contains both a user question
and a later user correction, the user taught — exclude entirely.

**Counts as a stumble** (include):

- User-role message expressing confusion (e.g. 「なぜ?」「わからない」
  「どういう意味?」, "why does X happen?").
- User-role message asking for help after a tool error or unexpected
  output.
- A user-proposed approach the assistant later corrected — the user's
  initial assumption is the stumble.
- User-role message accepting a non-obvious resolution
  (e.g. 「なるほど」「確かに」) — marks a just-resolved stumble.

**Other exclusions** (besides user-taught above):

- Assistant prose introducing a concept the user never asked about.
- Assistant's own mistakes (build retries, ranlib warnings) it fixed
  without user intervention.

**Citation test.** Every stumble in the textbook must be backed by a
specific user-role message. If the only evidence is in assistant-role
messages, drop it.

## Output contract

Plain markdown, ready to render directly in the Claude Code chat.

- **Do NOT wrap the textbook in a fenced code block.** The bootstrap
  caller displays the result as-is.
- A short opening line (e.g.「以下、教科書です。」) is allowed but not
  required.
- Narration / authorial asides may be mixed with exposition.
- Output should be readable top-to-bottom in the chat without copy-paste
  to another tool.

## Required structure of the textbook

Order matters:

1. **Topic premise** — what we're building a mental model of, and why
   now. Reference the user's session context (current project, the
   specific stumble that triggered this study).
2. **Concrete stumbling blocks from THIS session** — actual file
   contents the user saw, exact error messages, unexpected outputs.
   Numbered list. **Embed verbatim**; no abstraction needed since
   output stays internal.
3. **Mental-model goal** — tree-structured: trunk → main branches →
   finer branches → leaves → flowers. Specify which concepts must
   appear in the tree, each with its position.
4. **Style/format requirements**:
   - Top-down: trunk first, root concept up front
   - Each chapter opens with "this chapter builds X mental model" (1–2
     lines)
   - Each chapter closes with 1–2 understanding-check questions; answers
     in the next chapter's opening
   - Define every term on first use
   - Concrete examples (file snippets, command outputs, in-repo
     identifiers) preferred over abstract description
   - Position the user's stumbles within the tree, not as appendix
   - Single response, no "to be continued"
   - Closing: one-screen ASCII tree or table-of-contents summary
5. **Length** — 6–8 chapters, 20–30 minute read.

## Hard rules (the agent must enforce these on the textbook)

- No fixed template — each textbook reflects THIS session's specific
  incidents.
- No generalising stumbles into "common pitfalls" — embed the specific
  ones (the actual `.pc` file content, the actual segfault).
- **This DOES run within the current Claude Code session.** The output
  IS the textbook itself, not a prompt for another system. Don't add
  "ask claude.ai to explain X" deferrals.
- No self-censorship of proprietary content. The point of `/j-study`
  (vs `/j-handoff`) is that internal material can stay internal.
- **Never full-Read a transcript JSONL.** Always `grep -n` first (for
  `role:"user"` lines and topic keywords), then Read only
  `offset±50`-line windows around matches. Applies regardless of file
  size — JSONLs can be megabytes.

## Authorial voice (how the textbook reads)

- "Teacher" register: explanatory, third-person about the system,
  second-person (「あなた」) OK when addressing the user. Narration
  mixed with exposition is fine.
- First-person-as-user is **not** required (the agent is teaching, not
  the user speaking to claude.ai).
- Concrete examples are mandatory; never omit them to save space — they
  are the entire value-add. Where a `/j-handoff` prompt would have to
  abstract repo internals, here embed them verbatim.

## Language

Per `~/.claude/CLAUDE.md` global preference; project-level MEMORY.md may
override. Do not re-state the default inside the textbook.
