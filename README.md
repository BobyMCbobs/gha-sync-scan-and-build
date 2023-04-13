# gha-sync-scan-and-build

> Test GHA for image syncing, scanning and building

# Purpose

- sync to vendor images
- build container images
- run security scans again each destination image in configuration
- separate implementation from the data
  - implementation can be swapped out underneath, if need be and maintain the function

The repo is mostly concerned with base images or images used in build processes.

# Features

- declarative image sync management, given source and destination
- declarative apko build management
  - with Melange (Alpine APK building) integration
  - generated SBOMs
  - container image signing
- automatic security scanning for each image synced and built
- multiple build modes
  - apko
  - apko + melange
  - docker (fallback)
- automatic trigger of builds, sync and scan every week

# Layout

the structure of the repo is as follows:

- `config.yaml`: define configuration about runtime
- `images/NAME/{images.yaml|Dockerfile,*}`: images to build configurations
- `.github/workflows/{sync,scan,build}.yml`: lifecycle actions

# Set up

1. specify images for syncing and building in the respective _sync_ and _build_ fields
2. set the image push GitHub secrets of `QUAY_USERNAME` and `QUAY_ROBOT_TOKEN`
3. generate a signing keypair with `cosign generate-key-pair`
4. set signing key and password as GitHub secrets with `COSIGN_PRIVATE_KEY` and `COSIGN_PASSWORD` respectively
5. commit the _cosign.pub_ public key to the repo's root

# Usage

## Sync an image

Images are synced by specifying source and destinations like this

```yaml
sync:
  - source: docker.io/alpine:latest
    destination: ghcr.io/somecoolorg/images/alpine:latest
```

Images will only be synced if the digest of the source and destination don't match or if the `sync[].always` key is set to `true`

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
An entry in the apko YAML must be set `contents.repositories[last index]` to `@local /github/workspace/packages`, then packages can be installed with `NAME@local`; like this apko configuration

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

# Tips

## What is an image ref?

IMAGE_REF refers to an image reference in the format of either
- ghcr.io/somecoolorg/images/hello:latest (just a tag)
- ghcr.io/somecoolorg/images/hello@sha256:a61743b19423a01827ba68a1ec81a09d04f84bc69848303905ecbc73824fb88b (just a digest)
- ghcr.io/somecoolorg/images/hello:latest@sha256:a61743b19423a01827ba68a1ec81a09d04f84bc69848303905ecbc73824fb88b (a tag and a digest)

## Obtaining a digest

the digest can be obtained through

```shell
crane digest IMAGE_REF
```

to use it, it can be included when an image ref is formatted like the examples above in _What is an image ref?_.

## Tagging built images

determine the digest of the image, for example given the image is `ghcr.io/somecoolorg/images/hello:latest`
``` shell
crane digest ghcr.io/somecoolorg/images/hello:latest
```

the digest might be `sha256:a61743b19423a01827ba68a1ec81a09d04f84bc69848303905ecbc73824fb88b`.

add a new sync image to the sync key like
```yaml
  - source: ghcr.io/somecoolorg/images/hello@sha256:a61743b19423a01827ba68a1ec81a09d04f84bc69848303905ecbc73824fb88b
    destination: ghcr.io/somecoolorg/images/hello:some-cool-tag
```

## Verifying images

Images are able to be verified through

```shell
cosign verify --key cosign.pub IMAGE_REF
```

## See the tree of attached signatures and SBOMs

produces a nice and readible tree of signatures, attestations and SBOMs related to the IMAGE_REF
```shell
cosign tree IMAGE_REF
```

## View the SBOM in the attestation

verify the attestation
```shell
cosign verify-attestation --key cosign.pub IMAGE_REF
```

since SBOMs are the predicate of a signed attestation instead of just uploaded, it requires an extra layer to retrieve their content
```shell
cosign verify-attestation --key cosign.pub IMAGE_REF | jq -r .payload | base64 -d | jq -r .predicate.Data
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
| syft    | a cli tool to generate sboms based on container images and filesystems        | https://github.com/anchore/syft                                    |                                          |

# Patterns for discussion

> semi-related topics

- require signed container images for use in production
  - https://docs.sigstore.dev/cosign/sign
  - https://kyverno.io/policies/other/verify_image
  - https://docs.sigstore.dev/policy-controller/overview
- use distroless container images as base images
  - don't ship package managers into production
  - https://github.com/chainguard-images
- run containers as non-root users
  - limit workload privilege
  - https://github.com/chainguard-dev/apko/blob/main/docs/apko_file.md#accounts-top-level-element
  - set `pod.spec.containers.securityContext.runAsUser`; https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod
  - `docker run --user 10000 ...` https://docs.docker.com/engine/reference/commandline/run
  - https://docs.aws.amazon.com/config/latest/developerguide/ecs-task-definition-nonroot-user.html
- use a static tag with digest instead of latest
  - ensure that you are using the version of the image that you expect
  - don't use e.g: `alpine:latest`, instead use an immutable image reference like `alpine:3.17.3@sha256:b6ca290b6b4cdcca5b3db3ffa338ee0285c11744b4a6abaa9627746ee3291d8d` to ensure an expected version is always used
  - version full digests can be manually resolved for consumption using `crane digest alpine:3.17.3` (for example)
  - when using systems like Knative, these digests are automatically resolved
- deploy containers with a read-only root filesystem
  - https://docs.aws.amazon.com/config/latest/developerguide/ecs-containers-readonly-access.html
  - `docker run --read-only ...`; https://docs.docker.com/engine/reference/commandline/run
  - set `pod.spec.containers.securityContext.readOnlyRootFilesystem` to `true`; https://kubernetes.io/docs/tasks/configure-pod-container/security-context
- build Go apps with [ko](https://ko.build)
  - https://ko.build/advanced/migrating-from-dockerfile
  
# Good reads

- https://docs.sigstore.dev/history
