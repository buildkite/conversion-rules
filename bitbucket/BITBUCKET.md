# Bitbucket Pipelines to Buildkite Translation Rules

This document provides rules and patterns for converting Bitbucket Pipelines to Buildkite pipelines.

## Terminology

| Bitbucket Pipelines | Buildkite |
|---------------------|-----------|
| Pipeline | Pipeline |
| Step | Step (command step) |
| `image` | Docker plugin |
| `script` | `command` |
| `parallel` | Parallel group or steps without `depends_on` |
| `caches` | Cache plugin |
| `artifacts` | `artifact_paths` / `buildkite-agent artifact` |
| `deployment` | Deployment target / concurrency group |
| `condition.changesets` | Diff plugin or conditional |
| `size: 2x` | Agent queue with appropriate resources |
| `definitions.steps` | YAML anchors in `common:` section |
| `pipelines.custom` | Block step / triggered pipeline |

## Universal Rules

These rules apply to every Bitbucket-to-Buildkite translation:

1. **Reference TRANSLATION_RULES_GENERAL.md** - Always follow the general Buildkite pipeline best practices
2. **Do not specify plugin versions** - Omit version numbers to always use latest (e.g., `docker#:` not `docker#v5.13.0:`)
3. **Convert `script` to `command`** - Bitbucket's `script` array becomes Buildkite's `command` array
4. **Move global `image` to steps** - Bitbucket's top-level `image` must be applied per-step via Docker plugin
5. **Preserve YAML anchors** - Bitbucket's `definitions.steps` pattern maps well to Buildkite YAML anchors

## Quick Mappings

| Bitbucket | Buildkite |
|-----------|-----------|
| `name:` | `label:` |
| `script:` | `command:` |
| `image:` (global) | `plugins: - docker#:` (per step) |
| `size: 2x` | Agent queue or `resource_class` |
| `max-time:` | `timeout_in_minutes:` |
| `artifacts: download: true` | `depends_on:` + artifact plugin |
| `caches:` | Cache plugin |
| `parallel:` | Steps in same group without `depends_on` |
| `condition.changesets.includePaths` | Diff plugin |

---

## Translation Rules

### [BB-001] Global Image to Docker Plugin

Bitbucket allows a top-level `image:` that applies to all steps. Buildkite has no global image setting—use the Docker plugin on each step instead.

**Bitbucket:**
```yaml
image: node:20

pipelines:
  branches:
    main:
      - step:
          name: Build
          script:
            - npm install
            - npm run build
      - step:
          name: Test
          script:
            - npm test
```

**Buildkite (explicit per step):**
```yaml
steps:
  - label: "Build"
    command:
      - npm install
      - npm run build
    plugins:
      - docker#:
          image: node:20

  - label: "Test"
    command:
      - npm test
    plugins:
      - docker#:
          image: node:20
```

**Buildkite (YAML anchor to avoid repetition):**
```yaml
common:
  - docker_plugin: &docker
      docker#:
        image: node:20

steps:
  - label: "Build"
    command:
      - npm install
      - npm run build
    plugins:
      - *docker

  - label: "Test"
    command:
      - npm test
    plugins:
      - *docker
```

**Key points:**
- The `common:` key is ignored by Buildkite but allows defining reusable YAML anchors
- No version number on `docker#:` per general translation rules
- If steps need different images, specify different Docker plugin configurations per step

---

### [BB-002] Branch-Specific Pipelines

Bitbucket uses `pipelines.branches` to define completely different step lists for different branches. In Buildkite, combine all steps into one pipeline and use `branches:` filtering on individual steps.

> **Note:** If branch logic becomes very complex, consider splitting into separate pipeline YAML files and using `buildkite-agent pipeline upload` to dynamically select the appropriate configuration.

**Bitbucket:**
```yaml
pipelines:
  branches:
    main:
      - step:
          name: Deploy to production
          script:
            - ./deploy.sh prod
    release-*:
      - parallel:
          - step: *Unit-tests
          - step: *Integration-tests
      - step:
          name: Deploy to staging
          script:
            - ./deploy.sh staging
    develop:
      - step: *Unit-tests
```

