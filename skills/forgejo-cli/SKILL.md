---
name: forgejo-cli
description: Use the `fj` command-line tool (forgejo-cli) to work with Forgejo instances such as Codeberg or a self-hosted forge - issues, pull requests, repositories, releases, tags, milestones, labels, wikis, Actions/CI runs, organizations, teams and users. Use this skill whenever a task touches a Forgejo-hosted repository (reading or filing issues, opening, reviewing, checking out or merging PRs, checking CI, cutting releases) or mentions `fj`, forgejo-cli, Forgejo or Codeberg, even if the user only says "open a PR" or "look at issue 12" and the git remote is not GitHub or GitLab. Also use it when you would normally reach for `gh`, `glab` or `tea` but the remote is a Forgejo instance - `fj` has different command names, no JSON output and no `api` subcommand, so guessing from `gh` habits fails.
---

# Forgejo CLI Skill

`fj` (the `forgejo-cli` project) is to Forgejo what `gh` is to GitHub: it handles the
forge-specific things `git` does not - issues, pull requests, releases, CI status, orgs.
It works alongside `git`; it does not replace it.

This skill was written against **fj v0.7.0** talking to Forgejo 17. The tool is young and changes between
releases, so treat `fj <command> --help` as the final word when something here disagrees
with what you see. `fj version` prints the installed version (`fj --version` does not exist).

## How fj differs from gh (read this first)

These are the habits that make `gh`-trained agents stumble:

- **No JSON, no `fj api`.** Output is human-readable text only. There is no `--json`,
  `--jq` or raw API passthrough. When you need structured data or something `fj` cannot
  do, use the REST API directly - see [references/api-fallback.md](references/api-fallback.md).
- **`search`, not `list`.** `fj issue search` and `fj pr search` list issues and PRs
  (with no query they list everything open). There is no `issue list` or `pr list`.
- **The ID comes before the sub-action.** `fj issue view 12 comments`,
  `fj issue edit 12 title "New title"`, `fj pr view 12 diff`. Not `--comments`, not `--title`.
- **The repo usually travels inside the ID.** `fj issue view owner/repo#12`. Only some
  commands take `-r/--repo` (see "Pointing fj at the right repo").
- **It opens a text editor whenever a body is missing.** In an agent session that means
  a hang or an `unable to locate editor` error. Always pass the text yourself
  (see "Never let fj open an editor or browser").
- **Drafts are a title prefix.** A PR is a draft when its title starts with `WIP: `.
- **Anonymous reads work.** With no login for a host, `fj` still reads public data, so
  `fj -H codeberg.org issue search -r owner/repo` works without an account. Anything
  private or any write needs a login.

## Orient yourself

```bash
fj version              # installed version
fj auth list            # hosts fj is logged in to
fj whoami               # user on the current repo's host (use -H HOST outside a repo)
fj repo view            # the repo fj thinks you are in - confirms host and owner/name
git remote -v           # where the remotes actually point
```

Not sure a host is Forgejo? `curl -s https://HOST/api/v1/version` returns something like
`{"version":"17.0.1+gitea-1.22.0"}` on Forgejo. If `fj` is not installed, tell the user;
install options are `brew install forgejo-cli`, `cargo install --locked forgejo-cli`,
`apt install forgejo-cli`, or a binary from <https://codeberg.org/forgejo-contrib/forgejo-cli/releases>.

Do not log in on the user's behalf. `fj auth login` opens a browser and
`fj auth add-token` needs a secret; if a host is not logged in, ask the user to run one
of them. Never read or print fj's credential file.

## Pointing fj at the right repo

Inside a git checkout whose remote is a Forgejo instance, `fj` works it out from the
remote and you can use bare numbers: `fj issue view 12`. With several remotes it uses
the one the current branch tracks; override with `-R <remote-name>`.

Anywhere else, or for a different repo, be explicit. There are three forms:

| Form | Example | Notes |
|------|---------|-------|
| Repo inside the ID | `fj issue view owner/repo#12` | Works for every command that takes an issue or PR ID |
| `-r/--repo` flag | `fj issue search -r owner/repo` | Only on `issue search/create/templates`, `pr search/create`, and all of `milestone`, `wiki`, `actions`, `release`, `tag` |
| Positional repo | `fj repo view owner/repo` | `repo` subcommands |

The host comes from, in order: a host inside the repo spec (`codeberg.org/owner/repo` or
`codeberg.org/owner/repo#12`), then `-H/--host`, then the local git remote, then the
`FJ_FALLBACK_HOST` environment variable (must be a full URL such as `https://codeberg.org`).
Outside a git repo with none of those you get `cannot find repo, no host specified`.

```bash
fj -H git.example.com issue search -r team/app       # host flag + repo flag
fj issue view git.example.com/team/app#12            # everything in the ID
fj -C ~/src/app pr status                            # run as if in another directory
```

