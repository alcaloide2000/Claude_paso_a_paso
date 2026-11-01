# Claude Code essentials

A companion to the [skills guide](README.md): the habits and features that matter once your project is set up. Examples use the bridge-inspection project from the guide.

## 1. How Claude Code works

Claude Code is an agent, not a chat. It reads your files, runs commands, edits files and checks the result, in a loop, until the task is done. You can run it in three places:

| Where | Best for |
| --- | --- |
| Desktop app, Code tab | Working on a local folder without a terminal |
| Web, claude.ai/code | Cloud sessions on a GitHub repo that keep running when your laptop is closed |
| Terminal, `claude` | Full control, scripting, Docker and local tools such as ODM |

Install instructions: [code.claude.com/docs/en/setup](https://code.claude.com/docs/en/setup).

## 2. Keep the context clean

Everything Claude is working with sits in a limited context window: `CLAUDE.md`, opened files, the conversation, command output and connected tools. When it fills with unrelated material, results get worse.

| Command | Use it to |
| --- | --- |
| `/clear` | Start fresh when you switch tasks |
| `/compact <what to keep>` | Summarise a long session and free space, e.g. `/compact keep the ODM settings we chose` |
| `/context` | See what is taking up space |

Rule of thumb: one task, one conversation. Processing site A and debugging the Open3D script are two sessions.

## 3. Keep CLAUDE.md short

`CLAUDE.md` is read at the start of every session, so every line costs context every time.

- Include only what Claude cannot work out from the files: how to test, what never to touch, naming conventions.
- Write rules it can check: "run the quality script and show the numbers before saying a model is done", not "make good models".
- When Claude makes the same mistake twice, add one line to stop it.
- Use two levels: the project's `CLAUDE.md` for that repo, and `~/.claude/CLAUDE.md` for preferences across all projects (for example "answer in Spanish").

## 4. Work in five steps

1. **Explore.** "Read how the ODM outputs are organised. Don't change anything yet."
2. **Plan.** Switch to plan mode (Shift+Tab in the terminal, or the mode picker in the app). Claude researches and proposes a plan without editing files. Correct it before going on.
3. **Implement.** Approve the plan.
4. **Verify.** Ask Claude to prove it works: run the script, open the output, show the numbers.
5. **Commit.** Review the diff and commit to git.

For long jobs, keep a `PLAN.md` (stages, each with how to check it) and a `PROGRESS.md` (what is done, what is left). A new session reads them and continues.

Two brakes:
- **Esc** stops Claude immediately. Use it as soon as it heads the wrong way.
- **Esc twice**, or `/rewind`, goes back to an earlier point and can undo file changes.

## 5. Pick the right model

Switch with `/model`. Use the most capable model for planning, hard problems and reviews; a mid-size model for day-to-day work; the smallest, fastest model for mechanical jobs like searching, renaming or sorting files. Check the current lineup in `/model`, since it changes.

## 6. Extend Claude Code

| Piece | What it is | Where it lives | Example for this project |
| --- | --- | --- | --- |
| Skill | Saved instructions Claude loads only when needed | `.claude/skills/<name>/SKILL.md` | [`odm-concrete`](example/.claude/skills/odm-concrete/SKILL.md) |
| Subagent | A separate Claude with one job and its own context, returning only a summary | `.claude/agents/`, create with `/agents` | A reviewer that checks a point cloud report |
| Hook | A command that runs automatically on an event, every time | `.claude/settings.json`, edit with `/hooks` | Block any command that touches `data/` |
| MCP server | A connector to an outside tool or service | `claude mcp add`, or app connectors | GitHub, Google Drive with site photos |
| Plugin | A bundle of skills, agents, hooks and connectors | `/plugin` | Share your setup with a colleague |

Key difference: an instruction in `CLAUDE.md` is a request Claude usually follows. A hook always runs. Anything that must never fail goes in a hook.

Skills follow the open [Agent Skills](https://agentskills.io) format, so a skill you write is portable to other agent tools that support it.

Install plugins and MCP servers only from sources you trust: they can act on your computer, and each one uses some context.

## 7. Run work in parallel

- Run several sessions at once, each on its own task. The desktop app shows them side by side.
- Give each session its own git worktree so their edits don't collide (see [worktrees](https://code.claude.com/docs/en/worktrees)).
- `/loop` repeats a prompt on an interval while the session is open, e.g. check the ODM log every 10 minutes.
- `/schedule` creates a routine that runs in the cloud, even with your computer off.
- `claude -p "..."` runs one prompt without a conversation, for scripts.

## 8. Stay safe

1. **Set permissions.** In `.claude/settings.json` or with `/permissions`, allow what is routine (running ODM, tests) and deny what must never happen (deleting `data/`, pushing to `main`). See Step 4 of the guide.
2. **Consider auto mode.** Instead of approving every step yourself, auto mode has a separate check review each action and block risky ones. Approving dozens of prompts by hand tends to become approving without reading.
3. **Treat outside content as data.** Web pages, PDFs and emails can hide instructions aimed at the agent (prompt injection). Check what Claude does after reading external material.
4. **Keep secrets out of reach.** Put keys in a git-ignored `.env`, and add a deny rule or hook so Claude can't read it.
5. **Review before committing.** `/code-review` checks your changes for bugs; `/security-review` focuses on security issues.

## 9. Common mistakes

| Mistake | Fix |
| --- | --- |
| Several tasks in one session | `/clear` between tasks |
| Asking without saying why | Give the purpose: "for a crack-width inspection at 1 mm" changes the right ODM settings |
| Skipping the plan on big jobs | Plan mode first for anything longer than a few minutes |
| Not asking for proof | End with "show me it works" |
| A three-page `CLAUDE.md` | Cut it to one screen; move procedures into skills |
| Letting a wrong run continue | Press Esc and correct it |
| Installing every plugin you see | Install only what you use |
| Repeating the same explanation every week | Turn it into a skill |

## Further reading

- [Claude Code docs](https://code.claude.com/docs/en/overview) (official, always current)
- [Best practices](https://code.claude.com/docs/en/best-practices)
- Ismael (@n0tisma), ["CLAUDE CODE de 0 a experto: La guía completa"](https://x.com/n0tisma/status/2114335960053821538) on X, Oct 25, 2026 (Spanish). This page covers similar topics in its own words.
