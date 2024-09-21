# Diving Deeper with More Advanced Github Actions Features
## Contents
- Timeout Minutes & Continue on Error
- Running a Job Multiple Times Using a Mtrix
- Including & Excluding Matrix Configurations
- Hanling Failing Jobs in a Matrix
- Step and Job Outputs & Dynamic Matrices
- Running a Single Job or Workflow at a Time Using Concurrency
- Reusable Workflows
- Reusable Workflows Outputs
- Nesting Reusable Workflows
- Caching Files in Github Actions
- Updating Cache Keys Dinamically && Adding Restore Keys
- Cache Limits & Restrictions
- Uploading & Downloading Job Artifacts


## Timeout Minutes & Continue on Error

### continue-on-error
At step level. If true then the execution of the following steps continues even if this step fails.

In this example Step 2 fails. But as we have configured the step with **continue-on-error: true** then all the following steps are executed as if there wasn't one fail. For example, **success()** function return true, just as is there was not a failure.
```yaml
    steps:
      - name: Step 1
        run: sleep 80
        timeout-minutes: 1
      - name: Step 2
        id: step-2
        continue-on-error: true
        run: exit 1
      - name: Runs on setp 2 Failure
        if: failure() && steps.step-2.conclusion == 'failure'
        run: echo 'Step 2 has failed.'
      - name: Runs on Failure
        if: failure() 
        run: echo 'Failure'        
      - name: Runs on success
        # this is not needed beacuse this is the default behaviour.
        if: success()
        run: echo 'Runs on Success'
```

### timeout-minutes:

Stablish a number of minutes for stopping the execution of our workflow. For example this step will be cancelled after 1 minute:

```yaml
    steps:
      - name: Step 1
        run: sleep 90
        timeout-minutes: 1
      - name: Step 2
        id: step-2
        continue-on-error: true
        run: exit 1
```

This can be set also at job level (so the time corresponds to the full job)

## Running a Job Multiple Times Using a Mtrix
If you need to run the job multiple times you can define a matrix and the job will be executed multiple times.

At job level you can create a **strategy** tag, where we can configure the matrix and the behaviour of the job.

```yaml
jobs:
  node-matrix:
    continue-on-error: false
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [14, 15, 16]
      fail-fast: false
      max-parallel: 3
```

Here we can notice some different options. First we have a two dimension matrix. The first fimmension name is **os**, and the second dimmension is called **node-version**. The name of every dimension is free and the user can choose whatever is the most clear word. As you can see we have 2 values in the **os** dimension, and 3 values on **node-version**. This means that the job will be executed 6 times, one with every combination of values of the matrix. That is:

- One execution in ubuntu-latest, for version 14.
- One execution in ubuntu-latest, for version 15.
- One execution in ubuntu-latest, for version 16.
- One execution in windows-latest, for version 14.
- One execution in windows-latest, for version 15.
- One execution in windows-latest, for version 16.

We need to use the context to use the values of the matrix inside every execution. For example:

```yaml
    runs-on: ${{ matrix.os }}
    steps:
      - run: node -v
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: node -v
```

Here we use the value of **os** for the runs-on option, and the **node-version** inside one step.

Every execution of the matrix will be launched in parallel. If we want to limit the number of parallel executions we can use the option **max-parallel** to set the maximum number of parallel executions.


## Including & Excluding Matrix Configurations

We can configure and addapt the matrix values in a fine grain way with the **includes** keyword. Let's start with this example:

```yaml
jobs:
  node-matrix:
    continue-on-error: false
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [14, 15]
```

In this example we have 4 executions for the job:

1. **os: ubuntu-latest** and **node-version: 14**.
2. **os: ubuntu-latest** and **node-version: 15**.
3. **os: windows-latest** and **node-version: 14**.
4. **os: windows-latest** and **node-version: 15**.

Now we can create a **include** tag. Inside include tag you can define different objects to fine tune the matrix. Let's see some examples:

```yaml
jobs:
  node-matrix:
    continue-on-error: false
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node-version: [14, 15]
        include: 
          - os: ubuntu-latest
            is-ubuntu: true
          - os: macos-latest
            node-version: 15
          - experimental: false
          - os: ubuntu-latest
            node-version: 16
            experimental: true
          - os: ubuntu-latest
            node-version: 17            
```

