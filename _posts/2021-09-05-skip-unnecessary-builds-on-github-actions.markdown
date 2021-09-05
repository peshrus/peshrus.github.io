---
layout: post
title:  "Skip unnecessary builds on GitHub actions"
date:   2021-09-05 10:47:00 +0200
categories: github-actions skip-build
---

## Motivation

The initial problem that I faced was building both the front end and the back end from the same
project folder. Sometimes only one of them is changed or even none of them and only the
documentation. In such cases, the relevant build should be skipped.

## Prerequisites

[GitHub actions](https://github.com/features/actions) are used to build the project.

## Solution

**backend-workflow.yml**
```yaml
name: backend

on:
  pull_request:
    types: [ synchronize, opened, reopened, ready_for_review ]

jobs:
  backend:
    runs-on: ubuntu-latest
    if: github.event.pull_request.draft == false
    steps:
      - name: Checkout the Project
        uses: actions/checkout@v2
      - name: Skip the Build
        id: skip_build
        uses: fkirc/skip-duplicate-actions@master
        with:
          paths: '["backend/gradle/**", "backend/src/**", "backend/build.gradle", "backend/gradle.properties", "backend/settings.gradle"]'
      - name: Set up JDK 1.8
        if: ${{ steps.skip_build.outputs.should_skip != 'true' }}
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      # Any further step should have the IF-check as it's used in the previous step
``` 

**frontend-workflow.yml**
```yaml
name: frontend

on:
  pull_request:
    types: [ synchronize, opened, reopened, ready_for_review ]

jobs:
  frontend:
    runs-on: ubuntu-latest
    if: github.event.pull_request.draft == false
    steps:
      - name: Checkout the Project
        uses: actions/checkout@v2
      - name: Skip the Build
        id: skip_build
        uses: fkirc/skip-duplicate-actions@master
        with:
          paths: '["frontend/src/**", "frontend/package.json"]'
      - name: Setup Node.js environment
        if: ${{ steps.skip_build.outputs.should_skip != 'true' }}
        uses: actions/setup-node@v1
        with:
          node-version: 12
      # Any further step should have the IF-check as it's used in the previous step
``` 

## Solution

Further
reading: [Skip Duplicate Actions](https://github.com/fkirc/skip-duplicate-actions#skip-duplicate-actions)