**Buildkite:**
```yaml
steps:
  # Main branch only
  - label: "Deploy to production"
    command: "./deploy.sh prod"
    branches: "main"

  # Release branches - parallel tests (no depends_on = parallel)
  - label: "Unit tests"
    key: "unit-tests"
    command: "make unit-test"
    branches: "release-*"

  - label: "Integration tests"
    key: "integration-tests"
    command: "make integration-test"
    branches: "release-*"

  - label: "Deploy to staging"
    command: "./deploy.sh staging"
    depends_on:
      - "unit-tests"
      - "integration-tests"
    branches: "release-*"

  # Develop branch
  - label: "Unit tests"
    command: "make unit-test"
    branches: "develop"
```

**Key points:**
- Use `branches:` attribute on individual command steps to filter by branch
- Branch patterns support wildcards: `release-*`, `feature/*`
- Use `!` prefix to exclude branches: `branches: "main !main-legacy"`
- Group steps do not support `branches:`—use `if: build.branch =~ /pattern/` on groups instead
- Steps without `depends_on` run in parallel automatically

---

### [BB-003] Pull Request Pipelines

Bitbucket uses `pipelines.pull-requests` to define steps that run only on pull requests. In Buildkite, PR triggers are configured at the pipeline level rather than in the YAML.

**Bitbucket:**
```yaml
pipelines:
  pull-requests:
    "**":
      - step:
          name: Test
          script:
            - npm test
```

**Buildkite:**

Configure PR builds in the Buildkite pipeline settings:
1. Go to **Pipeline Settings → GitHub/Bitbucket/GitLab**
2. Enable **"Build pull requests"** 
3. Optionally enable **"Skip builds for existing commits"** to avoid duplicate builds

The steps defined in your pipeline YAML will run for both branch pushes and PRs (unless filtered with `branches:` or `if:` conditionals). If you need PR-only steps, use:

```yaml
steps:
  - label: "PR-only tests"
    command: "npm run test:pr"
    if: build.pull_request.id != null
```

**Key points:**
- PR trigger configuration lives in pipeline settings, not YAML
- Use `if: build.pull_request.id != null` for PR-only steps
- Use `if: build.pull_request.id == null` for branch-only steps (skip on PRs)

---

### [BB-004] Basic Step Definition

Bitbucket steps use `name:` and `script:`. In Buildkite, these map to `label:` and `command:`.

**Bitbucket:**
```yaml
- step:
    name: Build application
    script:
      - npm install
      - npm run build
```

**Buildkite:**
```yaml
- label: "Build application"
  command:
    - npm install
    - npm run build
```

**Key points:**
- `name:` becomes `label:` (displayed in the Buildkite UI)
- `script:` becomes `command:` (array of shell commands)
- Buildkite steps don't need a `step:` wrapper—each list item is a step
- Add `key:` if other steps need to reference this step via `depends_on:`

---

### [BB-005] YAML Anchors for Reusable Steps

Bitbucket defines reusable steps in `definitions.steps` using YAML anchors. Buildkite also supports YAML anchors—use a `common:` section (ignored by Buildkite) to define them.

**Bitbucket:**
```yaml
definitions:
  steps:
    - step: &build-step
        name: Build
        script:
          - npm run build
    - step: &test-step
        name: Test
        script:
          - npm test

pipelines:
  branches:
    main:
      - step: *build-step
      - step: *test-step
    develop:
      - step: *build-step
```

**Buildkite:**
```yaml
common:
  - build_step: &build-step
      label: "Build"
      command:
        - npm run build
  - test_step: &test-step
      label: "Test"
      command:
        - npm test

steps:
  # Main branch
  - <<: *build-step
    branches: "main"
  - <<: *test-step
    branches: "main"

  # Develop branch
  - <<: *build-step
    branches: "develop"
```

**Key points:**
- `definitions.steps` becomes a `common:` section (Buildkite ignores unknown top-level keys)
- Anchor syntax (`&name` / `*name`) works identically in both systems
- Use `<<: *anchor` to merge an anchor and add/override properties (like `branches:`)
- Direct `*anchor` reference works when no additional properties are needed

---

### [BB-006] Parallel Step Execution

Bitbucket uses explicit `parallel:` blocks. In Buildkite, steps without `depends_on:` run in parallel by default—no special syntax needed.

