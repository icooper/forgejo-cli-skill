# When fj is not enough: the Forgejo REST API

`fj` has no `api` subcommand and no JSON output. For anything it cannot do, or when you
need machine-readable data, call the instance's REST API with `curl`. Try `fj` and `git`
first - they are already authenticated and need no token handling.

## Contents

- [Basics](#basics)
- [Authentication](#authentication)
- [Setup snippet](#setup-snippet)
- [Recipes: reading](#recipes-reading)
- [Recipes: writing](#recipes-writing)
- [Finding other endpoints](#finding-other-endpoints)

## Basics

- Base URL: `https://HOST/api/v1`
- Responses are JSON. Pipe through `jq` (or `python3 -m json.tool`) to read them.
- Lists are paginated with `?page=N&limit=M`, and the order is the server's, not
  alphabetical. The total is in the `X-Total-Count` response header; see it with
  `curl -sS -D - -o /dev/null URL` (a `HEAD` request via `curl -I` is rejected with
  `405`). The server caps `limit`, commonly at 50.
- Forgejo's API is a superset of the Gitea API, so Gitea API knowledge mostly transfers.
  GitHub API knowledge only partly does: paths look similar, but check before assuming.
- Endpoints vary with the instance's Forgejo version. A `404` on an Actions endpoint
  usually means the instance is older than the feature.
- A private repo requested without a (sufficient) token answers `404`, not `401`. If a
  repo you know exists is "not found", suspect authentication first.
- Public instances rate-limit anonymous callers (`429`), search endpoints especially.
  Back off rather than retrying in a loop.

## Authentication

Public data needs no authentication. For private data or writes, send a token:

```
Authorization: token <TOKEN>
```

Where the token comes from:

1. An environment variable the user has set for this purpose (commonly `FORGEJO_TOKEN`).
   Check for it without printing it: `[ -n "$FORGEJO_TOKEN" ] && echo set`.
2. Otherwise ask the user to create an application token at
   `https://HOST/user/settings/applications` and export it. Suggest the narrowest scopes
   that cover the task.

`fj` keeps its own logins in a credentials file in its data directory. Treat that file
as off limits unless the user explicitly tells you to use it: it holds tokens for every
host they are logged in to. Never echo a token, put it in a URL, or write it to a file
in the repo.

## Setup snippet

Derive host and repo from the checkout instead of hard-coding them. The first line of
`fj repo view` is `owner/name` and the last is the web URL. Outside a checkout, set
`REPO=owner/name` and `HOST=the.host` by hand.

```bash
REPO=$(fj repo view | head -1)                        # owner/name
HOST=$(fj repo view | sed -n 's|^View online at https\{0,1\}://\([^/]*\)/.*|\1|p')
API="https://$HOST/api/v1"

fjapi() {  # usage: fjapi GET /route  |  fjapi POST /route '{"json":"body"}'
  local method=$1 route=$2 data=${3:-}
  set -- -sS --fail-with-body -X "$method" -H "Accept: application/json"
  [ -n "${FORGEJO_TOKEN:-}" ] && set -- "$@" -H "Authorization: token $FORGEJO_TOKEN"
  [ -n "$data" ] && set -- "$@" -H "Content-Type: application/json" -d "$data"
  curl "$@" "$API$route"
}
```

The function sends the token only when `FORGEJO_TOKEN` is set, so the same calls work
anonymously against public repos. It is written to behave the same in bash and zsh
(in zsh, never name a variable `path` - it is tied to `PATH`).

## Recipes: reading

```bash
# Issue or PR as JSON (state, labels, assignees, timestamps, html_url)
fjapi GET "/repos/$REPO/issues/12" | jq '{number,title,state,labels:[.labels[].name]}'
fjapi GET "/repos/$REPO/pulls/34"  | jq '{number,title,state,merged,mergeable,head:.head.ref,base:.base.ref}'

# List issues / PRs with fields fj does not print
fjapi GET "/repos/$REPO/issues?state=open&type=issues&limit=50" | jq -r '.[] | "\(.number)\t\(.updated_at)\t\(.title)"'
fjapi GET "/repos/$REPO/pulls?state=open&limit=50"              | jq -r '.[] | "\(.number)\t\(.user.login)\t\(.title)"'

# Search issues across every repo you can see (needs a token)
fjapi GET "/repos/issues/search?q=timeout&type=issues&state=open" | jq -r '.[] | "\(.repository.full_name)#\(.number) \(.title)"'

# Search repositories on the instance
fjapi GET "/repos/search?q=runner&limit=20" | jq -r '.data[].full_name'

# Branches, commits, files (prefer git when you have a checkout)
fjapi GET "/repos/$REPO/branches" | jq -r '.[].name'
fjapi GET "/repos/$REPO/commits?sha=main&limit=10&stat=false" | jq -r '.[] | "\(.sha[0:8]) \(.commit.message | split("\n")[0])"'
fjapi GET "/repos/$REPO/git/trees/main?recursive=true" | jq -r '.tree[].path'
fjapi GET "/repos/$REPO/raw/README.md?ref=main"                 # raw file contents, not JSON

# Compare two refs
fjapi GET "/repos/$REPO/compare/main...my-branch" | jq '{total_commits, files:[.files[]?.filename]}'

# PR reviews, and who has been asked to review but has not yet (fj shows only the former)
fjapi GET "/repos/$REPO/pulls/34/reviews" | jq -r '.[] | "\(.user.login)\t\(.state)\t\(.body)"'
fjapi GET "/repos/$REPO/pulls/34" | jq -r '.requested_reviewers[]?.login'

# CI: combined status for a commit or branch, then runs, jobs and logs
fjapi GET "/repos/$REPO/commits/main/status" | jq '{state, checks:[.statuses[]? | {context,status,target_url}]}'
fjapi GET "/repos/$REPO/actions/runs?limit=10"
fjapi GET "/repos/$REPO/actions/runs/RUN_ID/jobs"
fjapi GET "/repos/$REPO/actions/jobs/JOB_ID/logs"               # plain text

# Notifications for the logged-in user
fjapi GET "/notifications?limit=20" | jq -r '.[] | "\(.subject.type)\t\(.repository.full_name)\t\(.subject.title)"'

# Who am I (token check)
fjapi GET "/user" | jq -r .login
```

The shape of the Actions responses differs between Forgejo versions; look at the raw JSON
once before writing a `jq` filter for it.

## Recipes: writing

These change things other people see. Run them when the user asked for that outcome.

```bash
# Reopen a closed issue or PR (fj can close but not reopen; a merged PR cannot be reopened)
fjapi PATCH "/repos/$REPO/issues/12" '{"state":"open"}'

# Submit a PR review. event: APPROVED | REQUEST_CHANGES | COMMENT
fjapi POST "/repos/$REPO/pulls/34/reviews" '{"event":"APPROVED","body":"Looks good"}'

# Review with an inline comment
fjapi POST "/repos/$REPO/pulls/34/reviews" \
  '{"event":"COMMENT","body":"One note","comments":[{"path":"src/app.rs","new_position":42,"body":"Off by one?"}]}'

# Re-run a finished workflow run
fjapi POST "/repos/$REPO/actions/runs/RUN_ID/rerun"
```

Build JSON bodies with `jq -n --arg body "$TEXT" '{body:$body}'` when the text contains
quotes or newlines, rather than interpolating into a string.

## Finding other endpoints

Every instance publishes its own API description, which matches its exact version:

- Browsable: `https://HOST/api/swagger`
- Machine-readable: `https://HOST/swagger.v1.json`

```bash
# List paths that mention a keyword
curl -sS "https://$HOST/swagger.v1.json" | jq -r '.paths | keys[]' | grep -i protect

# Show methods and parameters for one path
curl -sS "https://$HOST/swagger.v1.json" \
  | jq '.paths["/repos/{owner}/{repo}/branches"] | map_values({summary, params:[.parameters[]?.name]})'
```
