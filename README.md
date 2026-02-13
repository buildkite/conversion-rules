# Introduction

This repository contains a set of rules or prompts that can be used with Large Language Models to convert CI pipelines from various platforms into Buildkite pipeline YAML files. These rules are used in our web-based [Pipeline Converter](https://buildkite.com/resources/convert/) app and in the [Buildkite CLI.](https://github.com/buildkite/cli)

There is also a Claude Code skill included to help get you started. 

If you install the Buildkite CLI, your LLM agent will use the `bk pipeline validate` command to validate the YAML that is generated. You can also set up the [Buildkite MCP Server,](https://github.com/buildkite/buildkite-mcp-server) or just use the CLI, to enable the agent to create pipelines for you in your Buildkite org with that YAML.