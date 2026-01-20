---
name: convert-github-actions
description: Converts a GitHub Actions workflow to Buildkite following established best practices and translation rules
---

# Convert GitHub Actions Workflow to Buildkite

When converting a GitHub Actions workflow to Buildkite:

1. Review the Buildkite pipeline best practices in `TRANSLATION_RULES_GENERAL.md`
2. Review the GitHub Actions-specific translation rules in `github-actions/GITHUB_TRANSLATION_RULES.md`
3. Read the GitHub Actions workflow file provided by the user
4. Convert the workflow to Buildkite YAML format following all the rules
5. Save the resulting Buildkite pipeline YAML as a file in the local directory, with a name based on the input workflow file (e.g., `ci.yml` becomes `ci.buildkite.yaml`)

If the user hasn't specified which GitHub Actions workflow file to convert, ask them for the path.