Quote IDs that contain `#` if your shell might treat it specially (`'owner/repo#12'`).

Commands that are not about one repo (`fj user ...`, `fj org ...`, `fj whoami`) still
need a host: run them inside a checkout or pass `-H`.

## Never let fj open an editor or browser

`fj` opens `core.editor`/`$EDITOR` whenever body text is left out, and some commands open
a web browser. Neither works for an agent. Supply everything on the command line:

| Command | What to pass |
|---------|--------------|
| `fj issue create "Title"` | `--body "..."` or `--body-file FILE` (`-` reads stdin) |
| `fj issue comment 12` | the body as a positional: `fj issue comment 12 "text"`, or `--body-file` |
| `fj issue edit 12 title` / `body` / `comment IDX` | the new text as the last positional |
| `fj issue close 12 -w` | give `-w` a value: `-w "reason"` (bare `-w` opens the editor) |
| `fj pr create "Title"` | `--body "..."`, `--body-file FILE`, or `-A/--autofill` |
| `fj pr comment 12 "text"` | always give the PR number; a lone positional is parsed as the PR |
| `fj pr merge -m` | give `-m` a value, or leave `-m` off for the default message |
| `fj release create` / `edit`, `fj tag create` | give `-b/--body` a value (bare `-b` opens the editor) |
| `fj milestone create` | give `--description` a value, or omit the flag |
| `fj actions variables create NAME` | pass the value as the second positional |

For multi-line text, write it to a file and use `--body-file`, or pipe it:
`printf '%s\n' "$BODY" | fj issue create "Title" --body-file -`.

Avoid in agent sessions: every `browse` subcommand and `--web` flag (they launch a
browser), `fj auth login` (browser OAuth), and `fj repo delete` (asks `y/N` on stdin).

## Reading: the commands you will use most

```bash
# Issues
fj issue search                              # open issues, 25 per page
fj issue search "crash" -s all --all         # text query, any state, every page
fj issue search -l Kind/Bug -a alice -c bob  # by label, assignee, creator
fj issue view 12                             # title, state, labels, body
fj issue view 12 comments                    # every comment
fj issue view 12 comment 0                   # one comment; indexes start at 0

# Pull requests
fj pr search -s all                          # same flags as issue search
fj pr view 34                                # title, author, state, +/- lines, branches, body
fj pr view 34 diff                           # unified diff (add --patch for git-am format)
fj pr view 34 files                          # changed files with +/- counts
fj pr view 34 commits --oneline
fj pr view 34 comments
fj pr status 34                              # mergeability + each CI check
fj pr status 34 --wait                       # block until checks finish
fj pr review 34 list --all --comments        # reviews, incl. stale ones and inline comments

# Repo, releases, CI
fj repo view owner/repo                      # description, counts, URL
fj repo readme owner/repo
fj repo labels owner/repo view               # label names (needed for -l and edit labels)
fj release list -r owner/repo                # regular releases only; -p = only prereleases, -d = only drafts
fj release view v1.2.0 -r owner/repo
fj tag list -r owner/repo
fj milestone search -r owner/repo --state all
fj actions tasks -r owner/repo               # recent CI runs: number, sha, status, workflow, time
fj wiki contents -r owner/repo ; fj wiki view Home -r owner/repo

# People and orgs
fj user view alice ; fj user repos alice ; fj user activity alice
fj org list -o ; fj org repo list myorg ; fj org members myorg ; fj org team list myorg
```

Inside a checkout, PR commands accept no ID at all and guess the PR from the current
branch's upstream: `fj pr view`, `fj pr status`, `fj pr view diff`. Issue commands
always need the number.

### Interpreting output

- Piped output is plain: no colour, `-` bullets, and body text prefixed with `> `.
  Strip the `> ` before quoting a body back to someone.
- Search results look like `#12: Title (by author)`, newest first, with no dates. The
  first line is a count (`23 issues`) or, when there is more than one page,
  `Page 1 out of 4 pages in total (25 issues)` - there the number is how many are on
  this page, not the total. If you see the page form, you have not seen everything:
  add `--all` or step through with `-p N`. No matches prints
  `Page 1 out of 0 pages in total (0 issues)`.
- A label filter that does not match an existing label is silently ignored and returns
  unfiltered results. Label names are exact, case-sensitive and often scoped
  (`Kind/Bug`), so look them up first with `fj repo labels view`. That one command
  prints colour codes even in a pipe, and shows *exclusive* scoped labels as two words
  (`Priority  Critical`) when the real name is `Priority/Critical`. The trailing number
  is the label ID. To read it cleanly: `fj repo labels view | sed $'s/\x1b\\[[0-9;]*m//g'`.