**Bitbucket:**
```yaml
- parallel:
    - step:
        name: Unit tests
        script:
          - npm run test:unit
    - step:
        name: Integration tests
        script:
          - npm run test:integration
- step:
    name: Deploy
    script:
      - ./deploy.sh
```

**Buildkite:**
```yaml
steps:
  # These run in parallel (no depends_on)
  - label: ":test_tube: Unit tests"
    key: "unit-tests"
    command: "npm run test:unit"

  - label: ":test_tube: Integration tests"
    key: "integration-tests"
    command: "npm run test:integration"

  # This waits for the parallel steps
  - label: ":rocket: Deploy"
    command: "./deploy.sh"
    depends_on:
      - "unit-tests"
      - "integration-tests"
```

**With group (for visual organization):**
```yaml
steps:
  - group: ":test_tube: Tests"
    key: "tests"
    steps:
      - label: "Unit tests"
        command: "npm run test:unit"
      - label: "Integration tests"
        command: "npm run test:integration"

  - label: ":rocket: Deploy"
    command: "./deploy.sh"
    depends_on: "tests"
```

**Key points:**
- Buildkite steps run in parallel by default—no `parallel:` keyword needed
- Use `depends_on:` to create sequential dependencies
- Use `group:` to visually organize related parallel steps in the UI
- Only use groups when they contain 2+ steps (per general translation rules)

---

### [BB-007] Caching Dependencies

Bitbucket defines caches in `definitions.caches` and references them by name. Bitbucket manages cache storage automatically. In Buildkite, caching requires additional setup depending on your agent type.

**Bitbucket:**
```yaml
definitions:
  caches:
    node-modules: ta-client/node_modules
    server-node: ta-server/node_modules

pipelines:
  default:
    - step:
        name: Build client
        caches:
          - node-modules
        script:
          - cd ta-client && npm install && npm run build
```

**Buildkite (comment-only approach for out-of-box compatibility):**
```yaml
steps:
  - label: "Build client"
    # TODO: Configure caching for node_modules
    # Bitbucket cached: node-modules (ta-client/node_modules)
    # Options:
    #   - Hosted agents: Enable container caching in cluster settings
    #   - Self-hosted: Use cache# plugin with S3/GCS backend
    #     Example:
    #     plugins:
    #       - cache#:
    #           path: ta-client/node_modules
    #           key: "node-{{ checksum 'ta-client/package-lock.json' }}"
    command:
      - cd ta-client && npm install && npm run build
```

**Key points:**
- Bitbucket manages cache storage automatically; Buildkite requires setup
- **Hosted agents**: Enable "container caching" or "Git mirror" at the cluster level (UI setting, not YAML)
- **Self-hosted agents**: Use the `cache#` plugin with an S3/GCS backend configured
- Translation preserves cache intent as comments to avoid broken pipelines
- **Key-based caching**: When the cache plugin is configured, use `key:` with checksums for smarter invalidation than Bitbucket's simple name-only approach:
  ```yaml
  plugins:
    - cache#:
        path: node_modules
        key: "node-{{ checksum 'package-lock.json' }}"
  ```
  This automatically invalidates the cache when `package-lock.json` changes.

**Common Bitbucket built-in caches (for reference in comments):**
| Bitbucket | Path |
|-----------|------|
| `node` | `node_modules` |
| `pip` | `~/.cache/pip` |
| `maven` | `~/.m2/repository` |
| `gradle` | `~/.gradle/caches` |

---

### [BB-008] Step Resource Sizing

Bitbucket uses `size: 2x` to allocate more resources to a step. In Buildkite, resource allocation depends on your agent infrastructure.

**Bitbucket:**
```yaml
- step:
    size: 2x
    name: Heavy build
    script:
      - npm run build
```

**Buildkite (comment-only approach for out-of-box compatibility):**
```yaml
steps:
  - label: "Heavy build"
    # TODO: Configure agent queue for increased resources
    # Bitbucket used: size: 2x (4GB RAM, 2 vCPUs)
    # Options:
    #   - Hosted agents: Use a larger instance type queue
    #   - Self-hosted: Target a queue with appropriately sized agents
    #     Example:
    #     agents:
    #       queue: "large"
    command: "npm run build"
```

