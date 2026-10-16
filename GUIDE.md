# Working with local models (Qwen 3.6 35B-A3B in pi)

Notes from the first real pi session (2026-10-15). They cover what went wrong, how to set things up, and how to prompt a model this size.

## What happened in the first session

I asked two things in one session: "tell me about voicememo", then "work out what tools, skills and MCP servers you'd need; write it up in ~/src/localfirst". Qwen answered the first well. On the second, it read pi's `skills.md` and `mcp.md` docs (about 20k characters), then looped. It re-ran `find`, `ls` and `cat README.md` on voicememo about 30 times and never created `localfirst`.

**Root cause: the context window, not the prompt.**

1. The LM Studio server was started without loading a model. The first request came from another app (the andrewlb.com capture-metadata job), so LM Studio loaded Qwen on demand at its **default 8,192-token context**. `llm-on` would have used 32,768.
2. Once the conversation passed 8k, LM Studio's default overflow policy, **TruncateMiddle**, kicked in. It keeps the system prompt and the *first* user message and deletes everything in between. The server log showed: `removing '58465' tokens from the middle` of a 61,767-token prompt.
3. From then on the model only ever saw "Take a look at voicememo", so it kept starting that task over.
4. pi was told every request was about 3,300 tokens, so its automatic summarising (compaction) never triggered.

**Spotting it:** the model redoes earlier work or answers an earlier question. That means the context is broken, and more prompting won't fix it. Start a new session.

**Checking it:** `lms ps` shows the loaded context in the CONTEXT column. It should say 32768. The logs are in `~/.lmstudio/server-logs/YYYY-MM/`, and searching for `TruncateMiddle` finds the overflows.

## Setup checklist

- [ ] Start the model with `llm-on` before running pi, not just `lms server start`.
- [ ] LM Studio → My Models → gear next to qwen3.6-35b-a3b:
  - **Context Length = 32768**, so loads triggered by other apps get 32k too.
  - **Context Overflow = Stop at limit**, so overflow shows up as an error instead of the middle being cut silently.
- [ ] `~/.pi/agent/models.json` says `contextWindow: 32768`. Keep it equal to the loaded context, or pi's compaction fires at the wrong time.
- [ ] Global rules for the model are in `pi/AGENTS.md` in this folder, linked from `~/.pi/agent/AGENTS.md`.
- [ ] Run `llm-off` before Ableton sessions to free about 20 GB of RAM.

## How to prompt a model this size

The model has about 32k tokens of working memory, and pi's own system prompt plus tool definitions take roughly 1.5k of that. One full read of a big file can use a quarter of what's left. Claude can recover from vague, sprawling requests. Qwen at this size mostly can't.

1. **Ask for one concrete deliverable per prompt.** Say what file to produce, what sections it should have, and how long it should be.
   - ✗ "What tools, skills, MCP servers would you need? Collect your thoughts."
   - ✓ "Create `~/src/localfirst/NEEDS.md` with three sections: Tools, MCP servers, Skills. Base it only on voicememo's README.md and package.json. Max one page."
2. **Limit the exploration.** Name the files to read, or set a cap ("read at most 5 files; use `head -100`"). Unbounded "look around" prompts are what fill the context.
3. **Have it write to disk early.** "Write the outline first, then fill in each section." Progress survives compaction, crashes and restarts.
4. **Use a fresh session per task** (`/new`). Leftover file dumps from the previous task are pure overhead.
5. **Give each project a short AGENTS.md.** That way it doesn't rediscover the project each session. Keep it under about 40 lines (template below). Claude is good at drafting these, and Qwen reads them.
6. **Break up big jobs yourself.** Plan with Claude, then hand Qwen one step at a time, each with its own files and acceptance check.
7. **Ask for checks, not prose.** "Run `npm test` and show the result" beats "explain what you changed."
8. **Stop loops early.** If it repeats a tool call or restates your earlier question, don't argue with it. Start a new session and restate the task, smaller.

### Good fits for the local model
- Well-specified edits in one or two files
- Boilerplate, renames, small refactors with tests to verify them
- Summarising a single file or diff; writing commit messages
- Admin chores: shell scripts, git housekeeping, file reorganisation with a clear rule
- Anything private or offline

### Better left to Claude
- Architecture, open-ended research, unfamiliar codebases
- Changes that span many files
- Reading long docs and putting them together

## Project AGENTS.md template (keep it under about 40 lines)

```markdown
# <project>
One sentence on what this is.

## Layout
- src/core/   – <what>
- src/ui/     – <what>
- tools/      – build scripts

## Commands
- Build: `npm run build`
- Test:  `npm test`
- Lint:  `npm run lint`

## Rules
- <the one or two things that are easy to get wrong>
- Don't edit generated files in <dir>.
```
