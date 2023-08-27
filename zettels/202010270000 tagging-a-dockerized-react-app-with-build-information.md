---
title: "How to add build information to a dockerized React app? (Docker Hub)"
slug: "tagging-a-dockerized-react-app-with-build-information"
date: "2020-10-27"
summary: ""
references: 
  - '[[202103170000 configuring-traefik-on-kubernetes]]'
  - '[[201801220000 -net-core-application-not-logging-output-to-docker]]'
---

#software-development #devops

To help myself stay sane in a world of continuous deployment and be able to effectively debug strange situations I wanted to tag every version of my app with version information, such as the commit hash, commit message, build date and more. My setup is one where I run an react app created using the `create-react-app` CLI tool, deployed within a Docker container which is built using Docker Hub (using the Autobuild feature).


To be able to tag information there are two things you need to do:

1. Configure a custom build step
2. Modify the `Dockerfile` to include environment variables


## A custom build step

The addition of a custom build step is a seemingly undocumented feature with Docker Hub which I only found out due to [this response on a GitHub issue](https://github.com/docker/hub-feedback/issues/508#issuecomment-222520720). The gist is that you'd need to create a `hooks/build` file from the same folder as your `Dockerfile` in which you can specify the way your docker container is built. This overrides the way your container build is configured within Docker Hub, but can be made in a way that it cooperates;

```bash
#!/usr/bin/env sh
docker build -t $IMAGE_NAME -f $DOCKERFILE_PATH .
```

For a more complete and minimal example regarding the possibilities with custom build steps; check out [this repository on GitHub (thibaultdelor/testAutobuildHooks)](https://github.com/thibaultdelor/testAutobuildHooks).

It's also this build step where you can define additional arguments to be available during the build step. Something which is not the case by default. Here are some examples of arguments I have added;

- `--build-arg REACT_APP_BUILD_DATE=``date -u +"%Y-%m-%dT%H:%M:%SZ"```
- `--build-arg REACT_APP_COMMIT="$SOURCE_COMMIT"`
- `--build-arg REACT_APP_COMMIT_MESSAGE="$COMMIT_MSG"`

Note that as a precautionary measure I have encapsulated the environment variables within a double quote (`"`), which is required to properly deal with potential spaces in the values represented by an environment variable.

For a list of environment variables exposed by Docker Hub, [look here](https://docs.docker.com/docker-hub/builds/advanced/) and [here for some undocumented ones](https://github.com/docker/hub-feedback/issues/508#issuecomment-240616319).


## Modifying the Dockerfile

In order to give your React app access to the environment variables there's still the need to modify the Dockerfile. For each variable you will need two additional lines;

```
ARG REACT_APP_BUILD_DATE
ENV REACT_APP_BUILD_DATE $REACT_APP_BUILD_DATE
```

The first one defines what variables can be defined using the `--build-arg` arguments on the `docker build` command. The second line makes these variables available to the app during runtime, from which the node environment can pick these up. For more in depth details on the differences between `ARG` and `ENV`, dive into [this StackOverflow thread](https://stackoverflow.com/a/41919137/1720761).