**Key points:**
- Bitbucket `size: 1x` = 4GB RAM, 2 vCPUs (default)
- Bitbucket `size: 2x` = 8GB RAM, 4 vCPUs
- Translation preserves sizing intent as comments
- Actual queue names depend on customer's Buildkite agent setup

---

### [BB-009] Path-Based Conditional Execution

Bitbucket uses `condition.changesets.includePaths` to run steps only when certain files change. Buildkite has a native `if_changed` attribute that works out of the box.

> ⚠️ **IMPORTANT: Always use native Buildkite features over plugins when available.**
> 
> `if_changed` is a **native Buildkite attribute** - do NOT use third-party plugins like `monorepo-diff`, `pathwatcher`, or similar for basic path-based conditional execution. Native features are more reliable, better supported, and require no plugin installation.

> 🚧 **CRITICAL LIMITATION: Agent-Applied Attribute**
> 
> `if_changed` is an **agent-applied attribute** that is ONLY processed by `buildkite-agent pipeline upload`. It is **NOT accepted** in:
> - Pipelines configured via the Buildkite web UI
> - Pipelines set via the API's `configuration` field
> 
> **The converted YAML must be stored in the repository** (e.g., `.buildkite/pipeline.yml`) and loaded dynamically. The pipeline's initial configuration should be:
> ```yaml
> steps:
>   - label: ":pipeline: Upload pipeline"
>     command: buildkite-agent pipeline upload .buildkite/pipeline.yml
> ```

**Bitbucket:**
```yaml
- step:
    name: Build client
    condition:
      changesets:
        includePaths:
          - "ta-client/**"
          - "packages/**"
    script:
      - cd ta-client && npm run build
```

**Buildkite:**
```yaml
steps:
  - label: "Build client"
    if_changed:
      - "ta-client/**"
      - "packages/**"
    command:
      - cd ta-client && npm run build
```

**Key points:**
- `if_changed` is a **native** Buildkite feature
- Supports single glob pattern or list of patterns
- Uses Buildkite's glob syntax (e.g., `**` for recursive matching)
- Supports `include` and `exclude` for more complex matching:

```yaml
steps:
  - label: "Run for spec/ but not spec/integration/"
    if_changed:
      include: "spec/**"
      exclude: "spec/integration/**"
    command: "npm test"
```

**❌ DO NOT use plugins for this:**
```yaml
# WRONG - unnecessary plugin when native feature exists
steps:
  - label: "Build client"
    plugins:
      - monorepo-diff#v1.0.0:
          watch:
            - path: "ta-client/**"
```

**Glob pattern differences:**
| Bitbucket | Buildkite |
|-----------|-----------|
| `ta-client/**` | `ta-client/**` (same) |
| `**/*.go` | `**.go` (Buildkite uses `**` without `/`) |

---

### [BB-010] Service Containers

Bitbucket `services:` runs sidecar containers. Convert database services to docker-compose plugin:

**Bitbucket:**
```yaml
definitions:
  services:
    mysql:
      image: mysql:5.7
      environment:
        MYSQL_DATABASE: test_db
        MYSQL_ROOT_PASSWORD: password

pipelines:
  default:
    - step:
        name: Integration tests
        services:
          - mysql
        script:
          - npm run test:integration
```

**Buildkite:**
```yaml
steps:
  - label: "Integration tests"
    plugins:
      - docker-compose#:
          run: app
          config: docker-compose.test.yml
    command:
      - npm run test:integration
```

With `docker-compose.test.yml`:
```yaml
services:
  app:
    build: .
    depends_on: [mysql]
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: test_db
      MYSQL_ROOT_PASSWORD: password
```

For `services: [docker]` (Docker-in-Docker): Hosted agents have Docker by default; self-hosted requires agent configuration

---

### [BB-011] Manually-Triggered Pipelines (Custom Pipelines)

Bitbucket's `pipelines.custom` defines pipelines that only run when manually triggered.

**Bitbucket:**
```yaml
pipelines:
  custom:
    deploy-to-qa:
      - step:
          name: Deploy to QA
          script:
            - ./deploy.sh qa
```

