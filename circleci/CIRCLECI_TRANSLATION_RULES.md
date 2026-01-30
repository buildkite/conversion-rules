# CircleCI to Buildkite Translation Rules

This document provides rules for AI agents to translate CircleCI pipelines into Buildkite pipelines.

## Table of Contents

- [Terminology](#terminology)
- [Universal Rules](#universal-rules)
- [Quick Mappings](#quick-mappings)
- [Core Translation Principles](#core-translation-principles)
  - [Principle 1: YAML Anchors](#principle-1-yaml-anchors-work-natively)
  - [Principle 2: Executors → Docker Plugin + Agents](#principle-2-executors-map-to-docker-plugin--agents)
  - [Principle 3: Docker Image Compatibility](#principle-3-docker-image-compatibility-cimg-images)
  - [Principle 4: Orbs → Plugins or Scripts](#principle-4-orbs-map-to-plugins-or-native-commands)
  - [Principle 5: Reusable Commands](#principle-5-reusable-commands-become-scripts-or-dynamic-pipelines)
  - [Principle 6: Matrix Builds](#principle-6-matrix-builds-use-native-matrix)
  - [Principle 7: Parallelism & Test Splitting](#principle-7-parallelism-and-test-splitting)
  - [Principle 8: Caching](#principle-8-caching-uses-cache-plugin)
  - [Principle 9: Workspace → Artifacts](#principle-9-workspace-becomes-artifacts)
  - [Principle 10: Dynamic Config](#principle-10-dynamic-configuration-setup-true)
  - [Principle 11: Machine Executor](#principle-11-machine-executor)
  - [Principle 12: setup_remote_docker](#principle-12-setup_remote_docker)
  - [Principle 13: Contexts (Secrets)](#principle-13-contexts-secrets)
  - [Principle 14: Workflow Conditions with Parameters](#principle-14-workflow-conditions-with-pipeline-parameters)
  - [Principle 15: Branch Filter Ignore Patterns](#principle-15-branch-filter-ignore-patterns)
- [Detailed Translation Rules](#detailed-translation-rules)
- [Plugin Reference](#plugin-reference)
- [Translation Checklist](#translation-checklist)

---

## Terminology

| CircleCI | Buildkite |
|----------|-----------|
| Pipeline | Pipeline |
| Workflow | Pipeline (workflows become step dependencies) |
| Job | Step (command step) |
| Step | Commands within a step |
| Executor | Docker plugin + agents block |
| Orb | Plugin (partial equivalent) |
| Command (reusable) | Dynamic pipeline or YAML anchor |
| Context | Environment variables on agent |

---

## Universal Rules

These apply to every translation:

1. **Remove `version:` key** - Buildkite YAML doesn't use version declarations.
2. **Remove `setup: true`** - Dynamic configuration is handled differently (see Dynamic Pipelines section).
3. **Remove `orbs:` block** - Translate orb commands to native Buildkite equivalents or plugins.
4. **Remove `executors:` block** - Translate to Docker plugin and agents configuration per-step.
5. **Remove `commands:` block** - Translate to inline scripts, YAML anchors, or dynamic pipelines.
6. **Flatten workflows** - CircleCI workflows with job dependencies become Buildkite steps with `depends_on`.
7. **Remove `checkout` step** - Buildkite checks out code automatically.

---

## Quick Mappings

| CircleCI | Buildkite |
|----------|-----------|
| `jobs.<id>` | `steps` array item with `key: "<id>"` |
| `jobs.<id>.docker[0].image` | `plugins: [docker#: {image: "..."}]` |
| `jobs.<id>.resource_class` | `agents: { queue: "..." }` |
| `jobs.<id>.environment` | `env` |
| `jobs.<id>.working_directory` | Prepend `cd dir &&` to commands or use Docker plugin `workdir` |
| `workflows.<wf>.jobs.<job>.requires` | `depends_on` |
| `workflows.<wf>.jobs.<job>.filters.branches.only` | `if: build.branch == "main"` |
| `when: always` | `depends_on: [{ step: "X", allow_failure: true }]` |
| `when: on_fail` | `depends_on` with `allow_failure: true` + `if: build.state == "failing"` |
| `store_artifacts` | `artifact_paths` on the step |
| `store_test_results` | Buildkite Test Engine (requires separate configuration) |
| `persist_to_workspace` | `buildkite-agent artifact upload` |
| `attach_workspace` | `buildkite-agent artifact download` |
| `save_cache` / `restore_cache` | Cache plugin |
| `<< pipeline.parameters.X >>` | `${X}` environment variable |
| `<< parameters.X >>` | `${X}` environment variable (step-level) |

---

## Core Translation Principles

These principles guide translation decisions for patterns not explicitly covered in Quick Mappings.

### Principle 1: YAML Anchors Work Natively

CircleCI and Buildkite both support standard YAML anchors. Translate them as-is.

```yaml
# CircleCI
defaults: &defaults
  docker:
    - image: node:18
  working_directory: ~/app

jobs:
  build:
    <<: *defaults
    steps:
      - run: npm build

# Buildkite - anchors work the same way
definitions:
  node_plugin: &node_plugin
    plugins:
      - docker#:
          image: node:18
          workdir: /app

steps:
  - label: "build"
    <<: *node_plugin
    command: npm build
```

**Note:** YAML anchors only support static value substitution. For parameterized reuse, see Principle 5 (Reusable Commands).

---

### Principle 2: Executors Map to Docker Plugin + Agents

CircleCI executors define the execution environment (Docker image, machine type, environment variables). Buildkite separates these concerns:

| Executor Component | Buildkite Equivalent |
|-------------------|---------------------|
| `docker[].image` | Docker plugin `image` |
| `docker[].environment` | Docker plugin `environment` |
| `resource_class` | `agents: { queue: "..." }` |
| `shell` | Docker plugin `shell` or agent configuration |
| `working_directory` | Docker plugin `workdir` |

**Simple executor translation:**

```yaml
# CircleCI
executors:
  node-executor:
    docker:
      - image: cimg/node:18.17
    resource_class: medium
    environment:
      NODE_ENV: test

jobs:
  test:
    executor: node-executor
    steps:
      - run: npm test

# Buildkite
steps:
  - label: "test"
    env:
      NODE_ENV: test
    plugins:
      - docker#:
          image: cimg/node:18.17
    # agents:
    #   queue: "medium"  # Configure based on your infrastructure
    command: npm test
```

> 📋 **Note: Consider Replacing `cimg/*` Images Post-Migration**
>
> CircleCI's convenience images (`cimg/*`) are designed specifically for CircleCI's environment and require several workarounds to function correctly in Buildkite (see warnings below). During initial conversion, it's reasonable to keep these images to minimize changes and maintain the same runtime environment. However, consider replacing them with standard Docker Hub images (`node:18`, `python:3.11`, `ruby:3.2`, etc.) as a follow-up task. Standard images:
> - Don't require login shell workarounds
> - Don't have UID/GID permission issues (run as root by default)
> - Don't break `buildkite-agent` PATH
> - Are more portable across CI platforms
> - Have fewer CircleCI-specific assumptions baked in

> ⚠️ **Critical: CircleCI `cimg/*` Images Require Login Shell**
>
> CircleCI's convenience images (`cimg/node`, `cimg/python`, `cimg/ruby`, etc.) install language runtimes via version managers like `nvm`, `pyenv`, or `rvm`. These version managers configure the PATH in shell profile files (`.bashrc`, `.bash_profile`) which are only sourced by **login shells**.
>
> The Buildkite Docker plugin defaults to `/bin/sh -e -c` which does NOT source these profiles, causing commands like `node`, `yarn`, `python`, or `ruby` to fail with "command not found" or version errors.
>
> **Solution:** Add the `shell` option to use a bash login shell:
>
> ```yaml
> plugins:
>   - docker#:
>       image: cimg/node:18.17
>       shell:
>         - "/bin/bash"
>         - "-l"    # login shell - sources profile files
>         - "-e"    # exit on error
>         - "-c"    # command follows
> ```
>
> **Alternative:** Use official Docker Hub images (`node:18`, `python:3.11`, `ruby:3.2`) which install runtimes directly in the system PATH and don't require login shells.

> ⚠️ **Critical: CircleCI `cimg/*` Images Require UID/GID Propagation**
>
> CircleCI `cimg/*` images run as a non-root user (`circleci`, UID 3434). When the Buildkite Docker plugin mounts the checkout directory, it may be owned by a different UID (typically root or the host agent user), causing permission errors like:
>
> ```
> EACCES: permission denied, mkdir '/workdir/node_modules'
> EACCES: permission denied, open '/workdir/yarn-error.log'
> ```
>
> **Solution:** Add `propagate-uid-gid: true` to run the container as the same UID/GID as the host:
>
> ```yaml
> plugins:
>   - docker#:
>       image: cimg/node:18.17
>       shell:
>         - "/bin/bash"
>         - "-l"
>         - "-e"
>         - "-c"
>       propagate-uid-gid: true  # Fix file permission issues
> ```
>
> **Note:** Both `shell` (for login shell) and `propagate-uid-gid` (for permissions) are typically required together when using `cimg/*` images.

> ⚠️ **Critical: CircleCI `cimg/*` Images Break `buildkite-agent` PATH**
>
> The Docker plugin mounts the `buildkite-agent` binary into containers at `/usr/bin/buildkite-agent` by default (`mount-buildkite-agent: true`). However, when using login shells with `cimg/*` images, the shell profile scripts may reset or modify PATH in ways that exclude `/usr/bin`, causing errors like:
>
> ```
> /bin/bash: line 19: buildkite-agent: command not found
> ```
>
> **Solution:** Use the `artifacts` plugin instead of calling `buildkite-agent artifact upload` from within the container. The artifacts plugin runs on the host after the Docker container exits:
>
> ```yaml
> # Instead of calling buildkite-agent inside Docker:
> steps:
>   - label: "Build"
>     command: |
>       npm run build
>       # Don't do this - it won't work with cimg/* + login shell:
>       # buildkite-agent artifact upload "dist/**/*"
>     plugins:
>       - docker#:
>           image: cimg/node:18.17
>           shell: ["/bin/bash", "-l", "-e", "-c"]
>           propagate-uid-gid: true
>       - artifacts#:
>           upload: "dist/**/*"  # Runs on host after Docker exits
> ```
>
> This pattern also applies to `buildkite-agent meta-data set/get` and `buildkite-agent annotate` commands.

> ⚠️ **Critical: Docker Plugin Environment Variables Do Not Inherit Step-Level `env:`**
>
> When using the Docker plugin, step-level `env:` variables are **not** automatically available inside the container. The Docker plugin's `environment:` list only passes variables that exist in the **host environment** at plugin initialization time—before Buildkite processes the step's `env:` block.
>
> **This will NOT work:**
>
> ```yaml
> steps:
>   - label: "Build"
>     env:
>       NODE_KEY: "node-18"  # ❌ Step-level env variable
>     plugins:
>       - docker#:
>           image: cimg/node:18.2.0
>           environment:
>             - NODE_KEY  # ❌ Docker looks in HOST env, not step env - results in empty value!
>     command: |
>       echo $NODE_KEY  # Prints nothing!
>       mkdir -p .artifacts/$NODE_KEY  # Creates .artifacts/ instead of .artifacts/node-18/
> ```
>
> **Solution: Use explicit assignment in the Docker plugin's `environment:` list:**
>
> ```yaml
> steps:
>   - label: "Build"
>     plugins:
>       - docker#:
>           image: cimg/node:18.2.0
>           environment:
>             - NODE_KEY=node-18  # ✅ Explicit assignment always works
>             - NODE_OPTIONS=--max-old-space-size=2048  # ✅ Works for any variable
>     command: |
>       echo $NODE_KEY  # Prints "node-18"
>       mkdir -p .artifacts/$NODE_KEY  # Creates .artifacts/node-18/ correctly
> ```
>
> **When this affects you:**
> - Any time you use the Docker plugin with custom environment variables
> - Variables needed inside the container for build logic (paths, flags, configuration)
> - Test runner configuration (Jest, Mocha, etc.): `NODE_OPTIONS`, `JEST_*`, `CI`, etc.
>
> **Two valid approaches:**
> 1. **Explicit assignment** (recommended): `- VAR_NAME=value` in the Docker plugin's `environment:` list
> 2. **Host environment**: Variables already set on the agent (e.g., from agent hooks or system environment) can be passed by name alone: `- VAR_NAME`
>
> **Note:** Variables like `CI`, `BUILDKITE`, and `BUILDKITE_*` are set by the Buildkite agent on the host, so they work with name-only syntax. Custom variables you define in the pipeline YAML do not.

**Parameterized executor translation:**

CircleCI executors can accept parameters. Translate using environment variables:

```yaml
# CircleCI
executors:
  node-executor:
    parameters:
      node_version:
        type: string
        default: "18"
    docker:
      - image: cimg/node:<< parameters.node_version >>

jobs:
  test:
    executor:
      name: node-executor
      node_version: "20"

# Buildkite - use environment variable for parameterization
steps:
  - label: "test"
    env:
      NODE_VERSION: "20"
    plugins:
      - docker#:
          image: cimg/node:${NODE_VERSION}
    command: npm test
```

For multiple executor variations, consider using matrix builds (see Principle 3).

---

### Principle 3: Matrix Builds Map Directly

CircleCI matrix jobs translate to Buildkite's native matrix support with minor syntax differences.

**Basic mapping:**

| CircleCI | Buildkite |
|----------|-----------|
| `matrix.parameters` | `matrix.setup` |
| `matrix.exclude` | `matrix.adjustments` with `skip: true` |
| `<< matrix.X >>` | `{{matrix.X}}` |

**Simple matrix:**

```yaml
# CircleCI
jobs:
  test:
    parameters:
      node_version:
        type: string
    docker:
      - image: cimg/node:<< parameters.node_version >>
    steps:
      - run: npm test

workflows:
  test-matrix:
    jobs:
      - test:
          matrix:
            parameters:
              node_version: ["18", "20", "22"]

# Buildkite
steps:
  - label: "test node-{{matrix.node_version}}"
    plugins:
      - docker#:
          image: cimg/node:{{matrix.node_version}}
    command: npm test
    matrix:
      setup:
        node_version:
          - "18"
          - "20"
          - "22"
```

**Multi-dimensional matrix with exclusions:**

```yaml
# CircleCI
workflows:
  test-matrix:
    jobs:
      - test:
          matrix:
            parameters:
              os: ["linux", "macos"]
              node_version: ["18", "20"]
            exclude:
              - os: macos
                node_version: "18"

# Buildkite
steps:
  - label: "test {{matrix.os}} node-{{matrix.node_version}}"
    command: npm test
    matrix:
      setup:
        os:
          - "linux"
          - "macos"
        node_version:
          - "18"
          - "20"
      adjustments:
        - with:
            os: "macos"
            node_version: "18"
          skip: true
```

**Matrix limits:** Buildkite supports up to 6 dimensions, 20 elements per dimension, 12 adjustments, and 50 total jobs per matrix step.

---

### Principle 4: Test Splitting Requires Test Engine

CircleCI's `circleci tests split` command distributes tests across parallel containers using timing data. The direct Buildkite equivalent is **Test Engine**, a paid service that provides similar functionality.

**Rule:** Translate test splitting as commented-out code with an explanation:

```yaml
# CircleCI
jobs:
  test:
    parallelism: 4
    steps:
      - run:
          name: Run tests
          command: |
            TESTS=$(circleci tests glob "spec/**/*_spec.rb" | circleci tests split --split-by=timings)
            bundle exec rspec $TESTS

# Buildkite
steps:
  - label: "test"
    # parallelism: 4
    command: |
      # NOTE: CircleCI test splitting requires Buildkite Test Engine for equivalent functionality.
      # Test Engine is a paid service that provides intelligent test distribution based on timing data.
      # See: https://buildkite.com/docs/test-engine
      #
      # Without Test Engine, options include:
      # 1. Run all tests in a single job (simplest)
      # 2. Manually partition tests across parallel jobs
      # 3. Use a third-party test splitting tool
      #
      # Original CircleCI command:
      # TESTS=$(circleci tests glob "spec/**/*_spec.rb" | circleci tests split --split-by=timings)
      # bundle exec rspec $TESTS
      
      bundle exec rspec spec/
```

---

### Principle 5: Reusable Commands Become Dynamic Pipelines

CircleCI `commands` are reusable step sequences with parameters and conditionals. YAML anchors cannot replicate this because they don't support parameterization or conditional logic.

**Translation approach:** Use inline shell scripts that generate pipeline YAML dynamically.

**Pattern: Inline dynamic pipeline generation**

```yaml
# CircleCI
commands:
  run-e2e-tests:
    parameters:
      test_path:
        type: string
      browser:
        type: string
        default: "chrome"
      retry_count:
        type: integer
        default: 2
    steps:
      - run:
          name: Run E2E tests
          command: |
            cd << parameters.test_path >>
            npm run e2e -- --browser=<< parameters.browser >>
      - when:
          condition: << parameters.retry_count >>
          steps:
            - run: echo "Retries enabled: << parameters.retry_count >>"

jobs:
  e2e-checkout:
    steps:
      - run-e2e-tests:
          test_path: "packages/checkout"
          browser: "firefox"

# Buildkite - inline script generates steps dynamically
steps:
  - label: ":pipeline: E2E checkout"
    env:
      TEST_PATH: "packages/checkout"
      BROWSER: "firefox"
      RETRY_COUNT: "2"
    command: |
      #!/bin/bash
      set -euo pipefail
      
      cat <<PIPELINE | buildkite-agent pipeline upload
      steps:
        - label: "E2E $TEST_PATH ($BROWSER)"
          retry:
            automatic:
              - exit_status: "*"
                limit: $RETRY_COUNT
          command: |
            cd "$TEST_PATH"
            npm run e2e -- --browser="$BROWSER"
      PIPELINE
```

**Key benefits of this pattern:**

1. **Self-contained** - Everything stays in one YAML file
2. **Parameterized** - Environment variables act like command parameters
3. **Conditional logic** - Bash `if`/`case` statements handle CircleCI's `when`/`unless`
4. **No external files** - Users don't need to add scripts to their repo

**Handling conditionals (`when`/`unless`):**

```yaml
# CircleCI
commands:
  deploy:
    parameters:
      environment:
        type: string
      notify:
        type: boolean
        default: true
    steps:
      - run: ./deploy.sh << parameters.environment >>
      - when:
          condition: << parameters.notify >>
          steps:
            - run: ./notify-slack.sh

# Buildkite
steps:
  - label: ":pipeline: Deploy"
    env:
      ENVIRONMENT: "production"
      NOTIFY: "true"
    command: |
      #!/bin/bash
      set -euo pipefail
      
      # Build the pipeline dynamically
      cat <<PIPELINE
      steps:
        - label: "Deploy to $ENVIRONMENT"
          command: ./deploy.sh "$ENVIRONMENT"
      PIPELINE
      
      # Conditional step based on NOTIFY parameter
      if [[ "$NOTIFY" == "true" ]]; then
        cat <<PIPELINE
        - label: "Notify Slack"
          command: ./notify-slack.sh
      PIPELINE
      fi
      
      # Upload the generated pipeline
      ) | buildkite-agent pipeline upload
```

**For simple command reuse (no parameters/conditionals):** Use YAML anchors instead:

```yaml
definitions:
  install_deps: &install_deps
    - npm ci
    - npm run bootstrap

steps:
  - label: "build"
    command:
      - *install_deps
      - npm run build

  - label: "test"
    command:
      - *install_deps
      - npm test
```

---

### Principle 6: Workspace and Caching

CircleCI has two distinct concepts for sharing data:
- **Workspaces**: Pass build outputs between jobs *within a single workflow run*
- **Caches**: Persist dependencies *across builds* (e.g., node_modules)

#### Workspace Translation (Artifacts)

CircleCI workspaces translate to Buildkite artifacts using `buildkite-agent artifact upload/download`.

**Simple case (no matrix):**

```yaml
# CircleCI
jobs:
  build:
    steps:
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist/
            - node_modules/

  deploy:
    steps:
      - attach_workspace:
          at: .
      - run: npm run deploy

# Buildkite
steps:
  - label: "build"
    key: "build"
    command:
      - npm run build
      - buildkite-agent artifact upload "dist/**/*"
      - buildkite-agent artifact upload "node_modules/**/*"

  - label: "deploy"
    depends_on: "build"
    command:
      - buildkite-agent artifact download "dist/**/*" .
      - buildkite-agent artifact download "node_modules/**/*" .
      - npm run deploy
```

**Matrix case (namespaced artifacts):**

When matrix jobs each produce their own workspace, artifacts must be namespaced to avoid overwrites. Downstream jobs download only the artifacts from their specific dependency.

```yaml
# CircleCI - each matrix job creates its own workspace
jobs:
  bootstrap:
    parameters:
      node_version:
        type: string
    steps:
      - run: yarn bootstrap
      - persist_to_workspace:
          root: ./
          paths:
            - "packages/"
            - "node_modules/"

  unit_tests_node18:
    steps:
      - attach_workspace:
          at: .
      - run: yarn test

  unit_tests_node20:
    steps:
      - attach_workspace:
          at: .
      - run: yarn test

workflows:
  build-test:
    jobs:
      - bootstrap:
          matrix:
            parameters:
              node_version: ["18.0.0", "20.0.0"]
      - unit_tests_node18:
          requires:
            - bootstrap-18.0.0
      - unit_tests_node20:
          requires:
            - bootstrap-20.0.0

# Buildkite - namespace artifacts by matrix value
steps:
  - label: "bootstrap-{{matrix.node_version}}"
    key: "bootstrap-{{matrix.node_version}}"
    matrix:
      setup:
        node_version:
          - "18.0.0"
          - "20.0.0"
    command:
      - yarn bootstrap
      # Namespace artifacts to avoid overwrites between matrix jobs
      - mkdir -p .artifacts/{{matrix.node_version}}
      - tar czf .artifacts/{{matrix.node_version}}/workspace.tar.gz packages node_modules
      - buildkite-agent artifact upload ".artifacts/{{matrix.node_version}}/workspace.tar.gz"

  - label: "unit-tests-node18"
    key: "unit-tests-node18"
    depends_on: "bootstrap-18.0.0"
    command:
      # Download and extract the specific workspace
      - buildkite-agent artifact download ".artifacts/18.0.0/workspace.tar.gz" .
      - tar xzf .artifacts/18.0.0/workspace.tar.gz
      - yarn test

  - label: "unit-tests-node20"
    key: "unit-tests-node20"
    depends_on: "bootstrap-20.0.0"
    command:
      - buildkite-agent artifact download ".artifacts/20.0.0/workspace.tar.gz" .
      - tar xzf .artifacts/20.0.0/workspace.tar.gz
      - yarn test
```

**Key points for workspace translation:**
- Use tar/compression for large directory trees (faster upload/download)
- Namespace by matrix value when multiple jobs persist overlapping paths
- Downstream jobs explicitly reference their dependency's namespace

#### Cache Translation (Cache Plugin)

CircleCI caches (for dependencies across builds) translate to the Buildkite cache plugin.

```yaml
# CircleCI
jobs:
  build:
    steps:
      - restore_cache:
          keys:
            - v1-deps-{{ checksum "package-lock.json" }}
            - v1-deps-
      - run: npm ci
      - save_cache:
          key: v1-deps-{{ checksum "package-lock.json" }}
          paths:
            - node_modules

# Buildkite
# ============================================================================
# CACHING OPTIONS
# ============================================================================
# Option 1: Cache plugin (works with any agent)
#   Requires backend configuration (S3 or filesystem).
#   See: https://github.com/buildkite-plugins/cache-buildkite-plugin
#
# Option 2: Buildkite Hosted Agents Cache Volumes (if using hosted agents)
#   Simpler configuration using the 'cache' key in pipeline YAML.
#   See: https://buildkite.com/docs/pipelines/hosted-agents/cache-volumes
#   Note: Cache volumes are best-effort and non-deterministic - design
#   workflows to handle cache misses gracefully.
# ============================================================================
steps:
  - label: "build"
    plugins:
      - cache#:
          manifest: package-lock.json
          path: node_modules
          restore: file
          save: file
    command:
      - npm ci
```

**Cache plugin features:**
- `manifest`: File(s) to hash for cache key (like CircleCI's `{{ checksum }}`)
- `restore`/`save`: Caching levels (`file`, `step`, `branch`, `pipeline`, `all`)
- `key-extra`: Additional cache key differentiation (useful for matrix builds)
- `compression`: `tgz`, `zstd`, or `zip` for faster transfers
- `backend`: `fs` (local) or `s3` (shared across agents)

**Fallback keys:** CircleCI's fallback key pattern (`v1-deps-{{ checksum }}`, then `v1-deps-`) maps to cache plugin's hierarchical restore levels:

```yaml
# CircleCI fallback pattern
- restore_cache:
    keys:
      - v1-deps-{{ checksum "package-lock.json" }}  # Exact match
      - v1-deps-                                      # Fallback

# Buildkite equivalent - restore checks levels in order
plugins:
  - cache#:
      manifest: package-lock.json
      path: node_modules
      restore: pipeline  # Checks: file → step → branch → pipeline
      save: file
```

**Alternative: Buildkite Hosted Agents Cache Volumes**

If using Buildkite Hosted Agents, cache volumes provide a simpler native caching mechanism:

```yaml
# Buildkite with Hosted Agents cache volumes
cache:
  paths:
    - "node_modules"
  size: "20g"

steps:
  - label: "build"
    command: npm ci
```

Note: Cache volumes are retained up to 14 days and attached on a best-effort basis. They are ideal for dependencies but should not be used for passing build outputs between jobs (use artifacts for that).

---

### Principle 7: Scheduled Workflows

CircleCI scheduled workflows are configured in the YAML. Buildkite schedules are configured in the UI.

**Translation approach:** Convert the CircleCI schedule configuration into a header comment that documents what needs to be set up manually in Buildkite.

```yaml
# CircleCI - schedule defined in config
workflows:
  nightly:
    triggers:
      - schedule:
          cron: "0 0 * * *"
          filters:
            branches:
              only:
                - main
    jobs:
      - nightly-tests

# Buildkite - document the schedule for manual configuration
# ============================================================================
# SCHEDULED BUILD CONFIGURATION
# ============================================================================
# The following schedule was defined in the original CircleCI config.
# Configure in Buildkite: Pipeline Settings → Schedules → New Schedule
#
#   Cron Expression: 0 0 * * *
#   Branch: main
# ============================================================================
steps:
  - label: "nightly-tests"
    command: npm run test:nightly
```

**Multiple schedules:** If the CircleCI config has multiple scheduled workflows, document each one:

```yaml
# ============================================================================
# SCHEDULED BUILD CONFIGURATION
# ============================================================================
# Configure these schedules in Buildkite: Pipeline Settings → Schedules
#
#   Name        | Cron Expression | Branch
#   ------------|-----------------|--------
#   Nightly     | 0 0 * * *       | main
#   Weekly      | 0 0 * * 0       | main
#   Hourly Dev  | 0 * * * *       | develop
# ============================================================================
```

---

### Principle 8: Windows and macOS Jobs

CircleCI uses `machine` executors with specific images for Windows/macOS. Buildkite uses agent queues.

```yaml
# CircleCI
jobs:
  windows-build:
    executor:
      name: win/default
      size: medium
    steps:
      - run: npm test

  macos-build:
    macos:
      xcode: 14.2.0
    steps:
      - run: xcodebuild test

# Buildkite
steps:
  - label: "windows build"
    agents:
      queue: "windows"
      # Or use agent tags:
      # os: "windows"
    command: npm test

  - label: "macos build"
    agents:
      queue: "macos"
      # Or use agent tags:
      # os: "macos"
      # xcode: "14.2"
    command: xcodebuild test
```

**Shell selection on Windows:**

CircleCI allows specifying `shell:` at the executor or step level. Buildkite doesn't have a shell attribute - instead, invoke the shell directly in the command:

```yaml
# CircleCI - mixed shells in one job
jobs:
  windows-tests:
    executor:
      name: win/default
      shell: powershell.exe
    steps:
      - run:
          command: ./scripts/check-files.sh
          shell: bash.exe
      - run:
          name: Clean build
          command: Remove-Item -Recurse -Force -Path "node_modules/"
      - run:
          name: Run tests
          command: yarn test

# Buildkite - specify shell per command
steps:
  - label: "windows tests"
    agents:
      queue: "windows"
    command:
      - bash ./scripts/check-files.sh
      - pwsh -Command "Remove-Item -Recurse -Force -Path 'node_modules/'"
      - yarn test
```

**Common shell invocations:**

| CircleCI `shell:` | Buildkite command prefix |
|-------------------|-------------------------|
| `bash.exe` | `bash` or `bash -c "..."` |
| `powershell.exe` | `pwsh` or `pwsh -Command "..."` |
| `cmd.exe` | `cmd /c "..."` |

**Note:** For multi-line PowerShell scripts, create a `.ps1` file and call it with `pwsh script.ps1`, or use a heredoc:

```yaml
steps:
  - label: "windows build"
    agents:
      queue: "windows"
    command: |
      pwsh -Command @'
        Remove-Item -Recurse -Force -Path "node_modules/sharp/"
        yarn install
        yarn test
      '@
```

**Document agent requirements:**

```yaml
# ============================================================================
# AGENT CONFIGURATION REQUIRED
# ============================================================================
# This pipeline requires agents with specific operating systems:
#
#   Job              | Original Executor        | Required Agent
#   -----------------|--------------------------|------------------
#   windows-build    | win/default (medium)     | queue: windows
#   macos-build      | macos (xcode 14.2.0)     | queue: macos
#
# Configure agents with appropriate tags or use Buildkite's hosted agents.
# ============================================================================
```

---

### Principle 9: Orb Commands to Native Commands

CircleCI orbs are reusable packages of configuration. Translate orb commands to their underlying implementations.

**Common orb translations:**

| Orb Command | Buildkite Equivalent |
|-------------|---------------------|
| `node/install-packages` | `npm ci` or cache plugin + `npm ci` |
| `docker/build` | `docker build` command |
| `docker/push` | `docker push` command |
| `aws-cli/setup` | Assume AWS CLI installed on agent |
| `slack/notify` | Slack notification plugin or `curl` to webhook |
| `continuation/continue` | `buildkite-agent pipeline upload` |

**Example - Docker orb:**

```yaml
# CircleCI
orbs:
  docker: circleci/docker@2.2.0

jobs:
  build:
    executor: docker/docker
    steps:
      - docker/check
      - docker/build:
          image: myapp
          tag: latest
      - docker/push:
          image: myapp
          tag: latest

# Buildkite
steps:
  - label: "build and push"
    plugins:
      - docker-login#:
          username: ${DOCKER_USERNAME}
          password-env: DOCKER_PASSWORD
    command:
      - docker build -t myapp:latest .
      - docker push myapp:latest
```

**Example - Slack orb:**

```yaml
# CircleCI
orbs:
  slack: circleci/slack@4.12.0

jobs:
  notify:
    steps:
      - slack/notify:
          event: fail
          template: basic_fail_1

# Buildkite - use slack-notification plugin or curl
steps:
  - label: "notify"
    plugins:
      - slack-notification#:
          webhook-url: ${SLACK_WEBHOOK_URL}
          message: "Build failed!"
    if: build.state == "failing"
```

---

### Principle 10: Dynamic Configuration (setup: true)

CircleCI's dynamic configuration pattern uses `setup: true` with the continuation orb to generate pipelines at runtime. Buildkite handles this with `buildkite-agent pipeline upload`.

```yaml
# CircleCI
setup: true
orbs:
  continuation: circleci/continuation@0.3.1

jobs:
  setup:
    steps:
      - run:
          name: Generate config
          command: ./generate-config.sh > generated-config.yml
      - continuation/continue:
          configuration_path: generated-config.yml

# Buildkite
steps:
  - label: ":pipeline: Generate pipeline"
    command: |
      ./generate-config.sh | buildkite-agent pipeline upload
```

**For path-based dynamic configuration:**

```yaml
# CircleCI using path-filtering orb
setup: true
orbs:
  path-filtering: circleci/path-filtering@0.1.3

jobs:
  setup:
    steps:
      - path-filtering/set-parameters:
          mapping: |
            packages/api/.* run-api-tests true
            packages/web/.* run-web-tests true

# Buildkite - use if_changed on steps
steps:
  - label: "API tests"
    command: npm run test:api
    if_changed:
      - "packages/api/**"

  - label: "Web tests"
    command: npm run test:web
    if_changed:
      - "packages/web/**"
```

---

### Principle 11: Machine Executor

CircleCI's `machine` executor runs jobs on a dedicated full virtual machine instead of inside a Docker container. This is used when you need native Docker access, kernel-level operations, or complex multi-container integration tests.

**Translation approach:** Use `agents.queue` to route jobs to VM-based agents.

```yaml
# CircleCI
jobs:
  integration-test:
    machine:
      image: ubuntu-2404:2024.04.4
    resource_class: xlarge
    steps:
      - checkout
      - run: docker-compose up -d
      - run: ./run-integration-tests.sh

# Buildkite
steps:
  - label: ":ubuntu: Integration Test"
    commands:
      - "docker-compose up -d"
      - "./run-integration-tests.sh"
    agents:
      queue: "linux-vm"
    # NOTE: This step requires a VM-based agent queue.
    # Create a queue named "linux-vm" and connect self-hosted agents running on VMs,
    # or use Buildkite Hosted Agents: https://buildkite.com/docs/pipelines/hosted-agents
```

**Key points:**
- No executor block needed—agent targeting is done per-step with `agents.queue`
- Customer needs to provision VMs with Buildkite agent installed and connect them to the specified queue
- Buildkite Hosted Agents provide managed VMs if customers don't want to self-host
- No `checkout` step needed—Buildkite agents check out code automatically

---

### Principle 12: setup_remote_docker

CircleCI's `setup_remote_docker` step creates a remote Docker environment when running inside a Docker executor (container). This is needed because containers don't have access to a Docker daemon.

**Translation approach:** Remove `setup_remote_docker` entirely—Buildkite agents typically run on VMs where Docker is available natively.

```yaml
# CircleCI
jobs:
  build-image:
    docker:
      - image: cimg/base:2024.01
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true
      - run: docker build -t myapp:latest .
      - run: docker push myapp:latest

# Buildkite
steps:
  - label: ":docker: Build Image"
    commands:
      - "docker build -t myapp:latest ."
      - "docker push myapp:latest"
    # NOTE: Assumes agent runs on a VM with Docker installed.
    # If using containerized agents, mount the Docker socket or use a VM-based queue.
```

**Key points:**
- Buildkite agents run on VMs by default, so Docker is available natively
- Simply run `docker build` and `docker push` commands directly
- If agents run in containers (e.g., Kubernetes), mount the Docker socket (`/var/run/docker.sock`) or route to a VM-based queue

---

### Principle 13: Context (Environment Variable Collections)

CircleCI contexts are named collections of environment variables defined in the CircleCI UI and attached to jobs at the workflow level. They're commonly used for secrets but can contain any key-value pairs.

```yaml
# CircleCI
workflows:
  deploy:
    jobs:
      - docker-build:
          context:
            - aws-credentials
            - deploy-settings
```

**Translation approach:** Buildkite doesn't have an equivalent "named collection" feature. Translate based on content type:

**For secrets:** Use Buildkite cluster secrets with the `secrets:` block in pipeline YAML:

```yaml
# Buildkite
steps:
  - label: ":docker: Build"
    commands:
      - "docker build -t myapp ."
      - "docker push myapp"
    secrets:
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY
    # NOTE: These secrets must be created in your Buildkite cluster.
    # See: https://buildkite.com/docs/pipelines/security/secrets/buildkite-secrets
    # Alternative: Use an external secrets manager (Vault, AWS Secrets Manager) with appropriate plugins.
```

**For non-secret environment variables:** Put them directly in the `env:` block:

```yaml
# Buildkite
steps:
  - label: "Deploy"
    env:
      DEPLOY_TIMEOUT: "300"
      RETRY_COUNT: "3"
      NOTIFY_CHANNEL: "deploys"
    command: "./deploy.sh"
```

**For shared non-secret variables across many steps:** Use YAML anchors to reduce duplication:

```yaml
# Buildkite
definitions:
  deploy_env: &deploy_env
    DEPLOY_TIMEOUT: "300"
    RETRY_COUNT: "3"
    NOTIFY_CHANNEL: "deploys"

steps:
  - label: "Deploy to staging"
    env:
      <<: *deploy_env
      ENVIRONMENT: "staging"
    command: "./deploy.sh"

  - label: "Deploy to production"
    env:
      <<: *deploy_env
      ENVIRONMENT: "production"
    command: "./deploy.sh"
```

**Key points:**
- Secrets → Buildkite cluster secrets (`secrets:` block) or external secrets manager with plugins
- Non-secrets → inline `env:` block, use YAML anchors if shared across steps
- Context names are not preserved—document which variables came from which context in comments if helpful

---

### Principle 14: Workflow Conditions with Pipeline Parameters

CircleCI allows conditionally running entire workflows based on pipeline parameters using `when` with boolean logic (`and`, `or`, `not`). This creates different "modes" of pipeline execution from a single config file.

```yaml
# CircleCI
parameters:
  scan-docker-images:
    type: boolean
    default: false
  qa-release:
    type: boolean
    default: false

workflows:
  scan-docker-images:
    when: << pipeline.parameters.scan-docker-images >>
    jobs:
      - trivy-scan-docker

  build-and-release:
    when:
      and:
        - not: << pipeline.parameters.scan-docker-images >>
        - not: << pipeline.parameters.qa-release >>
    jobs:
      - build
      - test
      - deploy

  qa-build-and-release:
    when: << pipeline.parameters.qa-release >>
    jobs:
      - build
      - deploy-to-qa
```

**Translation approach:** Use separate pipeline YAML files for each workflow mode, with a main pipeline that dynamically uploads the appropriate one based on user input or environment variables.

**Option A: Input step for interactive selection**

Create the following files in your repository:

```
.buildkite/
├── pipeline.yml              # Main entry point
└── workflows/
    ├── build-and-release.yml
    ├── scan-docker-images.yml
    └── qa-release.yml
```

---

**`.buildkite/pipeline.yml`**

```yaml
steps:
  - input: "Select workflow"
    key: "workflow-select"
    fields:
      - select: "Workflow type"
        key: "workflow"
        default: "build-and-release"
        options:
          - label: "Build and Release"
            value: "build-and-release"
          - label: "Scan Docker Images"
            value: "scan-docker-images"
          - label: "QA Release"
            value: "qa-release"

  - label: ":pipeline: Run selected workflow"
    depends_on: "workflow-select"
    command: |
      WORKFLOW=$(buildkite-agent meta-data get workflow)
      buildkite-agent pipeline upload ".buildkite/workflows/$${WORKFLOW}.yml"
```

---

**`.buildkite/workflows/build-and-release.yml`**

```yaml
steps:
  - label: ":hammer: Build"
    key: "build"
    command: "make build"

  - label: ":test_tube: Test"
    key: "test"
    depends_on: "build"
    command: "make test"

  - label: ":rocket: Deploy"
    depends_on: "test"
    command: "make deploy"
```

---

**`.buildkite/workflows/scan-docker-images.yml`**

```yaml
steps:
  - label: ":mag: Trivy Scan"
    command: "trivy image my-image:latest"
```

---

**`.buildkite/workflows/qa-release.yml`**

```yaml
steps:
  - label: ":hammer: Build"
    key: "build"
    command: "make build"

  - label: ":rocket: Deploy to QA"
    depends_on: "build"
    command: "make deploy-qa"
```

---

**Option B: Environment variable for API/scheduled triggers**

When triggering via API or schedules, pass the workflow type as an environment variable. Use the same file structure as Option A, but with a simpler main pipeline:

**`.buildkite/pipeline.yml`**

```yaml
steps:
  - label: ":pipeline: Run workflow"
    command: |
      # WORKFLOW_TYPE can be set via API trigger or schedule
      WORKFLOW="$${WORKFLOW_TYPE:-build-and-release}"
      buildkite-agent pipeline upload ".buildkite/workflows/$${WORKFLOW}.yml"
```

Trigger via API with:
```bash
curl -X POST "https://api.buildkite.com/v2/organizations/{org}/pipelines/{pipeline}/builds" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "commit": "HEAD",
    "branch": "main",
    "env": {
      "WORKFLOW_TYPE": "scan-docker-images"
    }
  }'
```

---

**Option C: Combine both approaches**

Use environment variable if set, otherwise prompt for input. Use the same file structure as Option A:

**`.buildkite/pipeline.yml`**

```yaml
steps:
  # Only show input if WORKFLOW_TYPE not already set
  - input: "Select workflow"
    key: "workflow-select"
    if: build.env("WORKFLOW_TYPE") == null
    fields:
      - select: "Workflow type"
        key: "workflow"
        default: "build-and-release"
        options:
          - label: "Build and Release"
            value: "build-and-release"
          - label: "Scan Docker Images"
            value: "scan-docker-images"
          - label: "QA Release"
            value: "qa-release"

  - label: ":pipeline: Run workflow"
    depends_on: "workflow-select"
    command: |
      # Use env var if set, otherwise get from input metadata
      if [[ -n "$${WORKFLOW_TYPE}" ]]; then
        WORKFLOW="$${WORKFLOW_TYPE}"
      else
        WORKFLOW=$(buildkite-agent meta-data get workflow)
      fi
      buildkite-agent pipeline upload ".buildkite/workflows/$${WORKFLOW}.yml"
```

---

**Key points:**
- No direct equivalent to workflow-level `when` conditions—use dynamic pipeline upload instead
- Create separate YAML files for each workflow mode in `.buildkite/workflows/`
- Use `input` steps to collect user selection interactively
- Use environment variables for API/scheduled triggers
- Input step data is stored in build metadata, retrieved with `buildkite-agent meta-data get`

---

### Principle 15: Branch Filter Ignore Patterns

CircleCI supports `branches.ignore` to exclude specific branches from running a job, using exact names or regex patterns. It also supports a common pattern of combining `tags.only` with `branches.ignore: /.*/` to run jobs only on tag pushes.

```yaml
# CircleCI - ignore specific branches
workflows:
  main:
    jobs:
      - deploy:
          filters:
            branches:
              ignore:
                - dev
                - staging

# CircleCI - ignore branches matching pattern
workflows:
  main:
    jobs:
      - test:
          filters:
            branches:
              ignore:
                - /^feature\/experimental-.*/

# CircleCI - run only on tags (ignore all branches)
workflows:
  release:
    jobs:
      - publish:
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
```

**Translation approach:** Buildkite offers two levels of branch filtering:

1. **Pipeline-level filtering (UI)**: Configured in Pipeline Settings, controls whether a build is created at all
2. **Step-level filtering (YAML)**: Controls whether individual steps run within a build

#### Step-level: Using `branches:` attribute

For simple ignore patterns, use the `branches:` attribute with `!` prefix for negation:

```yaml
# Buildkite - ignore specific branches
steps:
  - label: ":rocket: Deploy"
    command: "make deploy"
    branches: "!dev !staging"
```

You can combine positive and negative patterns. All positive patterns are OR'd, and all negative patterns must not match:

```yaml
# Buildkite - run on main and release/*, but never on release/beta
steps:
  - label: ":test_tube: Test"
    command: "make test"
    branches: "main release/* !release/beta"
```

#### Step-level: Using `if:` conditionals

For complex patterns or when you need regex, use `if:` conditionals. Note that `branches:` and `if:` **cannot be used together** on the same step.

```yaml
# Buildkite - ignore branches matching regex pattern
steps:
  - label: ":test_tube: Test"
    command: "make test"
    if: build.branch !~ /^feature\/experimental-/
```

#### Tag-only builds

For the CircleCI pattern of running only on tags (ignoring all branches), use `build.tag`:

```yaml
# Buildkite - run only on version tags
steps:
  - label: ":package: Publish"
    command: "make publish"
    if: build.tag =~ /^v/
```

To check that a tag exists (any tag):

```yaml
# Buildkite - run only when a tag is present
steps:
  - label: ":package: Publish"
    command: "make publish"
    if: build.tag != null
```

#### Pipeline-level filtering (UI configuration)

For pipeline-wide branch restrictions that should prevent builds from being created entirely, configure this in the **Pipeline Settings UI**, not in YAML:

1. Go to Pipeline → Settings
2. Under "Branch Limiting", set the branch pattern (e.g., `main release/*` or `!experimental/*`)
3. Commits to non-matching branches will not create builds

This is equivalent to having `branches.only` or `branches.ignore` at the workflow level in CircleCI.

**Key points:**
- Simple ignore patterns → use `branches:` with `!` prefix (e.g., `branches: "!dev !staging"`)
- Complex patterns or regex → use `if:` with `build.branch !~` operator
- Tag-only jobs → use `if: build.tag =~ /pattern/` or `if: build.tag != null`
- `branches:` and `if:` cannot be used together on the same step
- Pipeline-wide filtering is configured in the UI, not YAML
- Branch patterns support `*` wildcard and `!` negation

---

## Detailed Translation Rules

### Job Dependencies

```yaml
# CircleCI
workflows:
  main:
    jobs:
      - build
      - test:
          requires:
            - build
      - deploy:
          requires:
            - build
            - test

# Buildkite
steps:
  - label: "build"
    key: "build"
    command: npm run build

  - label: "test"
    key: "test"
    depends_on: "build"
    command: npm test

  - label: "deploy"
    depends_on:
      - "build"
      - "test"
    command: npm run deploy
```

---

### Conditionals (Branch Filtering)

See [Principle 15: Branch Filter Ignore Patterns](#principle-15-branch-filter-ignore-patterns) for comprehensive coverage of branch filtering, including `branches.only`, `branches.ignore`, tag filtering, and the `branches:` vs `if:` attributes.

---

### Artifacts

```yaml
# CircleCI
jobs:
  build:
    steps:
      - run: npm run build
      - store_artifacts:
          path: dist/
          destination: build-output

# Buildkite
steps:
  - label: "build"
    command:
      - npm run build
    artifact_paths:
      - "dist/**/*"
```

---

### Environment Variables

```yaml
# CircleCI
jobs:
  build:
    environment:
      NODE_ENV: production
      API_URL: https://api.example.com
    steps:
      - run: npm run build

# Buildkite
steps:
  - label: "build"
    env:
      NODE_ENV: production
      API_URL: https://api.example.com
    command: npm run build
```

**Pipeline-level environment:**

```yaml
# CircleCI (via context or project settings)
# Contexts are configured in CircleCI UI

# Buildkite - use top-level env block
env:
  NODE_ENV: production

steps:
  - label: "build"
    command: npm run build
```

---

### Approval Jobs

```yaml
# CircleCI
workflows:
  deploy:
    jobs:
      - build
      - hold:
          type: approval
          requires:
            - build
      - deploy:
          requires:
            - hold

# Buildkite
steps:
  - label: "build"
    key: "build"
    command: npm run build

  - block: ":rocket: Deploy to production?"
    key: "hold"
    depends_on: "build"

  - label: "deploy"
    depends_on: "hold"
    command: npm run deploy
```

---

### Resource Classes and Parallelism

```yaml
# CircleCI
jobs:
  test:
    resource_class: large
    parallelism: 4
    steps:
      - run: npm test

# Buildkite
steps:
  - label: "test"
    agents:
      queue: "large"
    parallelism: 4
    command: npm test
```

**Note:** Buildkite parallelism creates multiple jobs with `BUILDKITE_PARALLEL_JOB` and `BUILDKITE_PARALLEL_JOB_COUNT` environment variables. Use these for manual test distribution if not using Test Engine.

---

## Plugin Reference

| Purpose | Plugin |
|---------|--------|
| Docker containers | `docker#` |
| Docker Compose | `docker-compose#` |
| Caching | `cache#` |
| Docker registry login | `docker-login#` |
| Slack notifications | `slack-notification#` |

---

## Translation Checklist

- [ ] Remove `version:` key
- [ ] Remove `setup: true` and translate continuation pattern
- [ ] Remove `orbs:` block and translate orb commands
- [ ] Remove `executors:` block and add Docker plugin per-step
- [ ] Remove `commands:` block and use dynamic pipelines or YAML anchors
- [ ] Flatten workflows to steps with `depends_on`
- [ ] Remove `checkout` step
- [ ] Convert `requires` to `depends_on`
- [ ] Convert branch/tag filters to `if:` conditionals
- [ ] Convert `store_artifacts` to `artifact_paths`
- [ ] Convert `persist_to_workspace`/`attach_workspace` to artifact upload/download
- [ ] Convert `save_cache`/`restore_cache` to cache plugin
- [ ] Document scheduled workflows for UI configuration
- [ ] Document agent requirements for Windows/macOS jobs
- [ ] Add comments for test splitting (requires Test Engine)
- [ ] Convert approval jobs to `block` steps