- Release and tag lists are newest first and also print no dates; `fj release view`
  shows the date. `fj release list` shows only regular releases. `-p` and `-d` are
  filters, not additions: `-p` lists only prereleases, `-d` only drafts, and both
  together only releases that are both. An empty list may just mean the wrong filter.
- Most write commands print nothing when they succeed; exit code 0 is the confirmation.
  `issue create` and `pr create` print `created issue #N: Title` /
  `created pull request #N: Title` - take the number from there.
- Pull requests are issues in Forgejo's data model, so the `fj issue` commands (`view`,
  `comment`, `edit`, `close`) accept PR numbers too. The reverse is not true:
  `fj pr view N` on a plain issue fails with `not found`.
- `fj actions tasks` and `fj wiki contents` answer `not found` when the repo simply has
  Actions or the wiki switched off (some forges run CI elsewhere; `fj pr status` still
  shows those checks).
- Errors go to stderr as `Error: ...` followed by a `Location: src/...` line, with exit
  code 1. `not found` usually means a wrong repo, host or number - or a private repo
  the current login cannot see. `unauthorized`/`forbidden` means no login or a token
  without the needed scope: report that to the user instead of retrying.
- `fj pr status --wait` prints dots while waiting and exits 0 even when checks fail, so
  read the final lines rather than relying on the exit code.
- `fj pr status` on an open PR that has no CI checks at all fails with
  `the response from forgejo was not properly structured ... unknown variant`. That
  means "no checks reported", not a broken PR; use `fj pr view N` for its state.

## Writing: issues and pull requests

```bash
# Issues
fj issue create "Title" --body "Details" -r owner/repo
fj issue comment 12 "Thanks, fixed in #34"
fj issue edit 12 title "Clearer title"
fj issue edit 12 body "Replacement body"
fj issue edit 12 labels --add Kind/Bug --rm Status/Triage    # repeat flags for several
fj issue edit 12 milestone v1.2          # or --unset
fj issue assign 12 alice bob ; fj issue unassign 12 bob
fj issue close 12 -w "Fixed in v1.2"

# Pull requests (push the branch first with git; fj does not push)
git push -u origin my-branch
fj pr create "Add retry logic" --body "Why and what" --base main
fj pr create -A                          # title and body from the commits
fj pr create "WIP: spike" --body "..."   # draft
fj pr create "Fix" --body "..." --base ^ # from a fork, against the parent repo's default branch
fj pr create "Fix" --body "..." --agit   # no fork or push needed (AGit flow)
fj pr comment 34 "Addressed the review"
fj pr edit 34 title "Better title" ; fj pr edit 34 body "New body"
fj pr edit 34 labels --add Kind/Feature
fj pr assign --pr 34 alice               # note: --pr flag, not a positional
fj pr review 34 request alice --team backend
fj pr checkout 34                        # local branch pr-<host>-<owner>-34; needs a clean tree
fj pr merge 34 --method squash --delete  # methods: merge, rebase, rebase-merge, squash, fast-forward-only, manual
fj pr close 34 -w "Superseded by #40"
```

Things that are easy to get wrong:

- If a repo has issue templates, `fj issue create` refuses unless you pass
  `--template NAME` or `--no-template`. `fj issue templates` lists them: the template
  name is the first word of each entry (entries that are URLs are contact links, not
  templates), and a final line `'--no-template' is not allowed` means blank issues are
  disabled. Pass `--body`/`--body-file` together with `--template`, or the editor opens.
  For a form-style template the body has to be the filled-in form; see
  "Issue forms" below.
- In v0.7.0 `fj issue create --milestone` only takes effect together with `--template`.
  Otherwise set it afterwards with `fj issue edit N milestone NAME`.
- `fj pr create` needs a title unless `--autofill` is given. The head defaults to the
  current branch's upstream, so an unpushed branch fails with
  `config value 'branch.NAME.remote' was not found`; the base defaults to the repo's
  default branch. A `not found` from `pr create` straight after a push (seen on a
  brand-new repo) clears up when you pass `--base` explicitly or retry a moment later.
- Adding a label that does not exist prints `'X' doesn't exist` but still exits 0; any
  valid labels in the same command are still applied.
- `fj issue close N -w "msg"` (and `fj pr close`) posts the comment first. If the close
  then fails - for example the issue still has open dependencies - the comment stays,
  so do not repeat the `-w` on the retry.
- `depend remove` and `block remove` do remove the link but then report
  `unexpected status code 201 Created` and exit 1. Confirm with `depend list` /
  `block list` instead of retrying.
