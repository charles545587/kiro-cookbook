# Kiro Cookbook

A growing collection of patterns, workflows, and steering documents for agentic development with [Kiro](https://kiro.dev).

Each recipe here is a complete, working example you can drop into your own project. They're sanitized of any project-specific details but keep enough context that you can see how they fit together in practice.

## What's in here

### GitHub Actions workflows

- **[PR auto-fix](github-workflows/pr-auto-fix/)** — A workflow that reads unresolved findings from trusted review bots (AWS Security Agent, Amazon Q Developer) and uses Kiro's headless mode to write fixes, commit them, and reply inline on each thread.

*More recipes coming soon: issue-to-spec conversion, scheduled code review, release notes generation, steering document patterns.*

## Using these recipes

Each directory has its own README explaining what the recipe does, what it needs, and how to adapt it.

The recipes assume:
- A GitHub repository where you can run Actions
- A [Kiro account](https://kiro.dev) with an API key (for headless workflows)
- The `gh` CLI if you're following along with the setup steps

## Contributing

Found a bug, or have a recipe you'd like to add? Open an issue or PR. Please sanitize any organization-specific details before submitting.

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