The first thing we need to understand is that **include** will try to find existing combinations, and change this combinations, before adding one more option to the matrix. Let's see the first **include** object:

```yaml
          - os: ubuntu-latest
            is-ubuntu: true
```

This seems that is adding another tag (__is-ubuntu__) to the matrix, so now we have more combinations. But don't work that way. What this **include** object is doing is add the property **is-ubuntu**, with the value of **true** to every combination of the matrix where **os** tag has the **ubuntu-latest** value. Then our matrix will have this values:

1. **os: ubuntu-latest**, **node-version: 14** and **is-ubuntu: true**.
2. **os: ubuntu-latest**, **node-version: 15** and **is-ubuntu: true**.
3. **os: windows-latest** and **node-version: 14**.
4. **os: windows-latest** and **node-version: 15**.

Combinations 3 and 4 don't have the new tag with a value because they don't match the os property with the value "ubuntu-latest". This works as pattern matching in elixir or rust.

If we add an option with a tag that exists but a value that don't matches any existing combination, then a new execution is added to the matrix. This is what the next object is doing:

```yaml
          - os: macos-latest
            node-version: 15
```

After the addition of this object the execution matrix has this combinations:

1. **os: ubuntu-latest**, **node-version: 14** and **is-ubuntu: true**.
2. **os: ubuntu-latest**, **node-version: 15** and **is-ubuntu: true**.
3. **os: windows-latest** and **node-version: 14**.
4. **os: windows-latest** and **node-version: 15**.
5. **os: macos-latest** and **node-version: 15**.

In the case we add a new tag, that don't exists previously in the matrix, this is added as a new property and value to the existing combinations, not a new ones. For example this line:

```yaml
          - experimental: false
```

After this line inside **includes** the matrix execution combination has this values:

1. **os: ubuntu-latest**, **node-version: 14**, **is-ubuntu: true** and **experimental: false**.
2. **os: ubuntu-latest**, **node-version: 15**, **is-ubuntu: true** and **experimental: false**.
3. **os: windows-latest**, **node-version: 14** and **experimental: false**.
4. **os: windows-latest**, **node-version: 15** and **experimental: false**.
5. **os: macos-latest**, **node-version: 15**.

As you can see in the original matrix there wasn't a tag called experimental, so this tag is added to all the combinations, but no new execution is created. You muyst notice that this only changes the original values for the matrix, so the **macos-latest** combination remains unchanged. This is very important.

Now, If we want to change a value for the **experimental** tag, we can use patter matching as before:

```yaml
          - os: ubuntu-latest
            node-version: 15
            experimental: true
```

These are the new values:

1. **os: ubuntu-latest**, **node-version: 14**, **is-ubuntu: true** and **experimental: false**.
2. **os: ubuntu-latest**, **node-version: 15**, **is-ubuntu: true** and **experimental: true**.
3. **os: windows-latest**, **node-version: 14** and **experimental: false**.
4. **os: windows-latest**, **node-version: 15** and **experimental: false**.
5. **os: macos-latest**, **node-version: 15** and **experimental: false**.

With the pattern matching technique we changed the value **experimental** to **true** for a combination of operating system and node version.

Finally, if you add new combinations, they don't get the new values and tag created, only what you configure:

```yaml
          - os: ubuntu-latest
            node-version: 17            
```

1. **os: ubuntu-latest**, **node-version: 14**, **is-ubuntu: true** and **experimental: false**.
2. **os: ubuntu-latest**, **node-version: 15**, **is-ubuntu: true** and **experimental: true**.
3. **os: windows-latest**, **node-version: 14** and **experimental: false**.
4. **os: windows-latest**, **node-version: 15** and **experimental: false**.
5. **os: macos-latest**, **node-version: 15** and **experimental: false**.
6. **os: ubuntu-latest**, **node-version: 17**.

As you can see this new combination don't have a tag or value for **experimental**.

There's also an **exclude** tag that you can use to exclude executions that matches the pattern. For example we can add this object to exclude the execution of a node version inside one operating systema:


