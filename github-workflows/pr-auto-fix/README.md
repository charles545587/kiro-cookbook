# PR auto-fix with Kiro headless mode

A GitHub Actions workflow that closes the loop on automated code review. When one of your trusted review bots (AWS Security Agent, Amazon Q Developer, or any bot you add) leaves findings on a pull request, this workflow:

1. Collects every unresolved finding that doesn't already have a fixer-bot reply
2. Runs `kiro-cli` in headless mode with the findings as input
3. Commits the fixes to the PR branch
4. Posts a threaded reply on each original finding with a link to the fix commit
5. Posts a top-level summary comment on the PR

No manual intervention required.

## What's in this directory

- [`pr-trigger.yml`](pr-trigger.yml) — The full workflow file. Drop it into `.github/workflows/` in your repo.

## Prerequisites

1. A GitHub repository where you can push to the default branch and run Actions
2. A [Kiro account](https://kiro.dev) with an API key (generate one in your account settings)
3. Review bots already commenting on your PRs. The workflow is configured for AWS Security Agent and Amazon Q Developer by default — adjust the `user.login` allowlist to match whoever reviews your code

## Setup

### 1. Add the Kiro API key as a repository secret

```bash
gh secret set KIRO_API_SECRET --repo <owner>/<repo>
# Paste the API key when prompted
```

The secret is named `KIRO_API_SECRET` because GitHub blocks secrets with the `GITHUB_` prefix. The workflow maps it to `KIRO_API_KEY` (the env var the CLI expects) at runtime.

### 2. (Optional) Set up a GitHub App for workflow-file edits

The default `GITHUB_TOKEN` can't push commits that touch `.github/workflows/` files. If Kiro's fix needs to edit a workflow, the push will fail. To support this case, create a GitHub App with `contents: write`, `pull-requests: write`, and `workflows: write` permissions, install it on your repo, and add two more secrets:

```bash
gh secret set KIRO_FIXER_APP_ID --repo <owner>/<repo>
gh secret set KIRO_FIXER_PRIVATE_KEY --repo <owner>/<repo>
```

The workflow detects these automatically and falls back to `GITHUB_TOKEN` if they're not set.

### 3. Drop in the workflow

Copy [`pr-trigger.yml`](pr-trigger.yml) into `.github/workflows/` in your repo, commit it to the default branch, and open a PR to test.

GitHub only fires comment/review event workflows from the default branch, so the workflow must be merged to `main` before it can respond to bot comments.

## How it works

### Events that fire the workflow

- `issue_comment` — PR conversation comments
- `pull_request_review` — Review summaries at the top of a review
- `pull_request_review_comment` — Inline per-line comments
- `workflow_dispatch` — Manual trigger for sweeping a specific PR

### Trust boundary

The `if:` block restricts runs to comments/reviews authored by the trusted bots. Swap the bot logins to match your own reviewers (Snyk, CodeQL, Dependabot, etc.).

### Sweep-every-time model

On every run, the workflow queries the PR's review threads via the GraphQL API and filters to:
1. Threads that are not resolved
2. Authored by a trusted bot
3. Don't already have a fixer-bot reply

This means you don't have to figure out which specific comment triggered the run — the workflow always processes whatever's outstanding. Late bot comments, manual dispatches, and re-reviews all converge on the same outcome.

### Kiro prompt

Kiro receives two files on disk:
- `reviewer-comment.txt` — The triggering comment/review
- `findings.json` — Every outstanding finding on the PR

And writes two files back:
- `kiro-replies.json` — Per-finding status and explanation
- `output-summary.md` — Human-readable summary for the top-level comment

## Safety considerations

This workflow runs `kiro-cli` with `--trust-all-tools` and then `git push`. That's significant autonomy. On a personal or low-stakes repo it's fine. On a production codebase you'd want:

- An allowlist of paths Kiro can touch
- A secret scanner on the staged diff before commit
- An approval gate before the push
- A second job that runs tests after the fix commit

The trusted-bots allowlist prevents arbitrary external commenters from triggering the workflow, but if your trusted bots ever get compromised, the workflow would execute whatever they comment. Treat the bot allowlist with the same care you'd treat any other trust boundary.

## Trigger it manually

```bash
gh workflow run "Kiro PR Comment Auto-Fix" \
  --repo <owner>/<repo> \
  -f pr_number=42
```

Useful for sweeping findings that predate the workflow or testing the flow without waiting for a fresh bot review.

## Customization pointers

- **Change the trusted bots**: update the `github.event.*.user.login` checks in the `if:` blocks
- **Scope Kiro's edits**: modify the prompt in the "Run Kiro in headless mode" step to restrict paths or tool usage
- **Resolve threads after replying**: add a GraphQL `resolveReviewThread` mutation after the reply step
- **Re-trigger CI on the fix commit**: swap the `GITHUB_TOKEN` push for a PAT or GitHub App token so `pull_request.synchronize` workflows fire

## Related recipes

- **[Spec-driven development](../spec-driven/)** — Pairs naturally with this recipe. When you label a GitHub issue `kiro-spec`, Kiro drafts a spec PR. Label it `kiro-implement` and Kiro opens an implementation PR. That PR is reviewed by your bots, and this workflow (pr-auto-fix) resolves every finding they leave. End-to-end: label → merged feature, with you only in the loop for approvals.

## Companion blog posts

- [Part 1: How I automated fixing GitHub bot review comments using Kiro and GitHub Actions](https://www.linkedin.com/pulse/how-i-automated-fixing-github-bot-review-comments-using-roberts-t5jqe/) — the write-up of this recipe.
- [Part 2: From GitHub issue to merged PR, without opening my IDE](https://www.linkedin.com/pulse/from-github-issue-merged-pr-without-opening-my-ide-charles-roberts-9qjme/) — the write-up of the spec-driven recipe.

## License

Apache 2.0. See the [LICENSE](../../LICENSE) at the root of the repository.
