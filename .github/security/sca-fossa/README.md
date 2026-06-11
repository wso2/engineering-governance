# FOSSA SCA Scan Action

Run FOSSA scans from GitHub Actions with support for:

- `fossa analyze`
- optional `fossa test`
- optional diff-based test gating for pull requests
- optional attribution report generation
- optional debug mode
- optional high/critical vulnerability gating through the FOSSA issues API

## Prerequisites

- A FOSSA API key stored as a GitHub secret such as `FOSSA_API_KEY`
- A Linux or macOS runner with network access to your FOSSA endpoint
- For the optional vulnerability gate, a FOSSA project scope convention that can be resolved from `scope-id-prefix`, `github-org`, and `repo-name`

## Usage

Use a pinned release tag or commit SHA when consuming this action from another repository.

### Basic scan

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
```

### Scan and test

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          run-tests: "true"
```

### Scan a specific folder in a repository

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          working-directory: apps/web
          run-tests: "true"
```

### Pull request diff gate

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          run-tests: ${{ github.event_name == 'pull_request' }}
          test-diff-revision: ${{ github.event.pull_request.base.sha }}
```

### Generate an attribution report

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - id: fossa
        uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          generate-report: html

      - run: echo '${{ steps.fossa.outputs.report }}' > report.html
```

### Enable the vulnerability API gate

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          run-tests: "true"
          fail-on-vulnerabilities: "true"
          scope-id-prefix: custom+28356/github.com
```

## Inputs

- `api-key`: Required FOSSA API key.
- `run-tests`: Optional. Runs `fossa test` after analysis when set to `"true"`. Default: `"false"`.
- `generate-report`: Optional. Runs `fossa report attribution --format <value>` and exposes the report content as an output.
- `test-diff-revision`: Optional. Passed to `fossa test --diff`. Effective only when `run-tests` is enabled.
- `branch`: Optional. Branch passed to FOSSA CLI. Defaults to the current GitHub ref.
- `project`: Optional. Project name passed to FOSSA CLI.
- `endpoint`: Optional. FOSSA endpoint URL. Default: `https://app.fossa.com`.
- `debug`: Optional. Runs FOSSA commands in debug mode when set to `"true"`. Default: `"false"`.
- `working-directory`: Optional. Directory to scan. Default: `.`.
- `fail-on-vulnerabilities`: Optional. Enables the extra FOSSA issues API gate for active `high` and `critical` vulnerabilities. Default: `"false"`.
- `repo-name`: Optional. Repository name used by the vulnerability API gate. Defaults to the current repository name.
- `github-org`: Optional. Repository owner used by the vulnerability API gate. Defaults to the current repository owner.
- `scope-id-prefix`: Optional. FOSSA scope prefix used by the vulnerability API gate. Required for that gate to run.

## Outputs

- `revision`: Revision extracted from `fossa analyze` output. Falls back to `GITHUB_SHA` if the revision cannot be parsed.
- `active-count`: Active high/critical vulnerability count returned by the FOSSA issues API when the vulnerability gate is enabled.
- `report`: Generated attribution report content when `generate-report` is used.

## Admin Integration Guide

For most repositories, use a single workflow and declare scan targets with a matrix instead of creating one workflow per path.

### Recommended model

- Keep one GitHub workflow for FOSSA scanning in the product repository.
- Define one matrix entry per independently scanable component.
- Pass both `working-directory` and `project` for each component.
- Use a stable project naming convention such as `repo/path` so each component is tracked as a separate FOSSA project.
- Run all components on `main`, and optionally allow manual selection with `workflow_dispatch`.

### Example for a monorepo

```yaml
jobs:
  fossa-scan:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        app:
          - name: nextjs
            path: apps/nextjs
          - name: go
            path: apps/go
          - name: api
            path: services/api
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          working-directory: ${{ matrix.app.path }}
          project: ${{ format('{0}/{1}', github.event.repository.name, matrix.app.path) }}
```

### Example with manual target selection and path-based PR or push scans

```yaml
name: FOSSA Scan

on:
  workflow_dispatch:
    inputs:
      target:
        description: App to scan
        required: false
        default: all
        type: choice
        options:
          - all
          - nextjs
          - go
  pull_request:
    paths:
      - 'apps/nextjs/**'
      - 'apps/go/**'
      - '.github/workflows/fossa-scan.yml'
  push:
    branches:
      - main
    paths:
      - 'apps/nextjs/**'
      - 'apps/go/**'
      - '.github/workflows/fossa-scan.yml'

jobs:
  detect-changes:
    if: github.event_name != 'workflow_dispatch'
    runs-on: ubuntu-latest
    outputs:
      nextjs: ${{ steps.filter.outputs.nextjs }}
      go: ${{ steps.filter.outputs.go }}
    steps:
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            nextjs:
              - 'apps/nextjs/**'
              - '.github/workflows/fossa-scan.yml'
            go:
              - 'apps/go/**'
              - '.github/workflows/fossa-scan.yml'

  fossa-scan:
    if: |
      github.event_name == 'workflow_dispatch' ||
      needs.detect-changes.outputs.nextjs == 'true' ||
      needs.detect-changes.outputs.go == 'true'
    needs:
      - detect-changes
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        app:
          - name: nextjs
            path: apps/nextjs
          - name: go
            path: apps/go
    steps:
      - uses: actions/checkout@v4

      - uses: wso2/engineering-governance/.github/security/sca-fossa@main
        if: |
          (github.event_name == 'workflow_dispatch' && (
            github.event.inputs.target == 'all' ||
            github.event.inputs.target == matrix.app.name
          )) ||
          (github.event_name != 'workflow_dispatch' && (
            (matrix.app.name == 'nextjs' && needs.detect-changes.outputs.nextjs == 'true') ||
            (matrix.app.name == 'go' && needs.detect-changes.outputs.go == 'true')
          ))
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}
          working-directory: ${{ matrix.app.path }}
          project: ${{ format('{0}/{1}', github.event.repository.name, matrix.app.path) }}
          run-tests: ${{ github.event_name == 'pull_request' }}
          test-diff-revision: ${{ github.event.pull_request.base.sha }}
```

### When to use separate workflows

Use separate workflows only when components need materially different:

- triggers
- permissions
- ownership
- policy or release cadence

If those differences do not exist, one workflow with a matrix is the recommended setup.

### Operational guidance

- `working-directory` controls what folder is scanned.
- `project` controls which FOSSA project receives the scan.
- Use both together when the repository contains multiple components.
- Use `workflow_dispatch` inputs for manual target selection.
- Use path filtering plus a change-detection job for automatic PR and push target selection.
- Keep the workflow model centralized even when scans are selective.

## Notes

- The vulnerability gate is a WSO2-specific extension on top of the standard FOSSA analyze/test flow.
- The action currently installs a repo-managed FOSSA CLI version: `v3.17.10`. Update the action itself when you want to roll forward the CLI version across consumers.
- The examples above still use `@main` for readability. For production use, pin to a release tag or commit SHA.
