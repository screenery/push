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
| `version` | pinned | The `screenery` npm version to run. |
| `api-url` | — | Control plane origin, for a non-default deployment. |
| `oidc-audience` | — | What the OIDC token is minted for. Derived from `api-url` when unset. |
| `force` | `false` | Promote even when the channel is ahead (spec §6). |

Every input travels through the step's `env` and none is interpolated into the
script. GitHub expands `${{ inputs.* }}` before bash parses the line, so an
input carrying shell syntax would run on the runner rather than be passed as an
argument — and a caller is free to map a pull-request-controlled value onto any
of these.

There is deliberately **no `token` input**. `SCREENERY_TOKEN` is still read
from the environment when a caller sets one — a self-hosted runner with no
OIDC, or a deployment with no GitHub App — but offering an input for it would
make the secret path look like the normal path, and it is not.

## Fork pull requests

A pull request from a fork gets no `id-token` permission, so there is no token
to exchange and nothing is published. That is GitHub working as intended: a
fork's workflow must not be able to publish to your project. The step prints
one line saying so, and the CLI prints the full explanation — including what to
do if you actually want fork previews (a `pull_request_target` workflow, which
runs your base branch's code, and is a decision rather than a workaround).

## Where this comes from

**This repository is a copy. Do not edit it here.** The source of truth is
[`packages/action/`](https://github.com/jfreal/screenery/tree/main/packages/action)
in `jfreal/screenery`; `action.yml` and this README are copied across and the
result is tagged. A composite action needs nothing built.

To cut a release: copy `action.yml` and the README over, check that the
`version` input still names the `screenery` npm release the tag corresponds
to, commit, and move the `v1` tag. Pinning `version` is what stops a CLI
release changing what an existing workflow does without that workflow
changing.
