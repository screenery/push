# `screenery/push@v1`

The four lines from spec §17:

```yaml
- uses: screenery/push@v1
  with: { project: acme/dashboard, path: test-results }
```

with one line above them in the job:

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: write
```

**No token, and no secret to rotate.** The Action runs `npx screenery push`,
which asks the runner for a GitHub OIDC token and exchanges it at
`POST /v1/oidc/exchange` for a publish token that lives five minutes. The
exchange checks the repository's **numeric id** against the project's binding
(spec §5.1), so a repository rename does not break publishing and a fork
cannot pass.

## Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `project` | — | `{org}/{project}`. Required. One repository may feed several projects; this is what routes the build. |
| `path` | `test-results` | Directory of screenshots, walked recursively. |
| `version` | `0.2.0` | The `screenery` npm version to run. |
| `api-url` | — | Control plane origin, for a non-default deployment. |
| `oidc-audience` | — | What the OIDC token is minted for. Derived from `api-url` when unset. |
| `force` | `false` | Promote even when the channel is ahead (spec §6). |
| `diff-baseline` | — | Optional committed baseline directory. When set, compare current screenshots (and `findings.json` sidecars) against it **before** push; a regression fails the job and nothing is published. |
| `diff-threshold` | `0.1` | Pixelmatch threshold for `diff-baseline`, 0..1. |
| `comment` | `auto` | Post or update a PR comment from this build. `auto` posts on `pull_request`. |
| `gallery-out` | — | Write gallery Markdown to this path. |
| `readme-out` | — | Write paste-once README embed Markdown to this path. |
| `comment-out` | — | Write the PR comment Markdown to this path. |

`version` is pinned to `0.2.0`, the first CLI release with `diff` and the PR
comment publisher, so every input above works at the default. Override it only
to run a different CLI release on purpose.

Every input travels through the step's `env` and none is interpolated into the
script. GitHub expands `${{ inputs.* }}` before bash parses the line, so an
input carrying shell syntax would run on the runner rather than be passed as an
argument — and a caller is free to map a pull-request-controlled value onto any
of these.

There is deliberately **no `token` input**. `SCREENERY_TOKEN` is still read
from the environment when a caller sets one — a self-hosted runner with no
OIDC, or a deployment with no GitHub App — but offering an input for it would
make the secret path look like the normal path, and it is not.

`pull-requests: write` is what lets the step post (or update) a preview
comment. Without it the push still succeeds; the log warns and the gallery
Markdown is still written.

## After the push: PR comment, gallery, README snippets

The Action does not capture screenshots — Playwright (or anything that writes
PNG/GIF/WebM/MP4 under `path`) already did. After finalize it publishes from
those Screenery assets:

1. **PR comment** on `pull_request` (upserted, one per project, marked
   `<!-- screenery-preview:{org}/{project} -->`). A 4-up still strip, a
   Walkthrough section for GIF/WebM/MP4, PR-channel embed snippets, and
   paste-once `@latest` README snippets.
2. **Gallery Markdown** and README snippets as Action outputs and under
   `$RUNNER_TEMP/screenery/`. The declared outputs are `channel`,
   `comment-url`, `gallery-file`, `readme-file`, `comment-file`,
   `gallery-markdown`, `readme-markdown` and `pr-comment-markdown` —
   read them as `${{ steps.<id>.outputs.gallery-markdown }}`.
3. The run summary gets the same compact comment body.

GIF and video in the push become the Walkthrough section. They are ordinary
Screenery assets on the same channel — not a second capture product and not a
separate `walkthrough` channel (a channel is a whole-build pointer).

Only `public` assets are embedded. An `unlisted` asset needs a signature and a
`private` one needs a read token (spec §3), so a bare URL for either would
404; the comment names how many it withheld instead of publishing broken
images. The comment is also capped to stay inside GitHub's issue-comment body
limit — the gallery Markdown keeps the full set.

Turn the comment off with `comment: false`. Regenerate Markdown later without
re-uploading:

```bash
npx screenery publish --from "$RUNNER_TEMP/screenery/publish.json" --format gallery
```

## Fork pull requests

A pull request from a fork gets no `id-token` permission, so there is no token
to exchange and nothing is published. That is GitHub working as intended: a
fork's workflow must not be able to publish to your project. The step prints
one line saying so, and the CLI prints the full explanation — including what to
do if you actually want fork previews (a `pull_request_target` workflow, which
runs your base branch's code, and is a decision rather than a workaround).

## Publishing this action

This directory is the source. `screenery/push@v1` is a separate repository, and
the release step is to copy `action.yml` and this README there and tag it — a
composite action needs nothing built. Pin `version` to the `screenery` npm
release the tag corresponds to, so a CLI release cannot change what an existing
workflow does.