**Buildkite Option 1: Separate Pipeline**

Create a separate pipeline and disable automatic triggers in Pipeline Settings → GitHub Settings (uncheck "Build pushes" / "Build pull requests"). Trigger via UI, API, or trigger step.

**Buildkite Option 2: Conditional Steps**

Use `if:` conditionals to skip steps on automated builds:

```yaml
steps:
  - label: "Deploy to QA"
    command: "./deploy.sh qa"
    if: build.source == "ui" || build.source == "api" || build.source == "trigger_job"
```

**Useful conditionals:**
- `build.source` - `"webhook"`, `"ui"`, `"api"`, `"trigger_job"`, `"schedule"`
- `build.pull_request.id == null` - Skip on PR builds

---

### [BB-013] Global Timeout Configuration

Bitbucket's `options.max-time` sets a global timeout for all steps. In Buildkite, there is no pipeline-level timeout in YAML, but you can configure default timeouts in the UI or apply per-step.

**Bitbucket:**
```yaml
options:
  max-time: 40

pipelines:
  default:
    - step:
        name: Build
        script:
          - npm run build
    - step:
        name: Test
        script:
          - npm test
```

**Buildkite Option 1: Pipeline Default Timeout (Recommended)**

Configure in Pipeline Settings → Steps → "Default Command Step Timeout":
- Set to the desired timeout in minutes (e.g., 40)
- This applies to all steps without an explicit `timeout_in_minutes`

```yaml
# No timeout needed in YAML - uses pipeline default
steps:
  - label: "Build"
    command: "npm run build"

  - label: "Test"
    command: "npm test"
```

**Buildkite Option 2: Per-Step with YAML Anchor**

If you need the timeout in YAML (e.g., for version control), use a YAML anchor:

```yaml
common:
  - timeout: &default-timeout
      timeout_in_minutes: 40

steps:
  - label: "Build"
    command: "npm run build"
    <<: *default-timeout

  - label: "Test"
    command: "npm test"
    <<: *default-timeout
```

**Key points:**
- **Pipeline default** (Option 1) is cleaner and matches Bitbucket's global behavior
- **YAML anchor** (Option 2) keeps configuration in version control
- Organization-level defaults can also be set in Organization Settings
- Individual steps can override the default with their own `timeout_in_minutes`

---

### [BB-012] Deployment Environments

Bitbucket's `deployment:` tags a step with an environment name for tracking and provides deployment history. In Buildkite, use `concurrency_group` to serialize deployments and metadata for tracking.

**Bitbucket:**
```yaml
pipelines:
  branches:
    main:
      - step:
          name: Deploy to staging
          deployment: staging
          script:
            - ./deploy.sh staging
      - step:
          name: Deploy to production
          deployment: production
          trigger: manual
          script:
            - ./deploy.sh production
```

**Buildkite:**
```yaml
steps:
  - label: "Deploy to staging"
    command: "./deploy.sh staging"
    branches: "main"
    concurrency: 1
    concurrency_group: "deploy-staging"
    env:
      DEPLOYMENT_ENV: staging

  - block: "Deploy to production?"
    branches: "main"

  - label: "Deploy to production"
    command: "./deploy.sh production"
    branches: "main"
    concurrency: 1
    concurrency_group: "deploy-production"
    env:
      DEPLOYMENT_ENV: production
```

**Key points:**
- **`concurrency: 1` + `concurrency_group`**: Ensures only one deployment to each environment at a time
- **`block:` step**: Replaces Bitbucket's `trigger: manual` for requiring manual approval
- **Environment tracking**: Use `env:` or `meta-data` to track which environment is being deployed
- **GitHub Deployments**: If using GitHub, enable "Trigger builds on deployment" in pipeline settings and use `buildkite-agent meta-data set` to report deployment status back to GitHub

---

### [BB-014] Artifacts - Passing Files Between Steps

Bitbucket uses `artifacts:` to pass files from one step to subsequent steps. In Buildkite, use `artifact_paths` to upload and `buildkite-agent artifact download` to retrieve.

**Bitbucket:**
```yaml
pipelines:
  default:
    - step:
        name: Build
        script:
          - npm run build
        artifacts:
          - dist/**
          - package.json

    - step:
        name: Deploy
        script:
          - ./deploy.sh
```

