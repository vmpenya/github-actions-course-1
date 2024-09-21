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
## Exploring Shared Networks & Volumes Between Multiple Containers
## Creating a Custom Docker Entrypoint Script
## Sending a Slask Message Using a Docker Container
## An Overview to a Simple NodeJS Application
## Using Service Containers in Github Actions
## Publishing Docker Images Using Github Actions