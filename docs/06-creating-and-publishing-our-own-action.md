# Creating & Publishing Our Own Action

## Actions Overview
Actions are reusable units of code that we can use in our workflows.

Actions can be public (that are in a public repository and so are accesible by everybody) or private (and so it lives in our own repository). When used, actions can be referenced by commit ID (which is the safetest way of using and action), but also by branch or tag. This is donde with de 2 sign.

### Kind of actions
- Docker actions. Run in a docker container and we must provide everything we need in the container.
  - Stable environment.
  - Can be written in any language.
  - Only runs on Linux runners.
  - Not the fastest.
- Javascript actions.
  - Run directly on the runner machine.
  - Faster than Docker actions.
  - Run on Linux, macOK and Windows runners.
  - No control over environment.
  - Must be written in Javascript.
- Composite Actions. This actions simply combines multiple workflow steps in one action. This is very useful if we repeat the same steps in a lo of workflows.

## Creating a Simple JavaScript Action

We can use this toolkit for creating our action: github.com/actions/toolkit.

For creating new actions we must create a folder in our project inside .github/actions. For example, for a hello world action we can create a .github/actions/hello folder.

This folder must contain:
- A action.yaml file in which we will configure the action.
- All JS files needed.

Let's see one example:

```yaml
name: Hello World
author: VP
description: "Greet someone and record the time"
inputs:
  who_to_greet:
    description: "Who to greet"
    required: true
    default: "World"
outputs:
  time:
    description: The time of the greeting
runs:
  using: "node20"
  main: dist/index.js
  pre: setup.js
  pre-if: runner.os === "linux"
  post: cleanup.js
  post-if: runner.os == 'linux'
```

This file has different options:
- name: name or title of the action
- author: author of the action
- description: brief description
- inputs: inputs needed by the action
- outputs: outputs that will create the action
- runs: here we configure what will be executed
  - using: version of node used.
  - main: javascript file that will be executed executed
  - pre: we can set another JS file for executing before de main.
  - post: we can set another JS file for executing after finishing, for clean up and so.
  - pre-if and post-if: we can set conditions that will prevent de pre and post steps to be executed.

Let's see examples of this JS files.

```js
// require objects from github toolkit for using them in the action
const core = require("@actions/core");
const github = require("@actions/github");


try {
  // throw new Error('Some error message');

  // create a debug message, a warning and an error message
  core.debug('Debug Message');
  core.warning('Warning Message');
  core.error('Error Message');
  
  // This shows how to recover an input 
  const name = core.getInput('who_to_greet');
  console.log(`Hello ${name}`);

  // This shows how to create an output
  const time = Date();
  core.setOutput("time", time.toString());

  // We can also export variables
  core.exportVariable("HELLO_TIME", time);

  // We can even create logging roups, and use the contexts
  core.startGroup("Logging github context");
  console.log(JSON.stringify(github.context, null, 2));
  core.endGroup();

  // This shows how to control one error in the execution
} catch(error) {
  core.setFailed(error.message);
}
```

It's common to compile all the project so we don´t need the required modules in the runner machine. We can do this with @vercel/ncc.

We can call this action very easily in a workflow:

```yaml
name: Simple Action
on: [push]

jobs:
  simple-action:
    runs-on: ubuntu-latest
    steps:
      # We need to checkout the project
      - uses: actions/checkout@v3
      # This is where we call the created action. 
      - name: Simple JS Action
        id: greet
        uses: ./.github/actions/hello
        with:
          who_to_greet: Víctor
      # In this steps we use the output and environment variable created by our action
      - name: Log Greeting Time
        run: echo "${{ steps.greet.outputs.time }}"
      - name: Log ENV Var
        run: echo $HELLO_TIME
```

## Creating a Simple Docker Action

We can create Actions that are executed inside a Docker container. In this case some things change:

Firts, let's see the the **action.yaml** file:

```yaml
name: Hello World
author: VP
description: "Greet someone and record the time"
inputs:
  who_to_greet:
    description: "Who to greet"
#    required: true
    default: "World"
outputs:
  time:
    description: The time of the greeting
runs:
  using: "docker"
  # A remote image can be used
  # image: 'docker://node:22-alpine3.19'
  # Also a local one
  image: 'Dockerfile'
  # Override image defined in the local Dockerfile
  # entrypoint: 
  # Override the command of the dockerfile
  args:
    # use an exrepssion for getting the input
    - ${{ inputs.who_to_greet }}
  env:
    HELLO: WORLD
  post-entrypoint: "/cleanup.sh"
  post-if: runner.os == 'linux'
```

Some changes must be commented:

In the **runs** object we find:
- **image**. We can use a docker hub image or a local one (created by a Dockerfile).
- **args**. We can use args for providing the inputs of the action. In this case we use an expression for providing the first arg.
- **env**. We can also provide environment variables as another way to provide inputs to the action.
- **post-entrypoint** and **post-if** are executed just before de action finish.

This action is called as you can see in the next workflow yaml file:

```yaml
name: Simple Action
on: [push]

jobs:
  simple-action:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # Here is the call
      - name: Simple Docker Action
        id: greet
        uses: ./.github/actions/hello-docker
        with:
          who_to_greet: Víctor
      - name: Log Greeting Time
        run: echo "${{ steps.greet.outputs.time }}"
      - name: Log ENV Var
        run: echo $HELLO_TIME
```

The code, in the example, is a sh script. Let's see it and analize:

```sh
#!bin/sh

# Create a debug message
echo "::debug::Running entrypoint.sh"

# This is a warning message
echo "::warning::This is a warning"

# This is an error message
echo "::error::This is an error message"

# Here we use the first argument
echo "Hello $1"

# We can access the inputs as environment variables with the format INPUT_NAME_OF_INPUT
echo "INPUT_WHO_TO_GREET: $INPUT_WHO_TO_GREET"

# Another env variable example
echo "HELLO: $HELLO"

time=$(date)

# This is an output example
echo "time=$time" >> $GITHUB_OUTPUT

# And this is an output environment variable
echo "HELLO_TIME=$time" >> $GITHUB_ENV
```
For failing the action in this case you can use **exit 1**.
Finally we will need a Dockerfile.

```dockerfile
FROM alpine:3.19
COPY cleanup.sh /cleanup.sh
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT [ "/entrypoint.sh" ] 
CMD [ "World" ]
```

We configure the *entrypoint.sh* file as the entrypoint of the container.

## Composite Actions