```yaml
        excludes:
          - os: windows-version
            node-version: 15            
```

If we make this exclusion we will have this executions:

1. **os: ubuntu-latest**, **node-version: 14**, **is-ubuntu: true** and **experimental: false**.
2. **os: ubuntu-latest**, **node-version: 15**, **is-ubuntu: true** and **experimental: true**.
3. **os: windows-latest**, **node-version: 14** and **experimental: false**.
4. **os: macos-latest**, **node-version: 15** and **experimental: false**.
5. **os: ubuntu-latest**, **node-version: 17**.

You can see we now have one less execution.


## Hanling Failing Jobs in a Matrix

In the first section we saw how **continue-on-error** tag works in conjungtion with **fail-fast** tag**:
- **continue-on-error**. Default option is false, and that means that if a job fails Github Actions will cancel all next jobs. In the case of a matrix all not executed combinations will not be executed. Changing this option to true will make that all the jobs will be executed even if some of them fails.
- **fail-fast**. If this is set to true Github Actions will cancel running jobs when one of the jobs fails.

We can configure this better using the matrix options and their tags. If, for example, we want that jobs with the option **experimental: true** have the **continue-on-error** to true, so all the other jobs will be executed, we can configure this using the matrix context. In this example we allow the jobs to fail if they are experimental or macos.

```yaml
jobs:
  node-matrix:
    continue-on-error: ${{ (matrix.experimental == true) || (matrix.os == 'macos-latest') }}
```

## Step and Job Outputs & Dynamic Matrices

### Step output

We can output values from a step to use in next steps of a job. We do this adding key=value strings to a special file contained in $GITHUB_OUTPUT. Here we have an example. In the first and second step we create outputs in every step. We can create more than one output in any step:


```yaml
jobs:
  output-values-example:
    runs-on: ubuntu-latest
    steps:
      - run: echo "step-output=value" >> $GITHUB_OUTPUT
        id: step-1
      - run: |
          echo "type-of-animal=butterfly" >> $GITHUB_OUTPUT
          echo "color=red" >> $GITHUB_OUTPUT
        id: step-2
      - run: |
          echo '${{ steps.step-1.outputs.step-output }}'
          echo '${{ steps.step-2.outputs.type-of-animal }}'
          echo '${{ steps.step-2.outputs.color }}'
```

In this example we have created one step that has an output with the key **step-output** and the value **value**. Then there is another step that created 2 outputs. The first one is **type-of-animal** which value is **butterfly**, and the second one is **color** with the value **red**.

Then in the last step we access this outputs with the context **steps.id of the step.outputs.key of the output** syntax.

### Job outputs

Step outputs are not available outside of their job. If we need this we must set the job output. Here we have an interesting example:

```yaml
  prepare-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix-arrays: ${{ steps.matrix-arrays.outputs.result }}
    steps:
      - name: Get all configured options
        id: matrix-arrays
        uses: actions/github-script@v6
        with: 
          script: "return {os: context.payload.inputs['os'].split(','), 'node-version': context.payload.inputs['node-version'].split(',') }"
          result-encoding: json
      - name: Check the values recovered
        run: echo '${{ steps.matrix-arrays.outputs.result }}'
```

In the **prepare-matrix** jon we find the **outputs** tag. Here we create and output which name is **matrix-arrays** and the value is attached to the output of a step inside the job (which name is **matrix-arrays**).

In this case we used a Github Action (actions/github-script) that allows us to use javascript and returns everything we put in the return sentence. In this special case we convert two comma separated strings in an json object containing two arrays of values.

### Dynamic Matrices

We can then configure a job with a matrix in a dynamic way. 

First thing we need to do is to configure the job for executing after the **prepare-matrix** job, that created de output with the matrix values.

```yaml
  node-matrix:
    needs: prepare-matrix
```
After that we set our strategy, creating a matrix for every tag:

```yaml
  node-matrix:
    needs: prepare-matrix
    strategy:
      matrix:
        os: ${{ fromJson(needs.prepare-matrix.outputs.matrix-arrays).os }}
        node-version: ${{ fromJson(needs.prepare-matrix.outputs.matrix-arrays).node-version }}
```

