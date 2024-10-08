# Using Docker in Github Actions

## Running Jobs in Docker Containers 
We can configure our jobs and even our steps to run in a specific Docker container.

For using a specific container at job lever we use the container tag. The container is an object with different properties:
- image: we specify the image that will be used. By default you can provide any image available in dockerhub, and also an own image. In this second case a full full registry must be provided.
- credentials: in private containers we will need to provide credentials.
- env: with this key we configure any environment variable needed.
- ports: tell's the container wich ports will be available.
- volumes. Allows to create named paths, indicate directly available paths and map local paths to container paths.
- options: with this key we can configure the docker create.

```yaml
name: Docker
on: [workflow_dispatch]

jobs:
    docker-job: 
        runs-on: ubuntu-latest
        # By default uses dockerhub so you just need to chose the image and version.
        # For using another registry you must put the complete registry name
        #     for example, ghcr.io/owner/image 
        container: 
            image: node:22-alpine3.19
            # If we use a private image we will need to provide credentials
            # credentials:
            #     username:
            #     password:
            env:
                API_URL: some-url.com
            ports:
                - 80
#            volumes:
#                - vol_name:/patrh/in/container
#                - /path/to/container
#                - /path/in/host:/path/in/container
#            options: --cpus 1           # every options in docker create (except network realted ones)

        steps:
            - name: Log Node & OS Versions
              run: |
                node -v
                cat /etc/os-release
            - name: Log Env
              run: echo $API_URL
```

## Using Docker Containers in Steps

Different containers running in the same workflow are sibling containers, and share the same network.


We can specify a container inside a step with the uses key. We can override the default entrypoint of the container with de **entrypoint** keyword inside with.

```yaml
name: Docker
on: [workflow_dispatch]

jobs:
    docker-job: 
        runs-on: ubuntu-latest
        # By default uses dockerhub so you just need to chose the image and version.
        # For using another registry you must put the complete registry name
        #     for example, ghcr.io/owner/image 
        container: 
            image: node:22-alpine3.19
            # If we use a private image we will need to provide credentials
            # credentials:
            #     username:
            #     password:
            env:
                API_URL: some-url.com
            ports:
                - 80
#            volumes:
#                - vol_name:/patrh/in/container
#                - /path/to/container
#                - /path/in/host:/path/in/container
#            options: --cpus 1           # every options in docker create (except network realted ones)

        steps:
            - name: Log Node & OS Versions
              run: |
                node -v
                cat /etc/os-release
            - name: Log Env
              run: echo $API_URL
            - name: Container in a Step
              uses: docker://node:20-alpine3.20
              with:
                # overrides the entrypoint if necessary
                entrypoint: /usr/local/bin/node
                args: -p 2+3
            - name: Container in a Step
              uses: docker://node:20-alpine3.20
              with:
                args: -v             
```

In the example we have a container at job level, but in the last two steps we specified another container. For the last step no entrypoint is provided, so the default is used (in this case node is executed with the argument -v).


## Exploring Shared Networks & Volumes Between Multiple Containers

Containers used in the same job share the same network. Also working directory is shared between containers. This means that anything we create in the working directory of one container is accesible for the others containers in the same job (because they are sharing the same volume mounts). We can create files with one container, and read them, or modify them, in another one.


## Creating a Custom Docker Entrypoint Script

We can create our own entrypoint scripts in the project, and use them in the container.

1. First we create the entrypoint file. For example a bash script.
2. This file must be executable. This can be done by default linux command chmod +x filename.
3. We must do a checkout to have our files available inside of the container. For example with **actions/checkout**.
4. Inside the job we specify th new entrypoint as the file we just created. For example:

```yaml
            # First, the checkout
            - uses: actions/checkout@v3
            # now we use our own script as an entrypoint for the container
            - name: Run a bash script
              uses: docker://node:20-alpine3.20
              with:
                entrypoint: ./script.sh     # this file must be executable (chmod +x filename)
                args: "Alguna cadena rara que te he puesto porque es un ejemplo pero me he aburrio describir en ingles"
```

## Sending a Slack Message Using a Docker Container

This is an example using an specific docker image to send a message to a slack webhook. I copy the code here just for reference:

```yaml
            - name: Send a slack message
              uses: docker://technosophos/slack-notify
              env:
                SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
                SLACK_TITLE: From Github Actions
                SLACK_MESSAGE: "Actor: ${{github.actor}}, Event: ${{github.event_name}}"
                SLACK_COLOR: "#723fc4"
```
## An Overview to a Simple NodeJS Application

This is just an example of common docker info.


We will need to review this two in the future:

## Using Service Containers in Github Actions
## Publishing Docker Images Using Github Actions

https://github.com/docker/login-action
https://github.com/docker/metadata-action
https://github.com/docker/build-push-action


```yaml
name: Build & Publish Docker Image
on:
  release:
    types: [published]

jobs:
  push-to-dockerhub-and-GHCR:
    runs-on: ubuntu-latest
    # This permissions are needed for saving the package to GHCR
    permissions:
      packages: write
      contents: read
    steps:
      - uses: actions/checkout@v3
      - name: Login to Dockerhub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Login to GHCR
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}      
      - name: Extract Metadata
        id: metadata
        uses: docker/metadata-action@v4
        with:
          images: |
            alialaa17/simple-node-api
            ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      - name: Build  Publish Docker Image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.metadata.outputs.tags }}
          labels: ${{ steps.metadata.outputs.labels }}
```