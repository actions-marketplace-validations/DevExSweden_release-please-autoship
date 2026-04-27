## Release Please Autoship (Composite Action)

Run [Release Please](https://github.com/googleapis/release-please) to create or update a release PR, then automatically approve and merge it using a GitHub App token.

### What this action does

1. Generates a short-lived GitHub App token from the provided App ID and private key.
2. Runs `googleapis/release-please-action` against the target branch to create or update a release PR (and optionally a GitHub Release).
3. Parses the PR number from the Release Please JSON output.
4. Calls the internal `approve-and-merge-pr` action to approve and enable auto-merge on the release PR.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `target-branch` | **yes** | — | Branch that Release Please targets (e.g. `main`, `develop`). |
| `config-file` | no | `.github/release-please-config.json` | Path to the Release Please config file. |
| `manifest-file` | no | `.github/.release-please-manifest.json` | Path to the Release Please manifest file. |
| `skip-github-release` | no | `true` | When `true`, only the release PR is created; no GitHub Release is published. |
| `github-app-id` | **yes** | — | GitHub App ID used to create the authentication token. |
| `github-app-private-key` | **yes** | — | Private key of the GitHub App (PEM format). |

### Outputs

| Name | Description |
|------|-------------|
| `prs_created` | `"true"` if Release Please created or updated a PR, `"false"` otherwise. |
| `pr` | Full PR object returned by Release Please as a JSON string. |
| `pr_number` | The number of the release PR (populated only when `prs_created == 'true'`). |
| `release_created` | `"true"` if a GitHub Release was published (only relevant when `skip-github-release` is `false`). |

### Usage

Store your GitHub App credentials as repository (or organisation) secrets, then reference this action in a workflow step:

```yaml
name: Release Please

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Run Release Please and auto-merge
        uses: DevExSweden/release-please-autoship@main
        with:
          target-branch: main
          github-app-id: ${{ secrets.RELEASE_APP_ID }}
          github-app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

#### Using a custom config or manifest path

```yaml
      - name: Run Release Please and auto-merge
        uses: DevExSweden/release-please-autoship@main
        with:
          target-branch: main
          config-file: .github/release-please-config.json
          manifest-file: .github/.release-please-manifest.json
          github-app-id: ${{ secrets.RELEASE_APP_ID }}
          github-app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

#### Also publish a GitHub Release

```yaml
      - name: Run Release Please and auto-merge
        uses: DevExSweden/release-please-autoship@main
        with:
          target-branch: main
          skip-github-release: "false"
          github-app-id: ${{ secrets.RELEASE_APP_ID }}
          github-app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

#### Consuming outputs in subsequent steps

```yaml
      - name: Run Release Please and auto-merge
        id: rp
        uses: DevExSweden/release-please-autoship@main
        with:
          target-branch: main
          github-app-id: ${{ secrets.RELEASE_APP_ID }}
          github-app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}

      - name: Do something after release PR is created
        if: ${{ steps.rp.outputs.prs_created == 'true' }}
        run: echo "Release PR #${{ steps.rp.outputs.pr_number }} was created or updated."
```

### How it works

```
push to target-branch
        │
        ▼
actions/create-github-app-token   ← github-app-id + github-app-private-key
        │
        ▼
googleapis/release-please-action  ← creates/updates release PR
        │
        ├─ prs_created == 'false'  → done (no PR to merge)
        │
        └─ prs_created == 'true'
                │
                ▼
         Extract PR number (jq)
                │
                ▼
         approve-and-merge-pr             ← GITHUB_TOKEN approves, App token enables auto-merge
```

The action uses a **dual-identity pattern** for auto-merge:

- The **GitHub App token** creates the release PR and enables auto-merge (so the merge appears as the App's identity, satisfying branch-protection rules that require a non-author review).
- The built-in **`GITHUB_TOKEN`** approves the PR (a separate identity from the one that opened it).

### Requirements and permissions

**Job-level permissions** (add to the calling workflow job):

```yaml
permissions:
  contents: write       # required by release-please to push commits / tags
  pull-requests: write  # required to create / approve / merge PRs
```

**GitHub App** must have the following repository permissions:

| Permission | Access |
|------------|--------|
| `Contents` | Read & write |
| `Pull requests` | Read & write |
| `Metadata` | Read |

**Branch protection** on the target branch should require at least one approving review and allow the GitHub App as a bypass actor (or as an allowed merge actor) so that `gh pr merge --auto` can complete once the review is added.

**Runtime dependencies**: the action uses `jq` (pre-installed on GitHub-hosted runners) to parse the Release Please JSON output.

### Troubleshooting

- **No PR created**: Release Please only opens a PR when it detects [Conventional Commits](https://www.conventionalcommits.org/) since the last release. Verify commits on the target branch follow the convention.
- **Auto-merge not enabled / falling back to direct merge**: Branch protection is not configured on the target branch. The action will fall back to a direct merge automatically.
- **`gh pr review` fails**: The `GITHUB_TOKEN` in the calling workflow must have `pull-requests: write` permission. Check the `permissions` block in your workflow.
- **App token creation fails**: Confirm `github-app-id` and `github-app-private-key` are correct and the App is installed on the repository.
- **`jq` not found**: Use a GitHub-hosted runner (Ubuntu, macOS, Windows) — all include `jq` by default.