The complete code is this:

```yaml
  node-matrix:
    needs: prepare-matrix
    strategy:
      matrix:
        os: ${{ fromJson(needs.prepare-matrix.outputs.matrix-arrays).os }}
        node-version: ${{ fromJson(needs.prepare-matrix.outputs.matrix-arrays).node-version }}
    runs-on: ${{ matrix.os }}
    steps:
      - run: node -v
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: node -v
```

## Running a Single Job or Workflow at a Time Using Concurrency

Concurrency can be configured at job or workflow level. We asign a concurrency group tu a workflow or a job, and executions in the same group can only happen one after another, avoiding parallel execution. The syntax is:

```yaml
concurrency: 
  group: name-of-the-group
  cancel-in-progress: true
```

Github Actions will prevent from executing jobs in the same concurrency group.

If **cancel-in-progress** tag is set to true Github Actions will cancel in progress jobs to execute the last one. If set to false, the first one will be executed and those who are initiaded inside the execution time of the executing job will be cancelled. So if true GH executes the last one, if false, the first one.

We can define concurrency group names dynamically too. Just we must be sure that different workflows don't share the concurrency name. In a very few cases you will be interested in this.

Example:

```yaml
  concurrency:
    group: ${{ github.workflow }} - ${{ github.event.inputs.environment }}
    cancel-in-progress: true
```



## Reusable Workflows

When we want to reuse a workflow we will have one workflow that *calls* the other. The first one, the main workflow, it's known as **Caller workflow**. We will name the other one as the **Called workflow**. The Called workflow and the Caller workflow can be in the same repository, or in a different one. 

### Reusablle *Called* workflow

We can reuse a workflow from the same repository and from another one, but we need to ensure we have configured the corganization or the repository to allow this in Settings -> Actions -> General.

Reusable workflows executes on **workflow_call**, that has a similiar configuration as *workflow_dispatch*, so we can create inputs and secrets that we can use in the every execution inside de workflow. Let's see one example:

```yaml
name: Reusable Workflow
on:
  workflow_call:
    inputs:
      name: 
        description: 'Input description'
        type: string
        default: 'Víctor'
        required: false
    secrets:
      access-token: 
        description: 'Secret description'
        required: true

jobs:
  checkout:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: ls -a
  log-context-and-inputs:
    runs-on: ubuntu-latest
    steps:
      - name: Display Github Context
        run: echo '${{ toJSON(github) }}'
      - name: Display name inputs
        run: echo '${{ inputs.name }}'
      - name: Display access-token secret
        run: echo '${{ secrets.access-token }}'
```

Let's comment different options:
1. We use the **workflow_call** event and we configure **inputs** and **secrets**.
2. In this example we show the github context and the inputs and secrets. This will contain the context and also the inputs and secrets provided by the **caller workflow**.

### *Caller* workflow

In the caller workflow we will use the **uses** key, that we have used in steps fopr using an standard action, but a workflow level. Let's see one example:

```yaml
name: Calling Reusable Workflows
on:
  workflow_dispatch:
    inputs:
      name: 
        description: 'Input description'
        type: string
        default: 'Víctor'
        required: false

jobs:
  calling-a-resusable-wf:
    uses: VictorPenya/github-actions-course/.github/workflows/reusable.yaml@main
    with: 
      name: ${{ inputs.name }}
    # secrets: inherit     pass all secrets to the called workflow
    secrets:
      access-token: ${{ secrets.ACCESS_TOKEN }}
```

In the **jobs** section we create a job called *calling-a-reusable-wf* and here we use the *uses* keyword to indicate the workflow. As in the example the *called workflow* resides in another repository we need to provide the complete user-or-organization/repository/route-to-workflow-file path. If we are executing one workflow that is in the same repository we just need to write the local path to this file. For example: ./reusable.yaml if it's in the same folder.

In the example we are providing a branch (*main*), but this is only for an example. In a real environment we will be using one version or one commit ID instead to prevent problems derived from further changes in the workflow.

Once executed this execution will appear in the *caller workflow* repository.

## Reusable Workflows Outputs

