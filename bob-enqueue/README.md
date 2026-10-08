# bob-enqueue

Composite GitHub Action that enqueues a Bob AI code review, PR summary, or
issue triage request from any github.com repository.

Writes a JSON job descriptor to
[`Terracotta-OSS/pr-review-queue`](https://github.com/Terracotta-OSS/pr-review-queue)
(or a custom queue repo you control). An in-house CI poller picks the job up
and runs Bob Shell against the target PR.

## Usage

Add this file to your repo as `.github/workflows/bob.yml`:

```yaml
name: 'Bob'

on:
  issue_comment:
    types: [created]

jobs:
  enqueue:
    name: 'Enqueue Bob job'
    if: |-
      github.event.sender.type == 'User' &&
      contains(github.event.comment.body, '@bobshell') &&
      contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.comment.author_association)
    runs-on: ubuntu-latest
    permissions:
      contents: read
      issues: write
      pull-requests: write
    steps:
      - uses: 'Terracotta-OSS/terracotta-actions/bob-enqueue@main'
        with:
          queue_token: '${{ secrets.BOB_QUEUE_TOKEN }}'
          comment:     '${{ github.event.comment.body }}'
```

Then comment on any PR or issue:

| Comment | Action |
|---|---|
| `@bobshell /review` | Full AI code review posted as a GitHub review |
| `@bobshell /summary` | PR description generated and posted as a comment |
| `@bobshell /triage` | Issue analysed and labelled |

## Inputs

| Input | Required | Default | Description |
|---|:---:|---|---|
| `queue_token` | ✅ | | Fine-grained PAT with `contents: write` on `queue_repo` |
| `comment` | ✅ | | The `issue_comment.body` that triggered the workflow |
| `queue_repo` | | `Terracotta-OSS/pr-review-queue` | Override if you run your own queue |
| `max_cost_review` | | `5` | Bobcoin budget for `/review` |
| `max_cost_summary` | | `2` | Bobcoin budget for `/summary` |
| `max_cost_triage` | | `2` | Bobcoin budget for `/triage` |

## Outputs

| Output | Description |
|---|---|
| `job_file` | Path of the committed job descriptor, e.g. `jobs/20260101-120000-<uuid>.json` |
| `command` | Resolved command: `review`, `summary`, or `triage` |

## Setting up `BOB_QUEUE_TOKEN`

Create a **fine-grained PAT** scoped to `Terracotta-OSS/pr-review-queue` only:

- **Resource owner**: `Terracotta-OSS`
- **Repository access**: `Terracotta-OSS/pr-review-queue` only
- **Permissions**: `Contents: Read and write`

Store it as a secret named `BOB_QUEUE_TOKEN` in your repo (or org).

## Job descriptor schema

Each file written to `queue_repo/jobs/` is a JSON object:

```json
{
  "command":            "review",
  "target_org":         "acme-corp",
  "target_repo":        "my-service",
  "pr_number":          42,
  "actor":              "alice",
  "issue_title":        "Fix the thing",
  "target_github_host": "https://api.github.com",
  "max_cost_review":    5,
  "max_cost_summary":   2,
  "max_cost_triage":    2,
  "enqueued_at":        "2026-01-01T12:00:00Z",
  "source_run_url":     "https://github.com/acme-corp/my-service/actions/runs/12345"
}
```

The CI poller reads these fields and passes them as `workflow_dispatch` inputs
to `ibm-webmethods/terracotta-bob-action/.github/workflows/dispatch.yml`.
