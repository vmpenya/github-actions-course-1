# Expressions, Contexts, Functions, Environment Variables & Secrets.

[Main Documentation] (https://docs.github.com/en/actions/learn-github-actions/expressions).

Expressions can be used inside the run part of every step. They follow this format:

```
${{ Expresison }}
```
Here's an example

```yaml
name: External Events
on:
    repository_dispatch:
        types: [build]

jobs:
    echo-a-string:
        runs-on: ubuntu-latest
        steps:
            - run: echo ${{ github.event.client_payload.env }}
```

## Contexts

It's the _first word_ of an expression:

The context here is **github**:

```js
${{ github.event.client_payload.env }}
```

The context here is **inputs**:

```js
${{ inputs.texto }}
```

The context here is the **steps**:
```js
${{ steps.greet.outputs.time }}
```

## An example

```yaml
name: Expressions & Contexts
on: [push, pull_request, issues, workflow_dispatch]

jobs:
    using-expressions-and-contexts:
        runs-on: ubuntu-latest
        steps:
            - name: Expressions
              id: expressions_id
              # We can use numbers, strings, null, boolean values, comparison and logical operators
              run: |
                echo ${{ 1 }}                           
                echo ${{ 'This is a string' }}          
                echo ${{ null }}
                echo ${{ true }}
                echo ${{ 1 > 2 }}
                echo ${{ 'string' == 'String' }}
                echo ${{ true && false }}
                echo ${{ true || (false && true) }}
            - name: Dump Contexts
              # Here we use a function (toJson) to explore content of some contexts
              run: |
                echo '${{ toJson(github) }}'
                echo '${{ toJson(job) }}'
                echo '${{ toJson(secrets) }}'
                echo '${{ toJson(steps) }}'
                echo '${{ toJson(runner) }}'
```

## The IF key

Can be used in a job level or in a step level.

In a job level it's used like this:

```yaml
name: Expressions & Contexts
on: [push, pull_request, issues, workflow_dispatch]
run-name: 'Expressions & Contexts by @${{ github.actor }}, event ${{ github.event_name }}'

jobs:
    using-expressions-and-contexts:
        runs-on: ubuntu-latest
        if: ${{ github.event_name == 'push' }}  # This job will be executed only with push event
```

But we don't need to use the **${{ }}** syntax because Github actions will understand that everything in the IF is an expression.

```yaml
name: Expressions & Contexts
on: [push, pull_request, issues, workflow_dispatch]
run-name: 'Expressions & Contexts by @${{ github.actor }}, event ${{ github.event_name }}'

jobs:
    using-expressions-and-contexts:
        runs-on: ubuntu-latest
        if: github.event_name == 'push' # This job will be executed only with push event
```
You can use also if at a step level.

### Success

Github actions applies success condition to all if key by default. If we have this **if** key:

```yaml
if: steps.step-2.conclusion == 'failure'
```

Github Actions will _translate_ to something like this:

```yaml
if: success() && steps.step-2.conclusion == 'failure'
```

So if we need to apply this kind of condition you need to explicitly check for failure():

```yaml
if: failure() && steps.step-2.conclusion == 'failure'
```


## Functions


### contains
Functions that checks if a string is contained in another one. 

```js
contains(string, searched_string)
```

### Using an array with contains

```js
// First you need an array:

["first_value", "second_value"]

// You need no stringify this array with fromJson function

fromJson('["first_value", "second_value"]')     // See the use of simple quotation marks

// Te result is this
contains(fromJson('["first_value", "second_value"]'), "searched_string")        // false
contains(fromJson('["first_value", "second_value"]'), "second_value")           // true

```

### Access properties of an array of objects

If, for example, our workflow is launched by the creation or modification of an issue, this issue can have multiple labels. So inside the **github** context we can access the issue data through **github.events.issue**. Inside this theres one array of objets called **labels**. Every label has different properties (for example name). If we want to check if any label name of an issue is 'bug' (or any other string) an option is to get an array of all the names inside de **labels** array. We do this with this special syntax:

```js
    github.event.issue.labels.*.name            // This returns an array of names. ['bug', 'other', 'anything']
```

### Join

Joins in a string every member of an array seperated by a character

```js
    join(array, separator)

    join(github.event.issue.labels.*.name, ',')
    join(github.event.issue.labels.*.name, '||')
```

## Only execute steps depending on result (status) of previous steps

```yaml
name: Status Check Functions
on: [push]

jobs:
  job-1:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1
        run: sleep 20
      - name: Step 2
        id: step-2
        run: exit 1
      - name: Runs on Failure
        if: failure() && steps.step-2.conclusion == 'failure'
        run: echo 'Step 2 has failed.'
      - name: Runs on success
        # this is not needed beacuse this is the default behaviour.
        if: success()
        run: echo 'Runs on Success'
      - name: Always Runs
        # if: success() || failure()
        if: always()
        run: echo 'Always runs'
      - name: Runs When Cancelled
        if: cancelled()
        run: echo 'Runs on cancelled'
  job-2:
    runs-on: ubuntu-latest
    needs: job-1
    # only runs if job-1 fails
    if: failure()         
    steps: 
      - run: echo 'Job 2'
```

## Environment variables

In Github Actions there are default environment variables. The documentation for this variables is [here] (https://docs.github.com/en/actions/learn-github-actions/variables#default-environment-variables).

### User defined environment varibles.

Can be created by en **env** tag at workflow level, at job level and also at step level.

```yaml
env:
  WF_LEVEL_ENV: Available to all jobs
```

An environment variable defined at workflow level can be used inside any step of any job in the workflow. If it's defined at job level this variable can be used in ll steps of this job, but not in another job. One variable defined at step level can only be used in this steps.

A environment variable can be overriden in a lower level.

Here's one example:

```yaml
name: Environment Variables
on: [push]

# Define environment variables at workflow level available in all jobs
env:
  WF_LEVEL_ENV: Available to all jobs

jobs:
  env-vars-and-context:
    runs-on: ubuntu-latest
    # Here we need to use context because $GITHUB_REF don't exist in this level.
    if: github.ref == 'refs/heads/main'
    # if: $GITHUB_REF == 'refs/heads/main' will not work
    env:
      JOB_LEVEL_ENV: Available to all steps in env-vars-and-context job
    steps:
      # In this case is evaluated in the runner machine
      - name: Log ENV VAR
        run: echo $GITHUB_REF
      # In this case is evaluated by Github Actions before and sended to the runner machine once evaluated.
      - name: Log Context
        run: echo '${{ github.ref }}'
      - name: Log Custom ENV Vars
        env: 
          STEP_LEVEL_ENV: Only Available to only this step
          WF_LEVEL_ENV: Overriden in step
        run: |
          echo '${{ env.STEP_LEVEL_ENV }}'
          echo $STEP_LEVEL_ENV
          echo $WF_LEVEL_ENV
          echo $JOB_LEVEL_ENV   
```

### Evaluation of environment variables

We can use environment variables in 2 different ways. 
- $NAME_OF_VARIABLE. If we use this way the variable is evaluated inside the running machine.
- ${{ env.NAME_OF_VARIABLE }}. If we use this other method the variable is evaluated in Github Actions and then passed to the running machine. Using this method we can use environment variables inside **if** tag in the yaml.

### How to create a new variable or set a variable value while executing a workflow

For creating a new environment variable, or change the value of an existing one, inside the execution of a workflow, we need to add this variable or new value to a special file. We can access this file using the environment variable $GITHUB_ENV. Here's how we do this in bash:

#### Create new variable (or change the value)

```bash
echo "NEW_ENV_VAR=$(date)" >> "$GITHUB_ENV"
```

```powershell
"NEW_ENV_VAR=${date}" | Out-File -FilePath $env:GITHUB_ENV -Append
```

In the example below we created a new environment variable NEW_ENV_VAR and assign to it the value $(date) -which called the date function and get the time in the execution-.

#### Multiple lines values

For multiline strings, you may use a delimiter with the following syntax.

{name}<<{delimiter}
{value}
{delimiter}

Example in bash:

```bash
# For security reasons web create a random delimiter
DELIMITER=$(dd if=/dev/urandom bs=15 count=1 status=none | base64)
# Here we start the multi line value for the environment variable RANDOM_JOKE
echo "RANDOM_JOKE<<$DELIMITER" >> "$GITHUB_ENV"
echo "Here is a joke:" >> "$GITHUB_ENV"
curl -s https://icanhazdadjoke.com/ >> "$GITHUB_ENV"
# It's important to separate the end delimiter in a new line
echo "" >> "$GITHUB_ENV"
echo "$DELIMITER" >> "$GITHUB_ENV"
```

Example in powershell

[TO DO]

### Secrets and variables

You can configure secrets and variables at 3 levels:
- Organization Level. If you configure secrets or variables at organization level you can access this secrets or variables in any repository inside the organization.
- Repository Level. So you can access this variable or secret in any workflow. If we define a variable or secret that was defined in the organization level in Repository level we are overriding the value for this repository.
- Environment Level: same as both before. For accesing variables created at environment level you need to specify an environment in the workflow yaml file.

Example of using:

```yaml
name: Variables and Secrets
on: [push]

jobs:
  log-vars:
    runs-on: ${{ vars.JOBS_RUNNER }}
    environment: 'staging'
    env:
        BOOLEAN_SECRET: ${{ secrets.BOOLEAN_SECRET }}
        ENV_LEVEL_VAR: ${{ vars.ENV_LEVEL_VAR }}
        REPO_LEVEL_VAR: ${{ vars.REPO_LEVEL_VAR }}
        ORG_LEVEL_VAR: ${{ vars.ORG_LEVEL_VAR }}
    steps:
      - name: Run only if BOOLEAN_SECRET is true
        if: env.BOOLEAN_SECRET == 'true'
        run: echo "I ran only when BOOLEAN_SECRET is true"
      - name: Log vars
        run: |
          echo '${{ vars.JOBS_RUNNER }}'
          echo "ORG_LEVEL_VAR: $ORG_LEVEL_VAR"
          echo "REPO_LEVEL_VAR: $REPO_LEVEL_VAR"
          echo "ENV_LEVEL_VAR: $ENV_LEVEL_VAR" 
```

### Large secrets file

Secrets and variables has a size limit. If you need to have a large amount of secret data you can do this using gpg.

GPG is a tool for encrypt data. What you do is encrypt date locally with the tool. For example, if you want to encrypt the contents of a file "secret.json", you can do it this way:

```powershell
gpg --symmetric --cipher-algo AES256 secret.json
```
--cipher-algo selects cipher algorithm.
secret.json is the filename


This will create a encrypted file called secret.json.gpg with the encrypted data. To decrypt the file locally you can execute:

```powershell
gpg --decrypt --passphrase="123456" --output decrypted.json .\secret.json.gpg
```

Here we must provide the passphrase, the output option specifies the output file, and secret.json.gpg is the encrypted file.

To decrypt in the workflow use this:

```yaml
jobs:
  decrypt-file:
    runs-on: ${{ vars.JOBS_RUNNER }}
    steps:
      - uses: actions/checkout@v3
      - name: Decrypt file
        # The passphrase is configured as a secret in the repository
        env: 
          PASSPHRASE: ${{ secrets.PASSPHRASE }}
        # in this step we run gpg to decrypt the file
        run: |
          mkdir $HOME/secrets
          gpg --quiet --batch --yes --decrypt --passphrase="$PASSPHRASE" --output $HOME/secrets/secret.json secret.json.gpg
        # You shouldn't do this. This is only for testing
      - name: View Encrypted file contents
        run:
          cat $HOME/secrets/secret.json
```



## Permissions

We can stablish the permissions that the GITHUB_TOKEN will have inside github with the **permissions** key:

```yaml

jobs:
    pr-comment:
        runs-on: ubuntu-latest
        # when we specify one or more permissions all the other will be set to none.
        permissions:
            pull-requests: write
        # permissions: read-all or permissions: write-all or {} for no permissions

```

The token will have only the permissions that we specify, and all the other ones will be set to None.

We can configure the permissions at workflow level or at job level.

Heres the link to the [permissions table](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token).