We can configure workflow level outputs that can be recovered from a called workflow for use in the caller one. 

To do so, in the reusable workflow (the called one), we must create first of all a step Output. We do this copying the value in the file contained in $GITHUB_OUTPUT environment variable:

```yaml
    steps:
      # Create the variable "date", that contains a value of the execution date, and save it to $GITHUB_OUTPUT file
      - run: echo "date=$(date)" > $GITHUB_OUTPUT 
        id: date-step       # Create an id for the step so can be referenced later
```

The we create a job level output containing the "date" variable:

```yaml
jobs:
  generate-output:
    runs-on: ubuntu-latest
    # Here we have the job output
    outputs:
      # We return de date variable from date-step step
      date: ${{ steps.date-step.outputs.date }}
```

And then we can set the output at workflow level, using the output of this job. It's done as in the following example:

```yaml
name: Reusable Workflow
on:
  # This workflow is executed when it's called
  workflow_call:
    # Workflow level output, same as inputs
    outputs:
      date: 
        description: 'Date Value'
        # We access the context to recover the variable date from the job's outputs
        value: ${{ jobs.generate-output.outputs.date }}
```

This is the complete code of the reusable workflow that has outputs:

```yaml
name: Reusable Workflow
on:
  workflow_call:
    outputs:
      date: 
        description: 'Date Value'
        value: ${{ jobs.generate-output.outputs.date }}

jobs:
  generate-output:
    runs-on: ubuntu-latest
    outputs:
      date: ${{ steps.date-step.outputs.date }}
    steps:
      - run: echo "date=$(date)" > $GITHUB_OUTPUT 
        id: date-step
```

Here's an example of how to recover the output variable and use it in the caller workflow:

```yaml
jobs:
  # In this job we call the reusable workflow. It's in the local repo
  calling-a-reusable-wf-in-the-same-repo:
    uses: ./.github/workflows/reusable-workflow.yaml
  # Here we will use the optput
  using-reusable-wf-output:
    runs-on: ubuntu-latest
    # Firts: we must wait from previous job to finish so we have the output
    needs: calling-a-reusable-wf-in-the-same-repo
    steps:
      # In the context, in the needs object, we can access outputs of needed steps
      - run: echo ${{ needs.calling-a-reusable-wf-in-the-same-repo.outputs.date }}
```

## Nesting Reusable Workflows
Nesting workflows is the capability of calling a workflow inside a called workflow. There are two main important ideas that we need to understand:

1. Secrets and inputs must be provided in the call. If a workflow have inputs or secrets these must be provided by the caller workflow. You can inherit the inputs and secrets with the inherit keyword:

```yaml
jobs:
  calling-a-reusable-wf-in-the-same-repo:
    uses: ./.github/workflows/reusable-workflow.yaml
    # Pass all secrets to the called workflow
    secrets: inherit
```

2. Permissions of the called workflow are limited by those that the caller one has. A inherited workflow couldn't scale or increase permissions.

## Caching Files in Github Actions

When we use a tool like npm to manage our dependencies in the local machine we have all the dependencies cached in our machine, so the amount of time needed to install dependencies are reduced. The problema with Github Actions is that usually de dependencies folder are not syncrhronized with the github repository (because we put this folder inside .gitignore file). That means that everytime we use an action for deploy our code we need to install all dependencies, and that will consume a lot of time.

```yaml
name: Caching and Artifacts
on: [workflow_dispatch]

job:
  use-axios:
    runs-on: ubuntu_latest
    steps:
      - uses: actions/checkout@v3
      - name: Install dependencies
        run: rpm install
      - name: Use Axios
        uses: actions/github-script@v6
        with:
          script: |
            const axios = require('axios');
            const res = await axios('https://icanhazdadjoke.com', { headers: { Accept: 'text/plain' } })
            console.log(res.data)
```

Everytime we run this workflow all packages will be installed, so nothing is cached. For creating a cache we must add a new step before installing the dependencies. This is made with an action calles **cache**. Here we can see one example:

