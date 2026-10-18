# Local Claude project with skills, synced to the app

Step-by-step guide to building a local Claude Code project with skills, and making those skills available in the Claude app. As of Oct 15, 2026.

## Overview

You keep skills in your project's git repository, upload each one to the Claude app once, and the app pushes it to every other place Claude runs. The sync only goes one way: from the app down to Claude Code, Cowork and cloud sessions.

```text
                      loads project skills directly
        ┌─────────────────────────────────────────────────┐
        │                                                 ▼
┌───────────────────┐  upload   ┌──────────────────┐   ┌─────────────────────────┐
│ Your project repo │ ────────▶ │    Claude app    │──▶│ Claude Code             │
│ .claude/skills/   │   (ZIP)   │ Customize>Skills │   │ ~/.claude/skills/synced │
│ source of truth   │           └──────────────────┘   ├─────────────────────────┤
└───────────────────┘            auto-sync, download ─▶│ Cowork sessions         │
                                 only                  ├─────────────────────────┤
                                                    ─▶ │ Cloud sessions          │
                                                       └─────────────────────────┘
```

Claude Code reads your project skills straight from the folder, so it never needs the upload. The upload is what makes the same skill available in app chats, Cowork and on your other machines.

## Step 1: Check the prerequisites

Sync only works when Claude Code is signed in with the same claude.ai account you use in the app.

1. Install or update Claude Code, then check the version. Terminal sync needs v2.1.273 or later.

   ```bash
   claude --version
   claude update
   ```

2. Start `claude` and run `/login`. Choose your claude.ai account, not an API key. A session that uses an API key, `ANTHROPIC_AUTH_TOKEN` or an `apiKeyHelper` does not sync.
3. If you signed in during a running session, restart Claude Code so syncing starts.
4. Have `git` and a zip tool available. You will zip skill folders before uploading them.

## Step 2: Create the project folder

One git repository holds the project, its instructions and its skills. Git is your source of truth: the app copy is just a published version.

```bash
mkdir -p ~/projects/bridge-inspection
cd ~/projects/bridge-inspection
git init
mkdir -p .claude/skills
```

The target layout after the next steps:

```text
bridge-inspection/
├── CLAUDE.md                  # project context, read every session
├── .claude/
│   ├── settings.json          # shared settings (optional)
│   ├── settings.local.json    # your personal settings, not committed
│   └── skills/
│       └── odm-run/
│           ├── SKILL.md       # the skill itself
│           └── reference.md   # optional supporting file
├── data/                      # drone images (git-ignored)
└── outputs/                   # ODM results (git-ignored)
```

Add `data/`, `outputs/` and `.claude/settings.local.json` to `.gitignore` so images and personal settings stay out of the repository.

## Step 3: Write CLAUDE.md

`CLAUDE.md` holds the facts Claude should know in every session; skills hold procedures it loads only when needed.

1. In the project folder, start `claude` and run `/init`. Claude drafts a `CLAUDE.md` from what it finds.
2. Edit it down to short, durable facts. See [`example/CLAUDE.md`](example/CLAUDE.md).
3. Commit it: `git add CLAUDE.md && git commit -m "Add project context"`.

Keep step-by-step procedures out of `CLAUDE.md`. When a section turns into a procedure, move it into a skill.

## Step 4: Add project settings (optional)

Skip this until a prompt annoys you. Its main use is pre-approving commands you run all the time, so Claude stops asking. See [`example/.claude/settings.json`](example/.claude/settings.json).

Commit `settings.json` to share it. Put anything personal in `.claude/settings.local.json`, which stays out of git.

## Step 5: Create a project skill

A skill is a folder with a `SKILL.md` inside. Write it so it works in both Claude Code and the app from the start.

1. Create the folder. Its name becomes the command, so use lowercase and hyphens:

   ```bash
   mkdir -p .claude/skills/odm-run
   ```

2. Save `.claude/skills/odm-run/SKILL.md`. See [`example/.claude/skills/odm-run/SKILL.md`](example/.claude/skills/odm-run/SKILL.md) and adapt the command to the one you actually use.
3. Commit it: `git add .claude/skills && git commit -m "Add odm-run skill"`.

Rules that keep the skill uploadable to the app:

- Use only these frontmatter fields: `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`. Any other field, such as `argument-hint` or `context`, makes the app upload fail.
- Do not rely on `` !`command` `` lines or `$ARGUMENTS`. They are Claude Code features and do not work in the app.
- Put the main use case first in `description`. Claude decides when to load the skill from it.
- Keep `SKILL.md` under about 500 lines. Move long reference material into extra files in the same folder and link to them.

## Step 6: Test the skill in Claude Code

Claude Code picks up new and edited skills in a running session, with no restart.

1. From the project folder, start `claude`.
2. Run `/skills` and check that `odm-run` appears under the project skills.
3. Test both ways it can start:
   - Ask in plain words, for example "process the site puente-01". Claude should load the skill on its own.
   - Type `/odm-run` to invoke it directly.
4. If Claude ignores it, add the words you naturally use to the `description`, then try again.

Only move on to the app once the skill behaves well here.

## Step 7: Upload the skill to the Claude app

The app has no folder link to your project: you publish a skill by uploading it as a ZIP. Uploaded skills are private to your account.

1. Zip the skill folder itself, so the ZIP contains `odm-run/SKILL.md`:

   ```bash
   cd .claude/skills
   zip -r ~/odm-run.zip odm-run
   cd -
   ```

2. In the Claude app (desktop, web or mobile), open **Customize > Skills**.
3. Click **+**, then **Create skill**, then **Upload a skill**.
4. Select `odm-run.zip`.
5. Make sure the skill is toggled on.
6. Test it in a normal chat with a matching request.

In the app the skill runs in Claude's sandbox, not on your computer. Steps that need Docker or your local files only fully work in Claude Code or in a desktop session linked to your folders.

## Step 8: Understand how the sync works

Once a skill is enabled in the app, Claude Code downloads it automatically. Nothing ever uploads from your computer to the app.

| Where Claude runs | Which skills it loads |
| --- | --- |
| Claude Code on your computer | Project skills in `.claude/skills/`, personal skills in `~/.claude/skills/`, and your app skills (synced) |
| Claude app chat | Only skills enabled in Customize > Skills |
| Cowork and cloud sessions | App skills; cloud sessions also load skills committed in the repo's `.claude/skills/` |

How the download behaves:

- Synced skills land in `~/.claude/skills/synced/` when a session starts. Claude Code then checks for changes about every 10 minutes while you work, and about every 40 minutes when idle.
- `/skills` lists them under **claude.ai sync**.
- Editing files in `synced/` does nothing to the app copy, and the next sync can overwrite them.
- If a local skill has the same name, the local one wins for `/odm-run`. The synced copy then runs as `/anthropic-skills:odm-run`.

Never name a skill folder `synced`. Claude Code reserves that name and skips it.

## Step 9: Update a skill

Always edit the copy in your repository, then republish it. The loop takes about a minute:

1. Edit `.claude/skills/odm-run/SKILL.md` in the project.
2. Test it in Claude Code (`/odm-run`). The change applies without a restart.
3. Commit: `git commit -am "Update odm-run"`.
4. Re-zip the folder and upload it again in **Customize > Skills**.
5. Other machines and Cowork pick up the new version at their next sync.

To retire a skill, delete its folder in the repository and turn it off or delete it in **Customize > Skills**. Deleting only the `synced/` copy does not work: the next sync downloads it again.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Upload says "Unexpected key(s) in SKILL.md frontmatter" | Remove fields outside the six allowed ones (Step 5) |
| Upload rejects the ZIP | Zip the skill folder itself, not a parent folder or a GitHub "Download ZIP" |
| App skills don't appear in Claude Code | Check `claude --version` (2.1.273+), sign in with `/login`, restart, then run `/skills` |
| Skill never triggers | Put the words you actually use first in `description`, or invoke it with `/odm-run` |
| Skill works in Claude Code but not in the app | It needs local tools (Docker, your files); run that part in Claude Code |
| A synced edit vanished | Edit the repository copy and re-upload; `synced/` is overwritten |

## Sources

- [Extend Claude with skills](https://code.claude.com/docs/en/skills) (Claude Code docs: locations, frontmatter, sync from claude.ai)
- [Use skills in Claude](https://support.claude.com/en/articles/12512180-use-skills-in-claude) (Help Center: uploading in Customize > Skills)
- [Claude Code memory and CLAUDE.md](https://code.claude.com/docs/en/memory)
- [Claude Code settings](https://code.claude.com/docs/en/settings)