**Buildkite:**
```yaml
steps:
  - label: "Build"
    key: "build"
    command:
      - npm run build
    artifact_paths:
      - "dist/**"
      - "package.json"

  - label: "Deploy"
    depends_on: "build"
    command:
      - buildkite-agent artifact download "dist/**" .
      - buildkite-agent artifact download "package.json" .
      - ./deploy.sh
```

**Key points:**
- **Upload**: Use `artifact_paths:` on the step that creates the files
- **Download**: Use `buildkite-agent artifact download "<pattern>" <destination>` in subsequent steps
- **Explicit download required**: Unlike Bitbucket, Buildkite does not automatically make artifacts available - you must explicitly download them
- **Glob patterns**: Both platforms support glob patterns (`**`, `*`)
- **`depends_on`**: Ensure the downloading step waits for the uploading step to complete

**Download to specific directory:**
```yaml
- label: "Deploy"
  depends_on: "build"
  command:
    - mkdir -p build
    - buildkite-agent artifact download "dist/**" build/
    - ./deploy.sh build/dist/
```

---

### [BB-015] Pipeline Variables

Bitbucket supports variables at multiple levels (repository, deployment, pipeline) with optional user prompts. In Buildkite, use `env:` for static variables and `input` steps for user-selectable values.

**Bitbucket (static variables):**
```yaml
pipelines:
  default:
    - step:
        name: Build
        script:
          - echo "Building for $ENVIRONMENT"
          - ./build.sh
```
*(Variables defined in Bitbucket Repository Settings → Pipelines → Variables)*

**Buildkite (static variables):**
```yaml
# Top-level env applies to all steps
env:
  ENVIRONMENT: staging
  NODE_VERSION: "20"

steps:
  - label: "Build"
    command:
      - echo "Building for $ENVIRONMENT"
      - ./build.sh

  - label: "Test"
    # Per-step env overrides or adds to top-level
    env:
      ENVIRONMENT: test
    command:
      - ./test.sh
```

**Bitbucket (user-prompted variables):**
```yaml
pipelines:
  custom:
    deploy:
      - variables:
          - name: Environment
            default: staging
            allowed-values:
              - staging
              - production
      - step:
          name: Deploy
          script:
            - ./deploy.sh $Environment
```

**Buildkite (user-prompted variables via input step):**
```yaml
steps:
  - input: "Configure deployment"
    fields:
      - select: "Environment"
        key: "deploy-environment"
        default: "staging"
        options:
          - label: "Staging"
            value: "staging"
          - label: "Production"
            value: "production"

  - label: "Deploy"
    command:
      - ./deploy.sh $$(buildkite-agent meta-data get "deploy-environment")
```

**Key points:**
- **Top-level `env:`**: Applies to all steps in the pipeline
- **Per-step `env:`**: Overrides or extends top-level variables for that step
- **Repository-level secrets**: Configure in Buildkite Pipeline Settings → Environment Variables (can be marked as secret)
- **User prompts**: Use `input` steps with `fields:` for runtime variable selection
- **Accessing input values**: Use `buildkite-agent meta-data get "<key>"` to retrieve values from input steps

---

### [BB-016] Clone Settings

Bitbucket allows configuring clone behavior (depth, LFS, skip). In Buildkite, this is typically handled by agent configuration or the checkout plugin.

**Bitbucket:**
```yaml
clone:
  depth: 1        # Shallow clone
  lfs: true       # Enable Git LFS
  enabled: false  # Skip cloning entirely

pipelines:
  default:
    - step:
        name: Build
        script:
          - npm run build
```

**Buildkite:**
```yaml
steps:
  - label: "Build"
    # TODO: Configure clone settings
    # Bitbucket used: depth: 1, lfs: true
    # Options:
    #   - Use custom-checkout plugin:
    #     plugins:
    #       - custom-checkout#:
    #           clone_depth: 1
    #           lfs: true
    #   - Configure via agent environment variables:
    #     BUILDKITE_GIT_CLONE_FLAGS="--depth=1"
    #   - To skip checkout entirely:
    #     skip_checkout: true
    command:
      - npm run build
```

