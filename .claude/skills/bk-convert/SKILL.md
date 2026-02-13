---
name: bk-convert
description: Converts a CI pipeline from another vendor to Buildkite, following established best practices and translation rules
---

# Convert CI pipeline to Buildkite

When converting a CI pipeline to Buildkite:

1. Ask the user what vendor the source pipeline was from, if they don't specify.
2. Review the Buildkite pipeline best practices in `BUILDKITE_BEST_PRACTICES.md`
3. Look for a vendor-specific rules file in the appropriate subdirectory and review that as well.
4. If no vendor-specific rules file is found, use `BUILDKITE_BEST_PRACTICES.md` along with your knowledge of the source vendor's product.
5. Read the CI pipeline file/data provided by the user.
6. Convert the pipeline to Buildkite YAML format following all the rules.
7. Save the resulting Buildkite pipeline YAML as a file in the local directory.

If the user hasn't specified a pipeline file to convert, ask them for the path.