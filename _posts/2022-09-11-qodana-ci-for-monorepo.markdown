---
layout: post
title:  "How to run Qodana CI in backend and frontend sub-folders of a monorepo"
date:   2022-09-11 11:52:00 +0200
categories: github-actions qodana
---

## Intro

Some time ago JetBrains announced the release of [Qodana](https://www.jetbrains.com/qodana/), the code quality platform
for CI.

I decided to use it in a project I'd been working on to control and guarantee code quality.

## Problem

All the documentation examples don't imply that the project can be a sub-folder in a monorepo. My first attempt at
Qodana usage failed because of that, I couldn't build my backend and frontend that live as sub-folders in the same repo.

Eventually, I made it work, the result YAML file for GitHub Actions is below.

## Solution

### qodana.yml

```yaml
name: "[PR] Qodana"
on:
  pull_request:
    types: [ synchronize, opened, reopened, ready_for_review ]

concurrency:
  group: {% raw %}${{ github.workflow }}{% endraw %}-{% raw %}${{ github.head_ref || github.run_id }}{% endraw %}
  cancel-in-progress: true

jobs:
  qodana-backend:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    steps:
      - name: Checkout the project
        uses: actions/checkout@v3

      - name: Skip the backend scan
        id: skip-scan
        uses: fkirc/skip-duplicate-actions@master
        with:
          paths: '["backend/**"]'

      - name: Scan the project
        if: steps.skip-scan.outputs.should_skip != 'true'
        uses: JetBrains/qodana-action@v2022.2.2
        timeout-minutes: 20
        with:
          # https://github.com/jetbrains/qodana-cli#options-1
          args: --project-dir,backend,--baseline,qodana.sarif.json
          results-dir: {% raw %}${{ runner.temp }}{% endraw %}/qodana/results-backend
          artifact-name: qodana-report-backend
          cache-dir: {% raw %}${{ runner.temp }}{% endraw %}/qodana/caches-backend
          additional-cache-hash: {% raw %}${{ github.sha }}{% endraw %}-backend
          pr-mode: false # otherwise, it adds the --changes flag and the check fails

  qodana-frontend:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    steps:
      - name: Checkout the project
        uses: actions/checkout@v3

      - name: Skip the frontend scan
        id: skip-scan
        uses: fkirc/skip-duplicate-actions@master
        with:
          paths: '["frontend/**"]'

      - name: Scan the project
        if: steps.skip-scan.outputs.should_skip != 'true'
        uses: JetBrains/qodana-action@v2022.2.2
        timeout-minutes: 5
        with:
          # https://github.com/jetbrains/qodana-cli#options-1
          args: --project-dir,frontend,--baseline,qodana.sarif.json
          results-dir: {% raw %}${{ runner.temp }}{% endraw %}/qodana/results-frontend
          artifact-name: qodana-report-frontend
          cache-dir: {% raw %}${{ runner.temp }}{% endraw %}/qodana/caches-frontend
          additional-cache-hash: {% raw %}${{ github.sha }}{% endraw %}-frontend
          pr-mode: false # otherwise, it adds the --changes flag and the check fails
```

### Explanation

- The action runs only for Pull Requests when they are opened, reopened, synced or marked as ready for review (draft PRs
  are skipped).
- The action uses the `concurrency` setting to cancel previous runs when a newer version of code is pushed to the
  branch.
- To understand why `fkirc/skip-duplicate-actions@master` is used, see 
  [Skip unnecessary builds on GitHub actions](https://peshrus.github.io/github-actions/skip-build/2021/09/05/skip-unnecessary-builds-on-github-actions.html)
- `--project-dir,backend` & `--project-dir,frontend` are used to switch the Qodana project directory. In that directory I have `qodana.yaml`
  & `qodana.sarif.json` files.
- `--baseline,qodana.sarif.json` is used to set up the baseline for the Qodana scan.
- `results-dir`, `artifact-name`, `cache-dir`, and `additional-cache-hash` are used to add the `-backend` and
  the `-frontend` postfixes to separate 2 steps that are executed in the same job. Otherwise, they clash with each
  other.
- `pr-mode: true` is the default value that adds to `args` `--changes,--commit,CI<SHA>`. However `--changes` raises an
  exception when it's used in a sub-folder (see [the found bugs](#found-bugs)). That's the reason why `pr-mode` is set
  to `false`.

## Found bugs

- [qodana.baselinePath setting in build.gradle raises an error](https://youtrack.jetbrains.com/issue/QD-4013)
- [The --changes argument raises an exception when used in a sub-folder of a monorepo project](https://youtrack.jetbrains.com/issue/QD-4014)
- [JetBrains/qodana-action@v2022.2.1 fails with an exception when used in a monorepo project](https://youtrack.jetbrains.com/issue/QD-4015)
- [The --source-directory argument with a sub-folder in a monorepo produces no results](https://youtrack.jetbrains.com/issue/QD-4025)
- [jetbrains/qodana-js:latest behaves differently locally and on CI](https://youtrack.jetbrains.com/issue/QD-4038)
- [jetbrains/qodana-jvm:latest behaves differently locally and on CI](https://youtrack.jetbrains.com/issue/QD-4039)

## Further reading

- [Qodana Scan Configuration](https://github.com/JetBrains/qodana-action#configuration)
- [Qodana CLI Options](https://github.com/jetbrains/qodana-cli#options-1)
- [Gradle Qodana Plugin Configuration](https://github.com/JetBrains/gradle-qodana-plugin#qodana---extension-configuration)
