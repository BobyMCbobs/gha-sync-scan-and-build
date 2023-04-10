# gha-sync-scan-and-build

> Test GHA for image syncing, scanning and building

# Purpose

- sync to vendor images
- build container images
- run security scans again each destination image in configuration
- exercise capabilities

The repo is mostly concerned with base images or images used in build processes.

# Features

- declarative image sync management, given source and destination
- declarative apko build management
  - with Melange integration
  - generated SBOMs
- automatic security scanning for each image synced and built
- fall back to Docker, if required
- automatic trigger of builds, sync and scan every week

# Layout

- `config.yaml`: define configuration about runtime
- `images/NAME/{images.yaml|Dockerfile,*}`: images to build configurations
- `.github/workflows/{sync,scan,build}.yml`: lifecycle actions

# Usage

## Sync an image

Images are synced by specifying source and destinations like this
```yaml
sync:
  - source: docker.io/alpine:latest
    destination: ghcr.io/somecoolorg/images/alpine:latest
```

Images will only be synced if the digest of the source and destination don't match

## Build with apko

Images for building are specified with the source being an apko formatted YAML file and an image destination
```yaml
build:
  - source: ./images/acoolthing/image.yaml
    destination: ghcr.io/somecoolorg/images/acoolthing:latest
```

with `./images/acoolthing/image.yaml` being something like this

```yaml
contents:
  repositories:
    - https://dl-cdn.alpinelinux.org/alpine/v3.17/main
    - https://dl-cdn.alpinelinux.org/alpine/v3.17/community
  packages:
    - alpine-base
    - busybox
    - ca-certificates-bundle

entrypoint:
  command: /bin/sh -c

archs:
- x86_64
- aarch64
```

## Build with apko and Melange

Specify a source, destination and as many melangeConfigs
```yaml
build:
  - source: ./images/coolthing/image.yaml
    destination: ghcr.io/somecoolorg/images/coolthing:latest
    melangeConfigs:
      - ./images/coolthing/pkg-hello.yaml
```

with each melange config being a path to the package to build.
An entry in the apko YAML must be set `contentx.repositories[last index]` to `@local /github/workspace/packages`, then packages can be installed with `NAME@local`; like this apko configuration

```yaml
contents:
  repositories:
    - https://dl-cdn.alpinelinux.org/alpine/v3.17/main
    - https://dl-cdn.alpinelinux.org/alpine/v3.17/community
    - '@local /github/workspace/packages'
  packages:
    - alpine-base
    - busybox
    - ca-certificates-bundle
    - hello@local

entrypoint:
  command: /bin/sh -c

archs:
- x86_64
```

## Build with Docker

Only use this if you have to. It is better to build with apko.

Images can be built with Docker like this
```yaml
build:
  - source: ./images/oldschool/Dockerfile
    destination: ghcr.io/somecoolorg/images/oldschool:latest
    dockerOptions:
      context: ./images/oldschool
```

# Tooling

| Name    | Description                                                                   | Links                                                              | Related/Alternatives                     |
|---------|-------------------------------------------------------------------------------|--------------------------------------------------------------------|------------------------------------------|
| crane   | an officially supported cli container registry tool from Google               | https://github.com/google/go-containerregistry/tree/main/cmd/crane | skopeo                                   |
| yq      | a cli YAML parser                                                             | https://github.com/mikefarah/yq                                    | jq                                       |
| cosign  | a cli container image artifact signing utility by Sigstore (Linux Foundation) | https://github.com/sigstore/cosign                                 | ...                                      |
| melange | a cli Alpine APK package declarative builder supported by Chainguard          | https://github.com/chainguard-dev/melange                          | ...                                      |
| apko    | a cli tool for declaratively building Alpine based container images           | https://github.com/chainguard-dev/apko                             | ko (https://ko.build - Linux Foundation) |
| docker  | a container ecosystem, primarily for development                              | https://docker.io                                                  | podman                                   |
| trivy   | a container image scanner                                                     | https://github.com/aquasecurity/trivy                              | clair                                    |
