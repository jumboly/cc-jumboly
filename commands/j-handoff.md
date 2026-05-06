# /j-handoff — generate a claude.ai handoff prompt

Trigger: the user wants to take a topic to claude.ai (Web) for top-down deep
learning. Detailed trigger phrases live in
`~/.claude/skills/j-handoff/SPEC.md`.

1. Locate this session's transcript:

       ls -t ~/.claude/projects/$(pwd | tr / -)/*.jsonl 2>/dev/null | head -1

   (capture as `TRANSCRIPT_PATH`; may be empty)

2. Spawn an Agent (`subagent_type: general-purpose`) with this prompt:

       Generate a claude.ai handoff prompt per
       ~/.claude/skills/j-handoff/SPEC.md.
       ARGUMENTS: {{ARGUMENTS}}
       TRANSCRIPT_PATH: {{TRANSCRIPT_PATH}}
       CWD: {{CWD}}

       Output: one fenced markdown code block, nothing else.

3. Present the Agent's returned text verbatim, prefixed only by:
   `claude.ai に貼ってください:`
