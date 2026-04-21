# Day 46 – Reusable Workflows & Composite Actions

## 📌 Task 1: Theory

### What is a reusable workflow?

A reusable workflow is a GitHub Actions workflow that can be called by other workflows using `workflow_call`. It helps avoid duplication and improves maintainability.

### What is the `workflow_call` trigger?

`workflow_call` allows one workflow to be triggered by another workflow.

### Reusable workflow vs regular action (`uses:`)

* Reusable workflow → used at **job level**
* Action → used at **step level**

### Where must it live?

Reusable workflows must be inside:

```
.github/workflows/
```

---

## 📌 Task 2: Reusable Workflow

```yaml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      app_name:
        required: true
        type: string
      environment:
        required: true
        type: string
        default: staging
    secrets:
      docker_token:
        required: true
    outputs:
      build_version:
        description: "Generated build version"
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.set_version.outputs.version }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Print Info
        run: |
          echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"
          echo "Docker token is set: ${{ secrets.docker_token != '' }}"

      - name: Generate Version
        id: set_version
        run: |
          VERSION="v1.0-${GITHUB_SHA::7}"
          echo "version=$VERSION" >> $GITHUB_OUTPUT
```

---

## 📌 Task 3: Caller Workflow

```yaml
name: Call Reusable Workflow

on:
  push:
    branches: [ main ]

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      app_name: "my-web-app"
      environment: "production"
    secrets:
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  print-version:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Show Version
        run: echo "Build version is ${{ needs.build.outputs.build_version }}"
```

---

## 📌 Task 4: Outputs Explanation

The reusable workflow generates a version:

```
v1.0-<short-sha>
```

This value is passed to the caller workflow using outputs and accessed via:

```
needs.build.outputs.build_version
```

---

## 📌 Task 5: Composite Action

```yaml
name: Setup and Greet
description: Custom composite action

inputs:
  name:
    required: true
  language:
    required: false
    default: "en"

outputs:
  greeted:
    description: "Greeting status"
    value: "true"

runs:
  using: "composite"
  steps:
    - name: Greet User
      shell: bash
      run: |
        if [ "${{ inputs.language }}" = "hi" ]; then
          echo "Namaste ${{ inputs.name }}"
        else
          echo "Hello ${{ inputs.name }}"
        fi

    - name: Show System Info
      shell: bash
      run: |
        echo "Date: $(date)"
        echo "Runner OS: $RUNNER_OS"
```

---

## 📌 Workflow Using Composite Action

```yaml
name: Use Composite Action

on:
  workflow_dispatch:

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Run Custom Action
        uses: ./.github/actions/setup-and-greet
        with:
          name: "Guru"
          language: "hi"
```

---

## 📌 Task 6: Comparison Table

| Feature            | Reusable Workflow  | Composite Action     |
| ------------------ | ------------------ | -------------------- |
| Triggered by       | workflow_call      | uses: (step level)   |
| Can contain jobs?  | Yes                | No                   |
| Can contain steps? | Yes                | Yes                  |
| Lives where?       | .github/workflows/ | .github/actions/     |
| Accept secrets?    | Yes                | Indirect             |
| Best for           | Full pipelines     | Small reusable logic |

---

## 📌 Screenshot

(Add a screenshot of your GitHub Actions run here showing the caller workflow triggering the reusable workflow)

---

## ✅ Conclusion

* Learned how to create reusable workflows using `workflow_call`
* Understood passing inputs, secrets, and outputs
* Built a custom composite action
* Compared reusable workflows vs composite actions

---
