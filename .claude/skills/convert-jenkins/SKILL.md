---
name: convert-jenkins
description: Converts a Jenkins pipeline to Buildkite following established best practices and translation rules
---

# Convert Jenkins Pipeline to Buildkite

When converting a Jenkins pipeline to Buildkite:

1. Review the Buildkite pipeline best practices in `TRANSLATION_RULES_GENERAL.md`
2. Review the Jenkins-specific translation rules in `jenkins/JENKINS_TRANSLATION_RULES.md`
3. Read the Jenkins pipeline file provided by the user
4. Convert the pipeline to Buildkite YAML format following all the rules
5. Save the resulting Buildkite pipeline YAML as a file in the local directory, with a name based on the input Jenkinsfile (e.g., `Jenkinsfile-MyApp` becomes `Jenkinsfile-MyApp.buildkite.yaml`)

If the user hasn't specified which Jenkins file to convert, ask them for the path.