```yaml
name: Caching and Artifacts
on: [workflow_dispatch]

job:
  use-axios:
    runs-on: ubuntu_latest
    steps:
      - uses: actions/checkout@v3
      - name: Cache node modules
        id: cache-dependencies
        uses: actions/cache@v3
        with: 
          path: ~/.npm
          key: "npm-cache"
      - name: Display Cache Output
        run: echo '${{ toJSON(steps.cache-dependencies.outputs) }}'
      - name: Install dependencies
        run: rpm install
      - name: Use Axios
        uses: actions/github-script@v6
        with:
          script: |
            const axios = require('axios');
            const res = await axios('https://icanhazdadjoke.com', { headers: { Accept: 'text/plain' } })
            console.log(res.data)
```

First we create one step to create the cache of our libraries. This is made by the action **actions/cache**. In this extension documentation we can see the path and key needed for every case. In the example we use npm in a linux environment:

```yaml
      - name: Cache node modules
        id: cache-dependencies
        uses: actions/cache@v3
        with: 
          path: ~/.npm
          key: "npm-cache"
```

We can access the outputs of the cache step to see that everything worked fine:

```yaml
      - name: Display Cache Output
        run: echo '${{ toJSON(steps.cache-dependencies.outputs) }}'
```

This is simply made with the context.

## Updating Cache Keys Dinamically && Adding Restore Keys

We need to chenge the key everytime the dependencies change. For a JS project we have this dependencies in the file package-lock.json (if we use npm). One way to solve this is to change the static key for a dinamic one that will be created everytime we change the package-lock.json file. This can be donde with an expression, as shown in the example: 

```yaml
with:
  path: ~/.npm
  key: npm-cache-${{ hashFiles('**/package-lock.json') }}
```

**hashFiles** function creates a hash for the content of the file. With this solucion everytime the contents of the file changes, because dependencies are chenged, a new cache key is created automatically. We can improve this, for example, addind the operating system in the expression:

```yaml
with:
  path: ~/.npm
  key: ${{ runner.os }}npm-cache-${{ hashFiles('**/package-lock.json') }}
```

We can provide keys for those situations where the cache is not found with the generated key. This is donde with de **restore-keys** option., Let's see one example:

```yaml
with:
  path: ~/.npm
  key: ${{ runner.os }}npm-cache-${{ hashFiles('**/package-lock.json') }}
  restore_keys:
    ${{ runner.os }}-npm-cache-
    ${{ runner.os }}-
```

What happens when dependencies are changed in this scenario? First the workflow will look for a existing cache with the new calculated name. If it's not found then will look for a cache starting with the firt rule. In this case, a cache that starts with linux-npm-cache. If there is not a cache that matches the first rule, then will search for the second rule... Once one cache is founded the workflow will use this cache, run, and in the end create a new cache with de new name, so in next executions the new cache can be used.

Some actions control all the caching for you.

## Cache Limits & Restrictions

- Github will remove any cache not used in 7 days.
- Sizer limit es 10 GB

When we create child branches these child branches can acces the main branch caches, but not in the other direction (parent branches can't access child branches caches). And also different branches that are not related can't access caches between them.

Caches created on pull request will live only in the branch that are beeing pool. It's very important to clean this pull request caches frequently so we free space. We must have in mind the 10 Gb limit.

More information can be found in the [documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows#managing-caches). 


## Uploading & Downloading Job Artifacts

Artifact are similar to caching, but are use for different reasons. Caching is for files that don't change between workflow execution. Artifacts are used to save failes that are produced by a job (logs, test results...).

In the example we have an step to do the tests and we want to recover the report

```yaml
- name: Run Tests
  run: npm test

- name: Uploda Code Coverage Report
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: code-coverage
    path: coverage
    retention-days: 10   # by default, 90 days
```

The **upload-artifact** do the job of saving the files in the artifact. 

We can download the artifact in another step with this action:

```yaml
- name: Downlkoad Code Coverage
  uses: actions/download-artifact@v3
  with:
    name: code-coverage
    path: code-coverage-report
```

Here we downloaded the previously created artifact in a folder called **code-coverage-report**.

More documentation can be found [here](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/storing-and-sharing-data-from-a-workflow#passing-data-between-jobs-in-a-workflow).