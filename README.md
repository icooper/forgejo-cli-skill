# Forgejo CLI Skill

An agent skill that teaches coding agents (Claude, Codex and anything else that reads
the `SKILL.md` format) to use [`fj`](https://codeberg.org/forgejo-contrib/forgejo-cli),
the Forgejo command-line client, the way they already use `gh` for GitHub.

Without it, agents tend to guess `gh`-style commands that `fj` does not have
(`fj issue list`, `--json`, `fj api`), or hang waiting on a text editor. With it they know:

- how `fj` decides which host and repository a command applies to
- the commands for issues, pull requests, CI status, releases, tags, milestones, labels,
  wikis, organizations, teams and users
- how to stay non-interactive (no editor, no browser, no prompts)
- how to read `fj`'s plain-text output and its common errors
- what `fj` cannot do, and how to fall back to the Forgejo REST API for those cases

Written and checked against **fj v0.7.0**.

## What is in this repo

```
skills/forgejo-cli/
├── SKILL.md                    # loaded when the skill triggers: mental model, core commands, gh→fj table
└── references/
    ├── commands.md             # every fj command and flag
    └── api-fallback.md         # REST API recipes for what fj cannot do
```

The skill folder is `skills/forgejo-cli`. Installing means putting that folder (or a
symlink to it) where your agent looks for skills.

## Prerequisites

The skill only supplies knowledge; the machine the agent runs on needs `fj` itself.

1. Install `fj`: `brew install forgejo-cli`, `cargo install --locked forgejo-cli`,
   `apt install forgejo-cli` (Debian unstable / Ubuntu 25.10+), or see the
   [installation page](https://codeberg.org/forgejo-contrib/forgejo-cli/wiki/Installation).
2. Log in to your instance. On Codeberg and a few other public instances:

   ```bash
   fj -H codeberg.org auth login
   ```

   On a self-hosted instance, create a token at
   `https://YOUR-HOST/user/settings/applications` and store it:

   ```bash
   fj -H YOUR-HOST auth add-token
   ```

   The token's scopes are the limit of what the agent can do. A read-only token is a
   reasonable way to let an agent explore without being able to change anything.
3. Check it works:

   ```bash
   fj -H YOUR-HOST whoami
   ```

## Install

Clone this repo somewhere permanent first. Symlinking from the clone means a
`git pull` updates every agent at once.

```bash
git clone https://git.icoop.dev/vertekta/forgejo-cli-skill.git ~/src/forgejo-cli-skill
```

### Claude Code

For all your projects (personal skill):

```bash
mkdir -p ~/.claude/skills
```

```bash
ln -s ~/src/forgejo-cli-skill/skills/forgejo-cli ~/.claude/skills/forgejo-cli
```

For one project only, so teammates get it too, copy it into that repository and commit it:

```bash
mkdir -p .claude/skills && cp -R ~/src/forgejo-cli-skill/skills/forgejo-cli .claude/skills/
```

Claude Code picks up new skills in existing skill directories without a restart. If
`~/.claude/skills` or `.claude/skills` did not exist when the session started, run
`/reload-skills` or start a new session.

To check: type `/forgejo-cli` to load it explicitly, or just ask something like
"what issues are open on this repo?" in a Forgejo checkout and Claude should load it on
its own.

### Claude desktop and claude.ai

These take a ZIP upload instead of a folder on disk.

```bash
cd ~/src/forgejo-cli-skill/skills && zip -r ~/Downloads/forgejo-cli.zip forgejo-cli
```

Then open **Customize → Skills**, click **+**, choose **Create skill → Upload a skill**,
and pick `forgejo-cli.zip`. The ZIP must contain the `forgejo-cli` folder at its top
level, which the command above does.

Note that the skill runs `fj` in a shell, so it is only useful in Claude surfaces that
can run commands on a machine where `fj` is installed and logged in.

### Codex

Codex reads skills from `~/.agents/skills` (personal) and from `.agents/skills` in a
repository (project).

For all your projects:

```bash
mkdir -p ~/.agents/skills
```

```bash
ln -s ~/src/forgejo-cli-skill/skills/forgejo-cli ~/.agents/skills/forgejo-cli
```

For one project only:

```bash
mkdir -p .agents/skills && cp -R ~/src/forgejo-cli-skill/skills/forgejo-cli .agents/skills/
```

Codex detects new skills automatically; restart it if the skill does not show up.
To check: run `/skills` to see the list, or mention it explicitly with `$forgejo-cli`.

### Other agents

Any agent that supports the `SKILL.md` convention can use the same folder: place or
symlink `skills/forgejo-cli` in that agent's skills directory. Agents without skill
support can still be pointed at `skills/forgejo-cli/SKILL.md` as a reference document.

## Updating

```bash
git -C ~/src/forgejo-cli-skill pull
```

Symlinked installs update immediately. Copied installs (project folders, ZIP uploads)
need the copy or upload repeated.

## Keeping it current

`fj` is under active development and its commands change between releases. When you
upgrade `fj`, compare `fj <command> --help` with
[`skills/forgejo-cli/references/commands.md`](skills/forgejo-cli/references/commands.md)
and update the version noted at the top of `SKILL.md`. Two things worth watching for
upstream, because the skill currently works around their absence: a `fj api` subcommand
and machine-readable output.
