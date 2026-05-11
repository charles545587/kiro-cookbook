# Kiro Cookbook

A growing collection of patterns, workflows, and steering documents for agentic development with [Kiro](https://kiro.dev).

Each recipe here is a complete, working example you can drop into your own project. They're sanitized of any project-specific details but keep enough context that you can see how they fit together in practice.

## What's in here

### GitHub Actions workflows

- **[PR auto-fix](github-workflows/pr-auto-fix/)** — When a trusted review bot (AWS Security Agent, Amazon Q Developer) leaves unresolved findings on a pull request, Kiro reads them, writes fixes, commits to the PR branch, and posts threaded replies on each original finding. The closing minutes of a PR, handled automatically.

- **[Spec-driven development](github-workflows/spec-driven/)** — A two-workflow chain that turns a labelled GitHub issue into a merged feature. `kiro-spec` label → spec PR via multi-turn conversation. `kiro-implement` label → implementation PR with one commit per task. Pairs with the PR auto-fix recipe to close the whole loop. The opening minutes of a feature, handled automatically.

Combine the two recipes and the full chain is:

```
Issue labelled kiro-spec
  │
  ▼  issue-to-spec.yml  (spec-driven)
Spec PR  →  merge
  │
  ▼  Issue labelled kiro-implement
  ▼  spec-to-implementation.yml  (spec-driven)
Implementation PR
  │
  ▼  Bots review
  ▼  pr-trigger.yml  (pr-auto-fix)
Fixes applied inline, PR goes green
  │
  ▼  Final human review  →  merge
Feature on main, issue auto-closes
```

*More recipes coming soon: scheduled code review, release notes generation, steering document patterns.*

## Using these recipes

Each directory has its own README explaining what the recipe does, what it needs, and how to adapt it.

The recipes assume:
- A GitHub repository where you can run Actions
- A [Kiro account](https://kiro.dev) with an API key (for headless workflows)
- The `gh` CLI if you're following along with the setup steps

## Companion write-ups

Long-form articles that walk through the design of these recipes and the problems they solve:

- [Part 1: How I automated fixing GitHub bot review comments using Kiro and GitHub Actions](https://www.linkedin.com/pulse/how-i-automated-fixing-github-bot-review-comments-using-roberts-t5jqe/)
- [Part 2: From GitHub issue to merged PR, without opening my IDE](https://www.linkedin.com/pulse/from-github-issue-merged-pr-without-opening-my-ide-charles-roberts-9qjme/)

## Contributing

Found a bug, or have a recipe you'd like to add? Open an issue or PR. Please sanitize any organization-specific details before submitting.

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
