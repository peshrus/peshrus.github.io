---
layout: post
title:  "A GitHub action to upgrade the gradle wrapper"
date:   2023-12-05 21:25:00 +0100
categories: github-actions gradle
---

## The motivation

[Dependabot](https://github.com/dependabot) is an ideal tool that automatically updates project dependencies, but it
[does not support upgrading the Gradle wrapper](https://github.com/dependabot/dependabot-core/issues/2223).

Would be nice to have a GitHub action that would upgrade the Gradle wrapper to the latest version when it's released.

## Existing solutions

There is [an existing action](https://github.com/marketplace/actions/upgrade-gradle) in the market that could be used to
upgrade the Gradle wrapper. 

However, I wanted to print some details about the upgrade in
the [action summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#example-of-adding-a-job-summary).

## My solution

### upgrade-gradle.yml

```yaml
name: Upgrade Gradle

on:
  schedule:
    # * is a special character in YAML so you have to quote this string
    #        ┌───── minute (0 - 59)
    #        │ ┌──── hour (0 - 23)
    #        │ │ ┌─── day of the month (1 - 31)
    #        │ │ │ ┌── month (1 - 12 or JAN-DEC)
    #        │ │ │ │ ┌─ day of the week (0 - 6 or SUN-SAT)
    #        │ │ │ │ │
    #        * * * * *
    - cron: "0 6 * * MON-FRI"

jobs:
  upgrade-gradle:
    name: Upgrade Gradle
    runs-on: ubuntu-latest
    steps:
      - id: latest-gradle-version
        name: Find the latest released Gradle version
        uses: actions/github-script@v7
        with:
          script: |
            console.log("Finding the latest available Gradle version...");

            const latestRelease = (await github.rest.repos.getLatestRelease({ owner: "gradle", repo: "gradle"})).data;
            if (!latestRelease) {
                console.log("The latest Gradle release is not found");
                return;
            }
            console.log(`The latest Gradle version: ${latestRelease.name}`);

            let summary = core.summary;
            summary = summary.addHeading("The latest Gradle version");
            summary = summary.addLink(`${latestRelease.name}`, latestRelease.html_url);
            summary.write();

            core.setOutput("value", latestRelease.name);
            core.setOutput("description", latestRelease.body);

      - name: Checkout the project
        uses: actions/checkout@v4
        with:
          ref: master

      - id: used-gradle-version
        name: Get the used Gradle version
        run: |
          usedGradleVersion=$(./gradlew -v | grep -oP "^Gradle\ [0-9.]+" | sed "s/Gradle //g")

          echo "# The used Gradle version" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "[$usedGradleVersion](https://github.com/<ORG>/<REPO>/blob/master/gradle/wrapper/gradle-wrapper.properties#L3)" >> $GITHUB_STEP_SUMMARY

          echo "result=$usedGradleVersion" >> "$GITHUB_OUTPUT"

      - name: Set up JDK
        if: ${{ steps.latest-gradle-version.outputs.value != steps.used-gradle-version.outputs.result }}
        uses: actions/setup-java@v4
        with:
          distribution: corretto
          java-version-file: .java-version

      - name: Upgrade Gradle
        if: ${{ steps.latest-gradle-version.outputs.value != steps.used-gradle-version.outputs.result }}
        run: ./gradlew wrapper --gradle-version latest

      - id: create-pr
        name: Create PR
        if: ${{ steps.latest-gradle-version.outputs.value != steps.used-gradle-version.outputs.result }}
        uses: peter-evans/create-pull-request@v5
        with:
          commit-message: Upgrade Gradle to v${{ steps.latest-gradle-version.outputs.value }}
          branch: chore/upgrade-gradle-to-v${{ steps.latest-gradle-version.outputs.value }}
          title: Upgrade Gradle to v${{ steps.latest-gradle-version.outputs.value }}
          body: ${{ steps.latest-gradle-version.outputs.description }}
          labels: backend,dependencies,infrastructure

      - name: Add PR URL to the summary
        if: ${{ steps.latest-gradle-version.outputs.value != steps.used-gradle-version.outputs.result }}
        run: |
          echo "# Created PR" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "[${{ steps.create-pr.outputs.pull-request-number }}](${{ steps.create-pr.outputs.pull-request-url }})" >> $GITHUB_STEP_SUMMARY
```

### Explanation

- The action is scheduled to run every day at 6:00 AM UTC from Monday to Friday.
- The 1st step finds the latest released Gradle version in the Gradle repository and prints it in the action summary.
- The 2nd step checks out the project.
- The 3rd step gets the used Gradle version and prints it in the action summary.
- The 4th step sets up the JDK.
- The 5th step upgrades the Gradle wrapper.
- The 6th step creates a pull request with the changes.
- The 7th step prints the pull request URL in the action summary.

Steps 4-7 are executed only if the latest Gradle version is different from the used Gradle version.
