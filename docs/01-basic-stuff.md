# Basic stuff

Workflows are configured in yaml files. Yaml is like a simplified JSON format, but without the need of the keys. It's a super set of JSON.

## Basic structure of a yaml Github Actions Workflow file

```yaml
name: Name of the workflow          # Name for this workflow
on: [push]                          # array of conditions that runs our workflow

jobs:                               # every job is an object. Jobs are executed in parallel mode until you specify other thing
    job-name-1:
        runs-on: ubuntu-latest      # specifies machine where job runs
        steps:                      # every step is executed after the previous one finishes
              # Name of the step
            - name: Simple JS Action
              # Id of the step. It's needed with some actions
              id: greet
              # If we use an action you identify the action this way
              uses: actions/hello-world-javascript-action@319435519c77d9858289ca465032731e22c5416c
              # Parameters for the action
              with:
                who-to-greet: Víctor
            - name: Log Greeting time
              # You can run your own code instead of an action. In this case you must use bash or powershell depending on the system.
              run: echo "${{ steps.greet.outputs.time }}"        

```

## Dependent jobs

Here we have 3 jobs:
- run-shell-commands is the first job.
- parallel-macos-job runs in parallel with the previous job in its own machine.
- dependent-job runs only when run-shell-commands is finished

```yaml
name: First Workflow
on: [push]

jobs:
  run-shell-commands:
    runs-on: ubuntu-latest
    steps: 
      - name: echo a string
        run: echo "Hello World"
      - name: Multiline Commands
        run: |
          node -v
          npm -v
  parallel-job-macos:
    runs-on: macos-latest
    steps: 
      - name: View SW Version
        run: sw_vers
  dependent-job:
    runs-on: windows-latest
    needs: run-shell-commands           # Here we made this job dependent from the first one
    steps:
      - name: echo a string
        run: Write-Output "Windows String"
      - name: Error Step
        run: doesnotexistssss

```

## Don't run the workflow

We can skip running one workflow if we user one of this descriptions inside the push message:

- [skip ci]
- [ci skip]
- [no ci]
- [skip actions]
- [actions skip]

## Workflow commands

You can use workflow commands when running shell commands in a workflow or in an action's code.

Here's the link to the [Documentation] (https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions?tool=bash)


## Checkout

There's a Github Action for copying all the content of a branch of the repository inside de machine running the job: [actions/checkout] (https://github.com/marketplace/actions/checkout).

Here's an example of how to use it

```yaml
name: Checkout
on: [push]

jobs:
    checkout-action:
        runs-on: ubuntu-latest
        steps:
            - name: List Files Before
              run: ls -a
            - uses: actions/checkout@v3     # This step uses the action to copy all the content of the repository inside the machine
            - name: List Files After
              run: ls -a
    checkout-and-display-files:
        runs-on: ubuntu-latest
        steps:
            - name: List Files Before
              run: ls -a
            - name: Checkout                # In this step we do the same calling git mannually
              run: |
                git init
                git remote add origin "https://$GITHUB_ACTOR:${{ secrets.GITHUB_TOKEN }}@github.com/$GITHUB_REPOSITORY.git"
                git fetch origin
                git checkout main
            - name: List Files After
              run: ls -a
```