- A PR's author cannot be requested as its reviewer (`poster of pr can't be reviewer`).
- `fj pr merge` with no `--method` uses the repo's default merge style.
- There is no command to reopen an issue or PR, or to submit a review (approve /
  request changes). Use the REST API for those.

### Issue forms

Form templates expect the body in a specific layout. To see that layout without
creating anything, make the "editor" print the form and fail, which aborts the command
before it submits:

```bash
EDITOR='sh -c '"'"'cat "$1" >&2; exit 1'"'"' --' fj issue create "x" -r owner/repo --template NAME
```

It ends with `Error: editor exited unsuccessfully`, which is the point. (If git's
`core.editor` is set it takes priority over `$EDITOR`.) Fill the form in - keep the
headings and fences, change only the answer areas - save it to a file, and submit with
`--template NAME --body-file FILE`.

Forms often contain required checkboxes: a code of conduct, a licence sign-off, or a
declaration that the report was not written by an AI. Those are the user's promises to
make, not yours. Show them to the user and let them decide before ticking anything.

### Before you change anything

Creating, closing, merging, releasing and deleting are visible to other people and some
cannot be undone. Do them when the user asked for that outcome; when it is your own
idea, check first. Read the target (`fj issue view`, `fj pr status`) before you change
or close it.

## What fj cannot do

No equivalent exists for: raw API calls or JSON output, reopening issues/PRs, submitting
PR reviews, listing branches or commits, reading file contents, CI logs or re-runs,
searching repos or code across the instance, cross-repo issue search, notifications,
packages, webhooks, or branch protection. Use `git` for anything git can answer
(branches, log, file contents - without a checkout, `git ls-remote --heads URL` lists
branches) and the REST API for the rest:
[references/api-fallback.md](references/api-fallback.md).

## Coming from gh

| gh | fj |
|----|----|
| `gh auth status` | `fj auth list` / `fj whoami` |
| `gh repo view [o/r]` | `fj repo view [o/r]` |
| `gh repo clone o/r` | `fj repo clone o/r` |
| `gh repo fork o/r` | `fj repo fork o/r` |
| `gh repo create name --private` | `fj repo create name -P` (in an org: `fj org repo create ORG name`) |
| `gh issue list` | `fj issue search` |
| `gh issue list --search "x" --state all` | `fj issue search "x" -s all` |
| `gh issue view 12 --comments` | `fj issue view 12` then `fj issue view 12 comments` |
| `gh issue create -t T -b B` | `fj issue create "T" --body "B"` |
| `gh issue comment 12 -b B` | `fj issue comment 12 "B"` |
| `gh issue edit 12 --add-label L` | `fj issue edit 12 labels --add L` |
| `gh issue close 12 -c "msg"` | `fj issue close 12 -w "msg"` |
| `gh issue reopen 12` | not available - REST API |
| `gh pr list` | `fj pr search` |
| `gh pr view 34` | `fj pr view 34` |
| `gh pr diff 34` | `fj pr view 34 diff` |
| `gh pr checks 34 [--watch]` | `fj pr status 34 [--wait]` |
| `gh pr create -t T -b B -B main` | `fj pr create "T" --body "B" --base main` |
| `gh pr create --fill` | `fj pr create -A` |
| `gh pr create --draft` | title prefix `WIP: ` |
| `gh pr checkout 34` | `fj pr checkout 34` |
| `gh pr merge 34 --squash --delete-branch` | `fj pr merge 34 --method squash --delete` |
| `gh pr review 34 --approve` | not available - REST API |
| `gh pr edit 34 --add-reviewer u` | `fj pr review 34 request u` |
| `gh release list` / `view TAG` | `fj release list` / `fj release view NAME` (`--by-tag` to look up by tag) |
| `gh release create v1 -n notes file` | `fj release create v1 --create-tag -b "notes" -a file` |
| `gh run list` | `fj actions tasks` |
| `gh workflow run ci.yml --ref main -f k=v` | `fj actions dispatch ci.yml main -I k=v` |
| `gh secret set N -b V` / `gh variable set` | `fj actions secrets create N V` / `fj actions variables create N V` |
| `gh api ...` | not available - `curl` the REST API |
| `-R owner/repo` | `-r owner/repo` or `owner/repo#N` (in fj, `-R` means a git *remote name*) |

## Going deeper

- [references/commands.md](references/commands.md) - every command and flag, grouped by
  area. Read it when you need a command not shown above (repo settings, labels,
  milestones, releases and assets, Actions variables/secrets, orgs and teams, SSH/GPG keys).
- [references/api-fallback.md](references/api-fallback.md) - how to reach the Forgejo
  REST API for the things `fj` cannot do, with ready-made `curl` recipes.
- `fj <command> [<subcommand>] --help` - always current for the installed version.
- Upstream docs: `fj wiki view <Page> -r codeberg.org/forgejo-contrib/forgejo-cli`
  (pages: Home, Authentication, Issues, PRs, Repositories, Actions, Milestone,
  Organizations, Users). The wiki lags the tool and has a few wrong flag names, so
  prefer `--help` where they differ.