**Key points:**
- **Shallow clones**: Use `custom-checkout#` plugin with `clone_depth` or set `BUILDKITE_GIT_CLONE_FLAGS` agent env var
- **Git LFS**: Configure via `custom-checkout#` plugin or ensure `git-lfs` is installed on agents
- **Skip clone**: Use `skip_checkout: true` on the step
- **Agent-level config**: Clone settings can also be configured at the agent level via environment variables

---

### [BB-017] Fail-Fast for Parallel Steps

Bitbucket's `fail-fast` on parallel blocks cancels remaining parallel steps when one fails. In Buildkite, use `cancel_on_build_failing: true` on each step.

**Bitbucket:**
```yaml
- parallel:
    fail-fast: true
    steps:
      - step:
          name: Test 1
          script:
            - npm run test:1
      - step:
          name: Test 2
          script:
            - npm run test:2
      - step:
          name: Test 3
          script:
            - npm run test:3
```

**Buildkite:**
```yaml
steps:
  - label: "Test 1"
    command: "npm run test:1"
    cancel_on_build_failing: true

  - label: "Test 2"
    command: "npm run test:2"
    cancel_on_build_failing: true

  - label: "Test 3"
    command: "npm run test:3"
    cancel_on_build_failing: true
```

**Key points:**
- **`cancel_on_build_failing: true`**: Cancels this job if the build enters a failing state
- **Per-step attribute**: Must be added to each step that should be cancelled on failure
- **YAML anchor**: Use an anchor to avoid repetition when many steps need this attribute
- **`soft_fail` steps excluded**: Steps marked with `soft_fail` won't trigger cancellation of other steps

**With YAML anchor to reduce repetition:**
```yaml
common:
  - fast_fail: &fast-fail
      cancel_on_build_failing: true

steps:
  - label: "Test 1"
    command: "npm run test:1"
    <<: *fast-fail

  - label: "Test 2"
    command: "npm run test:2"
    <<: *fast-fail

  - label: "Test 3"
    command: "npm run test:3"
    <<: *fast-fail
```

---

### [BB-018] After-Script (Cleanup Commands)

Bitbucket's `after-script` runs after the main script regardless of success/failure.

**Bitbucket:**
```yaml
- step:
    name: Build
    script:
      - npm install
      - npm run build
    after-script:
      - echo "Cleaning up..."
      - rm -rf temp/
```

**Buildkite Option 1 - Shell `trap` (per-step):**
```yaml
steps:
  - label: "Build"
    command: |
      cleanup() { echo "Cleaning up..."; rm -rf temp/; }
      trap cleanup EXIT
      npm install
      npm run build
```

**Buildkite Option 2 - Repository `post-command` hook:**

For cleanup that runs after every step, create `.buildkite/hooks/post-command`:
```bash
#!/bin/bash
echo "Cleaning up..."
rm -rf temp/
```

**Key points:**
- **`trap ... EXIT`**: Best for step-specific cleanup; runs on exit regardless of success/failure
- **Repository hook**: Best for consistent cleanup across all steps in a repo
- Agent-level hooks also available for cleanup across all repos on an agent

---

## Recommended Plugins

| Plugin | Purpose |
|--------|---------|
| `docker#` | Run steps in Docker containers |
| `cache#` | Cache dependencies between builds |
| `artifacts#` | Upload/download artifacts |
| `diff#` | Conditional execution based on changed files |
| `docker-compose#` | Multi-container environments |

---

## Translation Checklist

- [ ] Converted global `image:` to Docker plugin on each step (or YAML anchor)
- [ ] Converted `script:` arrays to `command:` arrays
- [ ] Converted `name:` to `label:`
- [ ] Handled `pipelines.branches` branch filtering
- [ ] Handled `pipelines.pull-requests` triggers
- [ ] Converted `parallel:` blocks appropriately
- [ ] Converted `caches:` to cache plugin
- [ ] Converted `artifacts:` to artifact handling
- [ ] Converted `condition.changesets` to diff plugin or conditional
- [ ] Converted `size:` to appropriate agent queue
- [ ] Converted `max-time:` to `timeout_in_minutes:`
- [ ] Converted `deployment:` to deployment targets
- [ ] Verified no plugin version numbers are specified
