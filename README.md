# integrations-shared-workflows

Reusable GitHub Actions workflows shared across ClickHouse integrations repos
(language clients, connectors, …). Source repos call these via `workflow_call`
so the logic lives in one place and per-repo specifics are passed as inputs.

## Workflows

### `remote-docs-preview.yml` - Request a remote documentation preview

Starts a scoped documentation preview for a pull request in a repository
registered by `ClickHouse/mintlify-docs-dev/remotes.json`. A maintainer invokes
the caller by adding the `docs-preview` label to a pull request targeting the
remote repository's default branch. The shared workflow validates the trusted
label event, pins the pull request's exact head SHA, and directly
creates a Vercel deployment of the current trusted Nimbus `main` revision in the
`connect-preview` Custom Environment.

The workflow never checks out or executes pull-request content and never passes
a GitHub credential to Vercel. Inside the deployment, Vercel Connect exchanges
the deployment's OIDC identity for a short-lived `contents:read` token scoped to
the selected repository. Nimbus fetches the approved revision before it removes
the token and begins processing Markdown or MDX. The shared workflow waits for
the deployment and comments the resulting URL on the source pull request.

| Input | Required | Purpose |
|---|---|---|
| `remote_name` | yes | Source name in the central `remotes.json` registry. |
| `pull_request_number` | yes | Open pull request whose exact head SHA should be previewed. |

Required secrets are `VERCEL_TOKEN`, `VERCEL_ORG_ID`, and
`VERCEL_PROJECT_ID`. Define them once as organization Actions secrets and grant
them to the registered source repositories; callers can then pass the three
secrets explicitly without duplicating their values per repository. The Vercel
project must provide `DOCS_GITHUB_CONNECTOR` in its `connect-preview` Custom
Environment and allow that environment to use the Vercel Connect GitHub app.

See [`examples/caller-remote-docs-preview.yml`](examples/caller-remote-docs-preview.yml)
for a copy-paste `pull_request_target` caller. Set `remote_name` to the source's
registered name and configure the caller's native `paths` filter for the files
that should be eligible, such as `docs/**`. Remove `paths` when every pull
request should be eligible. GitHub starts the reusable workflow only when the
pull request changes a configured path and a maintainer adds `docs-preview`.
The caller deliberately listens only for `labeled` events and the shared
workflow independently verifies the label, action, pull request number, and
default target branch.

### `claude-docs-drift.yml` - Dispatch centralized docs drift checks

Sends a source pull request to the central docs-drift worker in
`ClickHouse/integrations-ai-playground`. The shared workflow is a thin relay. It
uses the Workflow Authentication GitHub App to validate the source PR and send
a `repository_dispatch` event containing the source repo, PR number, expected
head SHA, and rubric options. It does not run Claude or receive a model
credential.

The central worker reads the source repo's rubric from the trusted base
revision, reviews the PR diff, and reconciles the `needs-docs` label and sticky
comment back on the source PR. It checks the expected head SHA before and after
the model run so stale results cannot overwrite a newer push.

| Input | Required | Default | Purpose |
|---|---|---|---|
| `agent_path` | no | `.claude/agents/docs-drift-reviewer.md` | Source repo rubric path. A missing rubric makes the central check a no-op. |
| `label` | no | `needs-docs` | Drift label. Empty makes the check comment-only. |
| `model` | no | central default | Optional Claude model override. |
| `max_turns` | no | `30` | Maximum turns in the central worker. |
| `pr_number` | no | from event | Required only for `workflow_dispatch` callers. |
| `central_repo` | no | `ClickHouse/integrations-ai-playground` | Repository that receives the dispatch. |

Required secrets are `WORKFLOW_AUTH_PUBLIC_APP_ID` and
`WORKFLOW_AUTH_PUBLIC_PRIVATE_KEY`. These are the same GitHub App credentials
used by `cross-repo-bug-relay.yml`. The caller grants its `GITHUB_TOKEN` only
`pull-requests: read`; the App token is scoped to the central repository for
dispatch. Source repos do not need an Anthropic key.

See [`examples/caller-claude-docs-drift.yml`](examples/caller-claude-docs-drift.yml)
for a copy-paste caller. Add repository-specific `paths` filters when only part
of a repository can contain user-visible changes.

### `claude-pr-triage.yml` - Triage PRs with Claude

Classifies each PR (category + low/medium/high risk) against a rubric the caller
supplies, then applies `triage:*` / `risk:*` labels and upserts a single sticky
comment. Claude is read-only on PR state; a follow-up workflow step performs all
label/comment writes from Claude's validated JSON output, so a prompt injection
in a PR cannot mislabel or post a tampered comment.

For the judgement criteria, each caller passes its own rubric via
the `triage_instructions` input, so the same workflow serves repos with very
different risk surfaces.

| Input | Required | Default | Purpose |
|---|---|---|---|
| `triage_instructions` | yes | — | The repo's rubric only: category meanings, High/Medium/Low risk rules, optional reviewer-action policy. The method, Concerns guidance, schema, and comment format come from the skeleton. |
| `categories` | no | `bugfix,feature,refactor,perf,deps,docs,tests,infra` | Allowed `category` values; drives both the JSON-schema enum and validation. |
| `model` | no | action default | Claude model override (e.g. `claude-opus-4-8`). |
| `max_turns` | no | `15` | Max agent turns. |
| `pr_number` | no | from event | PR to triage; forward your `workflow_dispatch` input here. `pull_request` callers can omit it. |

**Secret:** `ANTHROPIC_API_KEY` (required), passed via `secrets: inherit`.

This workflow processes untrusted PR content, so it's hardened against secret
exfiltration: the Claude step has no network tools and no arbitrary Bash (only
read-only `gh` reads), can't write files or post comments, and never executes
the PR's code. The workflow references only `ANTHROPIC_API_KEY` and the
`permissions`-scoped `GITHUB_TOKEN`, and GitHub injects a secret into a step only
where it's explicitly referenced — so `secrets: inherit` doesn't expose anything
else, and a per-job `environment` would add no isolation here. Rotate the key if
you ever suspect compromise.

**Quick Start:**

 * Add `ANTHROPIC_API_KEY` secret.
 * Add `triage:` and `risk:` labels.
 * Copy paste caller example and provide the category and risk descriptions for your repo.

See [`examples/caller-claude-pr-triage.yml`](examples/caller-claude-pr-triage.yml)
for a copy-paste caller.

See [here](https://github.com/ClickHouse/clickhouse-cs/blob/main/.github/workflows/claude-pr-triage.yml) for the live workflow in the .NET repo.

### `cross-repo-bug-relay.yml` - Relay issues/PRs to a central repo

Called from source repos on `issues` / `pull_request` events to copy the item
into a central cross-repo-investigation repo (issues labelled `relayed`, PRs
`relayed-pr`). See the header of the workflow file for usage; `central_repo` is
configurable and defaults to the language-client pipeline.

## Conventions

- Pin reusable-workflow refs to `@main` (or a tag/SHA) in callers.
