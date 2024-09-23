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

github.com/actions/toolkit

