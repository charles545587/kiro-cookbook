# Spec-driven development with Kiro headless mode

Two GitHub Actions workflows that turn a labelled GitHub issue into a merged feature, without opening an IDE:

1. **`issue-to-spec.yml`** — when you label an issue `kiro-spec`, Kiro reads the issue and the full comment thread, then either:
   - asks clarifying questions as a comment (and re-triggers on your reply),
   - commits a draft spec to `.kiro/specs/<feature-name>/` and opens a PR against `main`,
   - or politely declines if the issue isn't something a spec would help with.
2. **`spec-to-implementation.yml`** — when you label an issue `kiro-implement` (or comment `/kiro implement`), Kiro reads the spec, works through `tasks.md`, commits one commit per task, and opens an implementation PR.

Used with the [PR auto-fix workflow](../pr-auto-fix/) from the same cookbook, the full chain is:

```
Issue labelled kiro-spec
  │
  ▼  issue-to-spec.yml
Spec PR
  │
  ▼  human review + merge
Spec on main
  │
  ▼  Issue labelled kiro-implement
Implementation PR
  │
  ▼  Bots review the PR
  ▼  pr-auto-fix workflow resolves every finding
Final human review + merge
  │
  ▼  Issue auto-closes via Closes #N
```

Every handoff is a PR, a label, or a comment on an issue. No shared state, no workflow-to-workflow secrets.

## What's in this directory

- [`issue-to-spec.yml`](issue-to-spec.yml) — issue → spec PR
- [`spec-to-implementation.yml`](spec-to-implementation.yml) — spec → implementation PR

Drop both into `.github/workflows/` in your repo.

## Prerequisites

