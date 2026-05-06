# /j-handoff specification

Read by the Agent spawned by `/j-handoff`. The slash-command body is kept
tiny so the caller's context isn't bloated by this file.

## Goal

Produce a single self-contained claude.ai handoff prompt that lets the user
take a topic to claude.ai (Web) for top-down, textbook-grade learning. The
caller (Claude Code) drives implementation; claude.ai provides the deep
explanation that doesn't fit in-line in a fast-moving build session.

## Trigger phrases (offered by the caller, not the agent)

The caller (Claude Code) should offer to invoke `/j-handoff` when the user
says any of these (Japanese-leaning):

- 「claude.ai に持っていきたい / 続きはあっちで」
- 「プロンプトにして / プロンプト用意して」
- 「整理して理解したい / メンタルモデル構築から」
- 「腹落ちさせたい / 教科書化したい」
- 「根っこから整理して」「概念が多すぎる」

The agent itself never evaluates these — by the time the agent runs, the
trigger has already fired.

## Inputs

The bootstrap caller passes these in the agent prompt:

- `ARGUMENTS` — user-supplied focus topic (may be empty).
- `TRANSCRIPT_PATH` — path to the caller's session transcript JSONL (may
  be empty if the project dir convention failed).
- `CWD` — caller's current working directory.

The agent reads, in addition:

- `~/.claude/CLAUDE.md` — global preferences (language, etc.). **Treat as
  the source of truth**; do not re-state defaults inside the generated
  prompt.
- The project memory dir at `~/.claude/projects/<encoded-cwd>/memory/`,
  where `<encoded-cwd>` is `CWD` with `/` replaced by `-`. Read MEMORY.md
  and any individual `*.md` files relevant to the topic.
- Cross-project memories at `~/.claude/projects/*/memory/*.md`. Grep for
  topic keywords (e.g., topic = pkg-config → `shpx*`,
  `setup-vcpkg-nuget-cache`, `libspatialite-sys`). Pull only relevant.
- The session transcript at `TRANSCRIPT_PATH` to extract concrete
  stumbling blocks the user encountered THIS session — actual file
  contents, exact error messages, unexpected command outputs.
  **Role attribution is critical** — see "Identifying genuine user
  stumbles" below.

## Fallbacks

- `TRANSCRIPT_PATH` empty → treat `ARGUMENTS` as the user's own summary of
  the session. If `ARGUMENTS` is also empty, return a code block whose
  content is a one-line apology in Japanese asking the user to re-invoke
  with a topic argument or run from the project directory.
- Project memory dir missing → proceed with global CLAUDE.md and
  ARGUMENTS only.

## Identifying genuine user stumbles (CRITICAL)

Governs how to read `TRANSCRIPT_PATH`. Treat `role: "user"` and
`role: "assistant"` lines as adversarial witnesses; mis-attribution
silently inverts the conversation.

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

**Citation test.** Every stumble in the generated prompt must be
backed by a specific user-role message. If the only evidence is in
assistant-role messages, drop it.

## Output contract

A single fenced markdown code block. Nothing else; no preamble, no
follow-up. The bootstrap caller wraps it with a one-line preamble.

## Required structure of the generated prompt

Order matters:

1. **User's premise** — role, language, current project. Brief but enough
   for claude.ai to assume zero shared context.
2. **Concrete stumbling blocks from THIS session** — actual file contents
   the user saw, exact error messages, unexpected outputs. Numbered list.
   These are the "concrete grounding" that distinguishes this prompt from
   a generic search query.
3. **Mental-model goal** — tree-structured: trunk → main branches → finer
   branches → leaves → flowers. Specify which concepts must appear in
   the tree, each with its position.
4. **Style/format requirements**:
   - Top-down: trunk first, root concept up front
   - Each chapter opens with "this chapter builds X mental model" (1–2
     lines)
   - Each chapter closes with 1–2 understanding-check questions; answers
     in the next chapter's opening
   - Define every term on first use
   - Concrete examples (file snippets, command outputs) preferred over
     abstract description
   - Position the user's stumbles within the tree, not as appendix
   - Single response, no "to be continued"
   - Closing: one-screen ASCII tree or table-of-contents summary
5. **Length** — 6–8 chapters, 20–30 minute read.

## Hard rules (the agent must enforce these on the generated prompt)

- No fixed template — each handoff reflects THIS session's specific
  incidents.
- No generalising stumbles into "common pitfalls" — embed the specific
  ones (the actual `.pc` file content, the actual segfault).
- Single fenced code block — never split.
- Don't use `/j-handoff` to explain inline within the current Claude
  Code session; the point is to hand off.

## Authorial voice (how the prompt itself reads)

- First-person, as the user speaking to claude.ai.
- The agent's narration must not appear inside the fenced block.
- Concrete examples are mandatory; never omit them to save space — they
  are the entire value-add.

## Language

Per `~/.claude/CLAUDE.md` global preference; project-level MEMORY.md may
override. Do not re-state the default inside the generated prompt.
