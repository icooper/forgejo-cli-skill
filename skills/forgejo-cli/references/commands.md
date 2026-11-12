# fj command reference (v0.7.0)

Every command and its flags, condensed from `fj ... --help`. Run `--help` on the
installed binary if a flag here is rejected - the tool changes between versions.

## Contents

- [Conventions](#conventions)
- [Global flags](#global-flags)
- [auth, whoami, version, completion](#auth-whoami-version-completion)
- [repo](#repo)
- [issue](#issue)
- [pr](#pr)
- [milestone](#milestone)
- [release](#release)
- [tag](#tag)
- [actions](#actions)
- [wiki](#wiki)
- [user](#user)
- [org](#org)

## Conventions

- `<X>` required, `[X]` optional, `...` repeatable.
- `ISSUE` / `PR` is a number (`12`) or a number with its repo (`owner/repo#12`,
  `host/owner/repo#12`).
- `REPO` is `owner/name` or `host/owner/name`.
- "editor" marks a value that opens a text editor when omitted. Agents must always
  supply it.
- "browser" marks commands that launch a web browser. Agents should avoid them.
- Boolean setters written `<true|false>` need the literal word.

## Global flags

| Flag | Meaning |
|------|---------|
| `-H, --host <HOST>` | Instance to talk to, e.g. `codeberg.org` or a full URL |
| `-C, --cwd <DIR>` | Run as if started in `DIR` |
| `--style <fancy\|minimal>` | Output style. Pipes always get `minimal` (no colour, ASCII markers) |
| `-R, --remote <NAME>` | On repo-scoped commands: which local git remote identifies the repo |
| `-h, --help` | Help for any command or subcommand |

Environment: `FJ_FALLBACK_HOST` (full URL) is used when no host can be found otherwise.
`core.editor` from git config, then `$EDITOR`, is the editor fj launches.

## auth, whoami, version, completion

| Command | Notes |
|---------|-------|
| `fj auth list` | Hosts with stored logins |
| `fj auth login` | OAuth via browser; only for instances fj has a client ID for (Codeberg and some others). Browser |
| `fj auth add-token [TOKEN]` | Store an application token; reads stdin if omitted. Token is created at `https://HOST/user/settings/applications` |
| `fj auth logout <HOST>` | Remove a stored login |
| `fj auth use-ssh [true\|false]` | Default to SSH for clones and remotes on this host |
| `fj whoami [-r REMOTE]` | Logged-in user on the current host |
| `fj version [-v]` | Version; `-v` adds user agent, build type, target |
| `fj completion <bash\|zsh\|fish\|elvish\|power-shell\|nushell>` | Shell completion script |

## repo

| Command | Notes |
|---------|-------|
| `fj repo view [REPO]` | Description, language, star/issue/PR/release counts, URL |
| `fj repo readme [REPO]` | Print the README; errors if there is none |
| `fj repo clone <REPO> [PATH]` | `-S/--ssh [true\|false]`, `-I/--identity-file FILE` |
| `fj repo create <NAME>` | In your own account. `-d DESC`, `-P/--private`, `-r/--remote NAME` (add a remote), `-p/--push` (push current branch; implies remote `origin`), `-S/--ssh` |
| `fj repo fork <REPO>` | `--name NEW`, `-R/--remote NAME` |
| `fj repo migrate <URL> <[OWNER/]NAME>` | `-m/--mirror`, `-p/--private`, `-i/--include lfs,wiki,issues,prs,milestones,labels,releases\|all`, `-s/--service git\|github\|gitlab\|forgejo\|gitea\|gogs\|...`, `-L/--lfs-endpoint URL`, `-t/--token` or `-l/--login` (both read stdin) |
| `fj repo edit [REPO]` | `-d DESC`, `--name NAME`, `-p/--private <true\|false>`, `-a/--archived <true\|false>`, `-t/--template <true\|false>`, `--default-branch B`, `-w/--website URL`, `-A/--avatar FILE`, `-u/--unset-avatar`, `--enable-prune <true\|false>`, `--mirror-interval 8h30m0s` |
| `fj repo delete <REPO>` | Irreversible. Prompts `y/N` on stdin |
| `fj repo star [REPO]` / `unstar` | |
| `fj repo star-status [REPO]` | `--list` lists everything you starred |
| `fj repo watch [REPO]` / `unwatch` | |
| `fj repo watch-status [REPO]` | `--list` lists everything you watch |
| `fj repo browse [REPO]` | Browser |

### repo labels

The repo is a positional *before* the subcommand: `fj repo labels owner/repo view`.

| Command | Notes |
|---------|-------|
| `fj repo labels [REPO] view` | `-a/--archived` to include archived. Prints name, numeric ID, description |
| `fj repo labels [REPO] create <NAME> <COLOR>` | Colour is hex. `-d DESC` (editor if bare), `-e/--exclusive`, `-a/--archived`. Use `scope/name` to namespace |
| `fj repo labels [REPO] edit <ID\|NAME>` | `-n NAME`, `-c COLOR`, `-d DESC`, `-e <true\|false>`, `-a <true\|false>` |
| `fj repo labels [REPO] delete <ID\|NAME>` | |

### repo units

Turn repo features on and off: `fj repo units [REPO] <unit> -e/--enable <true|false>`.
Units: `issues`, `prs`, `actions`, `wiki`, `packages`, `projects`, `releases`.

- `wiki` also takes `--branch B`, `--external-url URL`, `--globally-editable <true|false>`.
- `prs` also takes `--allow-merge-commits`, `--allow-rebase`, `--allow-rebase-explicit`,
  `--allow-rebase-update`, `--allow-squash-merge`, `--allow-fast-forward-only-merge`,
  `--allow-manual-merge`, `--autodetect-manual-merge`, `--default-allow-maintainer-edit`,
  `--default-delete-branch-after-merge`, `--ignore-whitespace-conflicts`
  (each `<true|false>`), `--default-merge-style merge|rebase|rebase-merge|squash|fast-forward-only`,
  `--default-update-style rebase|merge`.

## issue

| Command | Notes |
|---------|-------|
| `fj issue search [QUERY]` | `-r REPO`, `-s/--state open\|closed\|all` (default open), `-l/--labels A,B`, `-c/--creator USER`, `-a/--assignee USER`, `-m/--milestones A,B`, `-p/--page N` (25 per page), `--all` |
| `fj issue view <ISSUE>` | Title, author, state, labels, body, comment count |
| `fj issue view <ISSUE> comments` | All comments |
| `fj issue view <ISSUE> comment <IDX>` | One comment, 0-indexed |
| `fj issue view <ISSUE> assignees` | |
| `fj issue create <TITLE>` | `-r REPO`, `--body TEXT` (editor), `--body-file FILE\|-`, `--template NAME`, `--no-template`, `--milestone NAME`, `--web` (browser) |
| `fj issue templates` | `-r REPO`. Lists template names usable with `--template` |
| `fj issue comment <ISSUE> [BODY]` | Editor if no body; or `--body-file FILE\|-` |
| `fj issue edit <ISSUE> title [TEXT]` | Editor if omitted |
| `fj issue edit <ISSUE> body [TEXT]` | Editor if omitted |
| `fj issue edit <ISSUE> comment <IDX> [TEXT]` | 0-indexed. Editor if omitted |
| `fj issue edit <ISSUE> labels` | `-a/--add NAME`, `-r/--rm NAME`; repeat for several. Here `-r` is `--rm`, not `--repo` |
| `fj issue edit <ISSUE> milestone <NAME>` | or `-u/--unset` |
| `fj issue assign <ISSUE> [USERS]...` / `unassign` | |
| `fj issue close <ISSUE>` | `-w/--with-msg [TEXT]` adds a comment first (editor if bare); the comment stays even if the close then fails |
| `fj issue depend add <ISSUE> [TARGET]...` | TARGETs become dependencies of ISSUE |
| `fj issue depend remove <ISSUE> [TARGET]...` | Works, but reports `unexpected status code 201 Created` and exits 1 |
| `fj issue depend list <ISSUE>` | |
| `fj issue block add <ISSUE> [TARGET]...` | ISSUE blocks the TARGETs |
| `fj issue block remove <ISSUE> [TARGET]...` | Same spurious `201 Created` error as `depend remove` |
| `fj issue block list <ISSUE>` | |
| `fj issue browse <ISSUE>` | Browser |

## pr

Where `[PR]` is optional, fj guesses the PR from the current branch's upstream when it
is left out. That only works inside a checkout whose branch has an open PR.

| Command | Notes |
|---------|-------|
| `fj pr search [QUERY]` | Same flags as `issue search` |
| `fj pr view [PR]` | Title, author, state, `+N -M`, head and base branches, body |
| `fj pr view [PR] diff` | `-p/--patch` for `git format-patch` style, `-e/--editor` (editor) |
| `fj pr view [PR] files` | Changed files with +/- counts |
| `fj pr view [PR] commits` | `-o/--oneline` |
| `fj pr view [PR] comments` / `comment <IDX>` | 0-indexed |
| `fj pr view [PR] labels` / `assignees` | |
| `fj pr status [PR]` | Prints `Open — Can be merged` (or the merged state) plus one line per CI check, e.g. `- Success — ci/build`. `--wait` blocks until checks finish (exit code stays 0 on failed checks). Errors with `not properly structured` on an open PR that has no checks |
| `fj pr create [TITLE]` | `-r REPO`, `--base BRANCH` (`^` or `^BRANCH` targets the parent of a fork), `--head BRANCH`, `--body TEXT` (editor), `--body-file FILE\|-`, `-A/--autofill`, `-a/--agit`, `--milestone NAME`, `-w/--web` (browser). Prefix the title with `WIP: ` for a draft |
| `fj pr checkout <PR>` | `--branch-name NAME` (default `pr-<host>-<owner>-<N>`), `-S/--ssh`, `-I/--identity-file`. `^N` takes the PR from the parent repo. Refuses on a dirty tree |
| `fj pr comment [PR] [BODY]` | Give both: a single positional is taken as the PR. `--body-file FILE\|-` |
| `fj pr edit [PR] title [TEXT]` | Editor if omitted |
| `fj pr edit [PR] body [TEXT]` | Editor if omitted |
| `fj pr edit [PR] comment <IDX> [TEXT]` | 0-indexed. Editor if omitted |
| `fj pr edit [PR] labels` | `-a/--add NAME`, `-r/--rm NAME` |
| `fj pr edit [PR] milestone <NAME>` | or `-u/--unset` |
| `fj pr assign [USERS]...` / `unassign` | PR given with `-p/--pr N` |
| `fj pr review [PR] list` | `-a/--all` includes stale and dismissed, `-c/--comments` shows inline comments |
| `fj pr review [PR] request [USERS]...` | `-t/--team NAME` for team reviewers. The PR author cannot be a reviewer |
| `fj pr merge [PR]` | `-M/--method merge\|rebase\|rebase-merge\|squash\|manual\|fast-forward-only`, `-d/--delete` (delete branch), `-t/--title TEXT`, `-m/--message [TEXT]` (editor if bare) |
| `fj pr close [PR]` | `-w/--with-msg [TEXT]` (editor if bare) |
| `fj pr depend add\|remove [TARGET]...` / `list` | PR given with `-p/--pr N` |
| `fj pr block add\|remove [TARGET]...` / `list` | PR given with `-p/--pr N` |
| `fj pr browse [PR]` | Browser |

## milestone

All take `-r/--repo REPO`.

| Command | Notes |
|---------|-------|
| `fj milestone search` | `--state open\|closed\|all` (default open), `--title TEXT` |
| `fj milestone view <TITLE\|ID>` | State, open/closed counts, dates |
| `fj milestone create <TITLE> --state <open\|closed>` | `--state` is required. `--description [TEXT]` (editor if bare) |
| `fj milestone edit <TITLE>` | `--new-title T`, `--description T`, `--state open\|closed` |
| `fj milestone delete <TITLE\|ID>` | |

## release

All take `-r/--repo REPO`.

| Command | Notes |
|---------|-------|
| `fj release list` | Regular releases only. Despite the names, `-p/--include-prerelease` lists *only* prereleases and `-d/--include-draft` *only* drafts (both: only releases that are both) |
| `fj release view <NAME>` | `-t/--by-tag` looks up by tag instead of release name |
| `fj release create <NAME>` | One of `-t/--tag EXISTING` or `-T/--create-tag [TAG]` (defaults to NAME) is required. `-b/--body [TEXT]` (editor if bare), `-B/--branch B`, `-a/--attach FILE[:ASSET_NAME]` (repeatable), `-d/--draft`, `-p/--prerelease` |
| `fj release edit <NAME>` | `-n/--rename NEW`, `-t/--tag TAG`, `-b/--body [TEXT]`, `-d/--draft <true\|false>`, `-p/--prerelease <true\|false>` |
| `fj release delete <NAME>` | `-t/--by-tag` |
| `fj release asset create <RELEASE> <PATH> [NAME]` | Upload a file |
| `fj release asset download <RELEASE> <ASSET>` | `-o/--output FILE` |
| `fj release asset delete <RELEASE> <ASSET>` | |
| `fj release browse [NAME]` | Browser |

## tag

All take `-r/--repo REPO`.

| Command | Notes |
|---------|-------|
| `fj tag list` | `-p/--page N` |
| `fj tag view <NAME>` | Tag name, commit SHA, message |
| `fj tag create <NAME>` | `-B/--branch B`, `-b/--body [TEXT]` (editor if bare) |
| `fj tag delete <NAME>` | |

## actions

All take `-r/--repo REPO`.

| Command | Notes |
|---------|-------|
| `fj actions tasks` | Recent runs, 20 per page, `-p/--page N`. Line format: `#RUN (SHA) STATUS WORKFLOW DURATION (EVENT): COMMIT TITLE`. `not found` when the repo has Actions disabled |
| `fj actions dispatch <WORKFLOW> <REF>` | `-I/--inputs KEY=VALUE` (repeatable). WORKFLOW is the file name, e.g. `publish.yaml` |
| `fj actions variables list` | `-v/--verbose` |
| `fj actions variables create <NAME> [DATA]` | Editor if DATA omitted. `-f/--force` overwrites |
| `fj actions variables delete <NAME>` | |
| `fj actions secrets list` | Names only |
| `fj actions secrets create <NAME> <DATA>` | The value ends up in shell history and process lists; prefer having the user run it |
| `fj actions secrets delete <NAME>` | |

There is no command for run logs, re-runs or cancellation - see `api-fallback.md`.

## wiki

All take `-r/--repo REPO`. Read-only: edit a wiki by cloning it and pushing with git.

| Command | Notes |
|---------|-------|
| `fj wiki contents` | Page names. `not found` when the repo has no wiki |
| `fj wiki view <PAGE>` | Print a page |
| `fj wiki clone` | `-p/--path DIR`, `-S/--ssh`, `-I/--identity-file` |
| `fj wiki browse <PAGE>` | Browser |

## user

`[USER]` defaults to yourself. Outside a checkout, add `-H HOST`.

| Command | Notes |
|---------|-------|
| `fj user view [USER]` | Name, followers, website, email, join date |
| `fj user search <QUERY>` | `-p/--page N` |
| `fj user repos [USER]` | `--starred`, `--sort name\|modified\|created\|stars\|forks`, `--page N` |
| `fj user orgs [USER]` | |
| `fj user activity [USER]` | Recent pushes, repo creations, etc. |
| `fj user followers [USER]` / `following [USER]` | |
| `fj user follow <USER>` / `unfollow` / `block` / `unblock` | |
| `fj user edit bio [TEXT]` | Editor if omitted |
| `fj user edit name\|pronouns\|location\|website [VALUE]` | `-u/--unset` clears |
| `fj user edit activity -v <hidden\|public>` | |
| `fj user edit email` | `-v/--visibility hidden\|public`, `-a/--add ADDR`, `-r/--rm ADDR` |
| `fj user key list` | `-v/--verbose`. SSH keys |
| `fj user key view <ID>` / `delete <ID>` | |
| `fj user key upload [KEYFILE]` | `-t/--title T`, `-r/--read-only`, `-f/--force`; `-` reads stdin |
| `fj user gpg list` / `view <ID>` / `delete <ID>` / `upload <KEY>` / `verify <ID>` | `delete` asks for confirmation unless `-f`; `upload` and `verify` need local `gpg`; `upload -n/--no-verify` skips verification |
| `fj user browse [USER]` | Browser |

## org

| Command | Notes |
|---------|-------|
| `fj org list` | `-o/--only-member-of`, `-p/--page N` |
| `fj org view <NAME>` | Visibility, member and team counts |
| `fj org activity <NAME>` | Recent activity |
| `fj org members <ORG>` | `-p/--page N` |
| `fj org visibility <ORG>` | Your own membership visibility; `-s/--set private\|public` |
| `fj org create <NAME>` | `-f/--full-name`, `-d/--description`, `-e/--email`, `-l/--location`, `-w/--website`, `-v/--visibility private\|limited\|public`, `-a/--admin-can-change-team-access <true\|false>` |
| `fj org edit <NAME>` | Same flags as create. Renaming is web-only |
| `fj org repo list <ORG>` | `-p/--page N` |
| `fj org repo create <ORG> <REPO>` | Same flags as `fj repo create` |
| `fj org label list <ORG>` | Org-wide labels, available in all its repos |
| `fj org label add <ORG> <NAME> <COLOR>` | `-d DESC`, `-e/--exclusive` |
| `fj org label edit <ORG> <NAME>` | `-n/--new-name`, `-c/--color`, `-d`, `-e`, `-a/--archived <true\|false>` |
| `fj org label rm <ORG> <LABEL>` | |
| `fj org team list <ORG>` | |
| `fj org team view <ORG> <NAME>` | `-p/--list-permissions` |
| `fj org team create <ORG> <NAME>` | `-d DESC`, `-c/--can-create-repos`, `-i/--include-all-repos`, `-A/--admin`, `-r/--read-permissions LIST`, `-w/--write-permissions LIST` |
| `fj org team edit <ORG> <NAME>` | `-n/--new-name`; the same flags, with booleans as `<true\|false>` |
| `fj org team delete <ORG> <NAME>` | Does not delete the team's repos |
| `fj org team repo list\|add\|rm <ORG> <TEAM> [REPO]` | Repo access for a team |
| `fj org team member list\|add\|rm <ORG> <TEAM> [USER]` | Team membership |

Team permission lists are comma-separated from: `wiki`, `ext_wiki`, `issues`,
`ext_issues`, `pulls`, `projects`, `actions`, `code`, `releases`, `packages`, or `all`.
