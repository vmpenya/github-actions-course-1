# Introduction

Github actions can respond to a lot of events, and also be launched manually, or with an schedule, and also me called with a REST API.

You can configure your workflows to run when specific activity on GitHub happens, at a scheduled time, or when an event outside of GitHub occurs.

The documentation is [here] (https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows).

## Events as an array

You can define Events as an array

```yaml
name: Repository Events
on: [push, pull_request, issue]
```

## Activity types

Every event has some activity types that we can use to control when our workflow is launched:

```yaml
name: Repository Events
on:                         # Now we express every event as an object
  push:                     # If we don't put any type, the workflow is executed every time the event occurs
  pull_request:             # Types is an array of activity types. The workflow only runs with this activity type.
    types: [opened, assigned, reopened, synchronize]
  issues:
    types: [opened]
```

## Github Script

There's an action called [actions/github-script] (https://github.com/actions/github-script) that allows us to interact with github in our workflows. 

You need to use the rest api with this action. The documentation is [here] (https://octokit.github.io/rest.js/v19).

```yaml
name: Pull Request Comment
on: 
    pull_request_target:
        types: [opened]

jobs:
    pr-comment:
        runs-on: ubuntu-latest
        steps:
            - name: Comment on new PRs
              uses: actions/github-script@v6                # We use this action, github-script
              with:
                script: |
                    github.rest.issues.createComment({
                        owner: context.repo.owner,
                        repo: context.repo.repo,
                        issue_number: context.issue.number,
                        body: 'Thanks for contributing!',
                    });
```

## Run a workflow after another one

You can configure workflows to run after another one is finished. It only accepts 3 levels of nesting. Example code:

```yaml
name: Workflow Run
on:
    workflow_run:                               # The workflow will only run after the workflow named Repository Events is completed.
        workflows: [Repository Events]
        types: [completed]
        branches:                               # We can filter for only run the workflow when the previous workflow runs on one branch.
            - main
jobs:
    echo-string: 
        runs-on: ubuntu-latest
        steps:
            - run: echo "I was triggered becaause 'Repository Events' waas completed"
```

## Filtering by branches, tags and paths

```yaml

name: Repository Events
on: 
  push:
    # All rules should be matched to run the workflow
    branches:
      - main
      - "feature/*"   # matches feature/featA, feature/featB, does not match feature/featA/featB
      - "feature/**"  # matches feature/featA, feature/featB, an also feature/featA/featB
      - "!feature/featA"  # excludes this branch. The order is important. The last rule is what is done.
#    branches-ignore:
#      - develop
    tags: 
      - v1.* # match v1.1, v1.1.2...
      - "!v1.1.1"
    paths:
      - "**.js"
      - "!app.js"   # excludes this file

```

## Manually triggering a Workflow

You can configure the workflow for beeing triggered manually. Example:

```yaml
name: Manually Triggered
on:
    workflow_dispatch:
        inputs:
            texto:
                description: Un campo de texto
                type: string
                required: true
                default: "Valor por defecto"
            numero:
                description: Un campo numérico
                type: number
                default: 5
            opciones:
                description: Múltiples opciones
                required: true
                default: "Tercera Opción"
                type: choice
                options:
                    - Primera Opción
                    - Segunda Opción
                    - Tercera Opción
            booleano:
                description: Un campo verdadero o falso
                required: false
                type: boolean
            entorno:
                description: "Entorno"
                type: environment
                required: true
```

You can dispatch de event also by the [REST API] (https://docs.github.com/en/rest/actions/workflows?apiVersion=2022-11-28#create-a-workflow-dispatch-event).


## Schedule

Uses cron format.

This tool can help you with the format:

https://crontab.guru/

```yaml
name: Stale Issues & PRs
on:
    schedule:
         - cron: "0 14 * * *"
         - cron: "0/5 * * * *" # JUST FOR TESTING. This is the minimum time
```