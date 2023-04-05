# gha-sync-scan-and-build
Test GHA for image syncing, scanning and building

# Features

- declarative image sync management, given source and destination
- declarative apko build management
  - with Melange integration
  - generated SBOMs
- automatic security scanning for each image synced and built
- fall back to Docker, if required

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