1. A GitHub repository where you can push to the default branch and run Actions.
2. A [Kiro account](https://kiro.dev) with an API key (generate one in your account settings).
3. The [PR auto-fix workflow](../pr-auto-fix/) from this cookbook, installed in the same repo. Strictly optional but strongly recommended — without it, bot review findings on the implementation PR won't auto-resolve.
4. At least one existing spec under `.kiro/specs/<feature-name>/` in your repo, so Kiro has a house style to copy. Fifteen minutes of hand-writing one spec up front pays back for every auto-generated spec after it.
5. (Optional but high leverage) At least one steering file under `.kiro/steering/*.md` capturing project conventions. Kiro reads these before drafting specs or implementing tasks and the quality difference is large.

## Setup

### 1. Add the Kiro API key as a repository secret

```bash
gh secret set KIRO_API_SECRET --repo <owner>/<repo>
# Paste the API key when prompted
```

### 2. (Optional) Set up a GitHub App for workflow-file edits

If you expect Kiro to ever produce specs or implementations that touch `.github/workflows/` files, the default `GITHUB_TOKEN` can't push those changes — you need a GitHub App token. Create a GitHub App with `contents: write`, `pull-requests: write`, `issues: write`, and `workflows: write` permissions, install it on your repo, and add two more secrets:

```bash
gh secret set KIRO_FIXER_APP_ID    --repo <owner>/<repo>
gh secret set KIRO_FIXER_PRIVATE_KEY --repo <owner>/<repo>
```

The workflows detect the app credentials automatically and fall back to `GITHUB_TOKEN` when they're not set.

### 3. Create the labels

Both workflows auto-create their trigger labels on first run, but you can create them up front if you prefer:

```bash
gh label create kiro-spec      --color 1f6feb --description "Ask Kiro to draft a spec for this issue" --repo <owner>/<repo>
gh label create kiro-implement --color a371f7 --description "Ask Kiro to implement the spec for this issue" --repo <owner>/<repo>
```

### 4. Commit the workflows to your default branch

GitHub only recognises workflows on the default branch for `issues` and `issue_comment` events, so the workflow files must be merged to `main` (or whatever your default branch is) before they can fire.

```bash
git checkout -b chore/add-kiro-spec-driven
mkdir -p .github/workflows
cp /path/to/cookbook/github-workflows/spec-driven/*.yml .github/workflows/
git add .github/workflows/
git commit -m "ci: Add Kiro spec-driven workflows"
git push -u origin chore/add-kiro-spec-driven
gh pr create --fill
gh pr merge --squash
```

## How they work

### `issue-to-spec.yml`

Triggers on:
- `issues.labeled` with label `kiro-spec` — primary path
- `issue_comment.created` on an issue that already has `kiro-spec`, from a repo collaborator — for multi-turn conversation
- `workflow_dispatch` — for manual retries

The workflow builds a `thread.md` file containing the issue title, body, and every comment in order (each tagged with its author login), then passes it to Kiro with a prompt that requires exactly one of three outcomes:

| Outcome | What happens |
|---|---|
| `ask` | Kiro posts clarifying questions as a comment on the issue, then the job ends. Your reply re-triggers the workflow. |
| `spec` | Kiro writes `requirements.md` (or `bugfix.md`), `design.md`, and `tasks.md` under `.kiro/specs/<feature-name>/`, commits to a new branch `spec/<issue-number>-<feature-name>`, and opens a PR. |
| `decline` | Kiro posts a polite explanation of why a spec won't help. |

All three outcomes are written as a single `kiro-outcome.json` file with a known schema, so the workflow can branch cleanly on the outcome without parsing free-form text.

### `spec-to-implementation.yml`

Triggers on:
- `issues.labeled` with label `kiro-implement`
- `issue_comment.created` with body starting `/kiro implement`, from a repo collaborator
- `workflow_dispatch`

The workflow first tries to find the spec by parsing the marker comment left by `issue-to-spec.yml` (a comment containing the phrase "Drafted a spec" and a `.kiro/specs/<feature-name>/` path reference). If no marker is found, Kiro is asked to match the issue to an existing spec at run time.

It then:
1. Branches off `main` as `impl/<issue-number>-<feature-name>`.
2. Runs Kiro with a prompt that requires:
   - Work through `tasks.md` in order
   - Commit after each task with a message `feat(<feature>): <task title> (#<issue-number>)` or `fix(...)`
   - Tick the corresponding checkbox in `tasks.md` as part of the same commit
   - Never modify the spec's `requirements.md`/`design.md`/`bugfix.md` files
3. Sweeps up any uncommitted stragglers into a single `chore(...)` commit.
4. Force-with-leases the branch and opens a PR titled `feat: <feature-name> (#<issue-number>)` or `fix: ...`.
5. Posts a comment on the original issue linking to the PR.

The resulting PR body lists completed and skipped tasks and includes `Closes #<issue-number>` so merging auto-closes the issue.

## Safety model

Both workflows run with `contents: write`, `issues: write`, `pull-requests: write`. They execute an AI agent (`kiro-cli chat --trust-all-tools`). Three gates protect against misuse:

1. **Label gate.** Only issues carrying the relevant label trigger the workflow. Removing the label stops further triggers.
2. **Author-association gate.** Comment triggers require the commenter to be a repo `OWNER`, `MEMBER`, or `COLLABORATOR`. External users who can see the labelled issue can't drive the workflow.
3. **PR gate.** Every change Kiro makes lands on a branch and reaches `main` only through a PR. **Require PR reviews on your default branch.** This is non-negotiable for this pattern.

The `announce` job that posts the "Kiro is on it" heads-up comment is deliberately split into a separate job *outside* the concurrency group. Without this split, the `issue_comment` webhook generated by the announce comment would re-enter the concurrency group and cancel the very run that posted it. If you adapt these workflows, preserve the split.

## Customising

A few adjustment points:

- **Comment trigger gate.** The `author_association` check in both workflows lists `OWNER`, `MEMBER`, `COLLABORATOR`. Remove `COLLABORATOR` if you only want maintainer-level users to trigger.
- **Concurrency behaviour.** `issue-to-spec.yml` sets `cancel-in-progress: true` (later comments supersede earlier runs). `spec-to-implementation.yml` sets `cancel-in-progress: false` (interrupting a half-done implementation is worse than waiting). Change as needed.
- **Label colours.** `1f6feb` (blue) for `kiro-spec` and `a371f7` (purple) for `kiro-implement` match my cookbook convention. Feel free to change.
- **Prompt wording.** Both workflows embed the Kiro prompt as a heredoc in the YAML. Edit to taste — common tweaks include changing the number of clarifying questions (default 3–5), adjusting the spec file conventions (feature vs bugfix), or adding project-specific commit-message formatting rules.

## Troubleshooting

- **Workflow doesn't fire on label.** The workflow file must be on the default branch. Merge the PR that adds the workflow before trying to trigger it.
- **"Author-association" failures.** The commenter's `author_association` field is computed per-comment. If you comment as a bot account, the association may be `NONE` and the trigger will be gated out. Use your human GitHub account for slash-commands.
- **Spec branch has stale commits.** `issue-to-spec.yml` uses `force-with-lease` to re-push the spec branch if the workflow is re-triggered. This is intentional — each run produces a fresh spec. If you've pushed edits of your own to the spec branch, `force-with-lease` will refuse the push; the run will fail rather than silently clobber your work.
- **Implementation PR has odd commit shapes.** If Kiro produces one giant "implement all tasks" commit instead of commit-per-task, the prompt's commit-per-task instruction was ignored. This usually means the Kiro model context ran out. Break the spec into smaller features or split the tasks across multiple runs.

## Companion blog posts

- [Part 1: How I automated fixing GitHub bot review comments using Kiro and GitHub Actions](https://www.linkedin.com/pulse/how-i-automated-fixing-github-bot-review-comments-using-roberts-t5jqe/)
- [Part 2: From GitHub issue to merged PR, without opening my IDE](https://www.linkedin.com/pulse/from-github-issue-merged-pr-without-opening-my-ide-charles-roberts-9qjme/)

## Licence

Apache-2.0. Steal wholesale.
