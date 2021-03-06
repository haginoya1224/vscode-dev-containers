# Build and image generation for vscode-dev-containers

This folder contains scripts to build and push images into the Microsoft Container Registry (MCR) from this repository, generate or modify any associated content to use the built image, track dependencies, and create an npm package with the result that is shipped in the VS Code Remote - Containers extension.

## Build CLI

The Node.js based build CLI (`build/vsdc`) has commands to:

1. Build and push to a repository: `build/vsdc push`
2. Build, push, and npm package assets that are modified as described above: `build/vsdc package`
3. Generate cgmanifest.json: `build/vsdc cg`

Run with the `--help` option to see inputs.

## Setting up a container to be built

> **Note:** Only Microsoft VS Code team members can currently onboard an image to this process since it requires access the Microsoft Container Registry. However, if you have your own pre-built image or build process, you can simply reference it directly in you contributed container.

Image build/push to MCR is managed using config in `definition-build.json` files that are located in the container definition folder. The config lets you set dependencies between definitions and map actual image tags to multiple definitions. So, the steps to onboard an image are:

1. Create a `definition-build.json` file
2. Update the `vscode` config files for MCR as appropriate (MS internal only).

Let's run through the `definition-build.json` file.

### The build property

The `build` property defines how the definition maps to image tags and what type of "stub" should be used. Consider the Debian 9 [definition-build.json](../containers/debian-9-git/definition-build.json) file.

```json
"build": {
    "rootDistro": "debian",
    "latest": true,
    "tags": [
        "base:${VERSION}-debian-9",
        "base:${VERSION}-stretch"
    ]
}
```

The **`rootDistro`** property can be `debian`, `alpine`, or `redhat` currently. Ubuntu-based containers should use `debian`.

The **`latest`** and **`tags`** properties affect how tags are applied. For example, here is how several dev container folders map:

```text
debian-9-git => mcr.microsoft.com/vscode/devcontainers/base:debian-9
alpine-3.10-git => mcr.microsoft.com/vscode/devcontainers/base:alpine-3.10
ubnutu-18.04-git => mcr.microsoft.com/vscode/devcontainers/base:ubuntu-18.04
javascript-node-12 => mcr.microsoft.com/vscode/devcontainers/javascript-node:12
    typescript-node-10 => mcr.microsoft.com/vscode/devcontainers/typescript-node:12
javascript-node-10 => mcr.microsoft.com/vscode/devcontainers/javascript-node:10
    typescript-node-10 => mcr.microsoft.com/vscode/devcontainers/typescript-node:10
```

This results in just three "repositories" in the registry much like you would see for other images in Docker Hub.

- mcr.microsoft.com/vscode/devcontainers/base
- mcr.microsoft.com/vscode/devcontainers/javascript-node
- mcr.microsoft.com/vscode/devcontainers/typescript-node

The package version is then automatically added to these various tags in the `${VERSION}` location for an item in the `tags` property array as a part of the release. For example, release 0.40.0 would result in:

mcr.microsoft.com/vscode/devcontainers/base

- 0.40.0-debian-9
- 0.40-debian-9
- 0-debian-9
- debian-9 <= Equivalent of latest for debian-9 specifically
- 0.40.0-stretch
- 0.40-stretch
- 0-stretch
- stretch <= Equivalent of latest for stretch specifically

In this case, Debian is also the one that is used for `latest` for the `base` repository, so that tag gets applied too.  If you ran only the Alpine or Ubuntu versions, the latest tag would not update.

> **NOTE:** The version number used for this repository should be kept in sync with the VS Code Remote - Containers extension to make it easy for developers to find.

There's a special "dev" version that can be used to build master on CI - I ended up needing this to test and others would if they base an image off of one of the MCR images.  e.g. `dev-debian-9`.

Finally, there is a **`parent`** property that can be used to specify if the container depends an image created as a part of another container build. For example, `typescript-node-10` uses the image from `javascript-node-10` and therefore includes the following:

```json
"build" {
    "parent": "javascript-node-10"
}
```

### The dependencies property

The dependencies property is used for dependency and version tracking. Consider the Debian 9 [definition-build.json](../containers/debian-9-git/definition-build.json) file.

```json
"dependencies": {
    "image": "debian:9",
    "imageLink": "https://hub.docker.com/_/debian",
    "debian": [
        "apt-utils",
        "..."
    ],
    "manual": [
        {
            "Component": {
                "Type": "git",
                "git": {
                    "Name": "Oh My Zsh!",
                    "repositoryUrl": "https://github.com/robbyrussell/oh-my-zsh",
                    "commitHash": "c130aadb6a66aa680a322c08d87ad773316f713d"
                }
            }
        }
    ]
}
```

The **`image`** property is the actual base Docker image either in Docker Hub or MCR. **`imageLink`** is then the link to a dscription of the image.

Following this is a list of libraries installed in the image by its Dockerfile. The following package types are currently supported:

- `debian` - Debian packages (apt-get)
- `ubuntu` - Ubuntu packages (apt-get)
- `alpine` - Alpine Linux packages (apk)
- `npm` - npmjs.com packages installed using npm or yarn
- `manual` - Useful for other types of registrations that do not have a specific type, like `git` in the example above.

For everything but `manual`, the image that is generated is started so the package versions can be queried automatically.

## Details

Currently the vscode-dev-containers repo contains pure Dockerfiles that need to be built when used. While this works well from the perspective of providing samples, some of the images install a significant number of runtimes or tools which can take a long time to build.

We can resolve this by pre-building some of these images, but in-so-doing we want to make sure we:

1. Ensure vscode-dev-containers continues to be a good source of samples
2. Improve the performance using the most popular (or in some cases slowest to build) container images
3. Make it easy for users to add additional software to the images
4. Make it easy for contributors to build, customize, and contribute new definitions

We won't be able to build all images in the repository or publish them under a Microsoft registry, but we would want to allow contributors to build their own images for contributions if they wanted to do so by keeping the ability to use a stand alone Dockerfile, image reference, or docker-compose file like today.

## Image generation for vscode-dev-containers

In order to meet the above requirements, this first phase keeps the way the dev containers deployed as it is now - an npm package. Future phases could introduce a formal registry, but this is overkill current state.

### Versioning

At this phase, versioning of the image can be tied to the release of the npm package. For example, as of this writing, the current version is 0.35.0. All images that are generated would then inherit this version.

We would follow semver rules for bumping up the repository name - any fix that needs to be deployed immediately would be "fix" level semver bump. When released as latest, the image would be tagged as is done in other Docker images. Using the 0.35.0 example, new images would be tagged as:

- 0
- 0.35
- 0.35.0
- latest

In the unlikely event a break fix needed to be deployed and not tagged latest, we would have a facility to tag as follows:

- 0.35
- 0.35.0

This has a few advantages:

1. Tags are generated for each version of the repository that is cut. This allows us to refer directly to the exact version of the source code used to build the container image to avoid confusion. e.g. `https://github.com/microsoft/vscode-dev-containers/tree/v0.35.0/containers/javascript-node-8`
2. Similarly, as containers are deprecated and removed from the repository, you can still refer back to the container source and README.
3. Upstream changes that break existing images can be handled as needed.
4. Developers can opt to use the image tag 0.35 to get the latest break fix version if desired or 0 to always get the latest non-breaking update.

This scheme will work in the near term where we do not have a large number of dev containers in the repository, but over time will not scale. However, to realistically support independent versioning, we'd need to build out a registry service - which may or may not be required. This will be evaluated at a later date.

### Deprecation of container definitions

The versioning scheme above allows us to version dev containers and reference them even when they are removed from `master`. To keep the number of containers in `master` reasonable, we would deprecate and remove containers under the following scenarios:

1. It refers to a runtime that is no longer supported - e.g. Node.js 8 is out of support as of the end of 2019, so we would deprecate `javascript-node-8`. Until that point, we would have containers for node 8, 10, and 12 (which just went LTS).
2. The container is not used enough to maintain and has broken.
3. The container refers to a product that has been deprecated.
4. The container was contributed by a 3rd party, has issues, and the 3rd party is not responsive.

Since the images would continue to exist after this point and the source code is available under the version label, we can safely remove the containers from master without impacting customers.

### Release process and the contents of the npm package

When a release is cut, there are a few things that need to happen. One is obviously releasing the appropriate image. However, to continue to help customers understand how to customize their images, we would want to reference a user modifiable "stub" Dockerfile instead of an image directly. This also is important to help deal with shortcomings and workarounds that require something like specifying a build argument. For example:

```Dockerfile
FROM mcr.microsoft.com/vscode/devcontainer/javascript-node:0-10

ARG USER_UID=1000
ARG USER_GID=$USER_UID
ENV DEBIAN_FRONTEND=noninteractive

# [Optional] Update UID/GID if needed and install additional software
RUN if [ "$USER_GID" != "1000" ] || [ "$USER_UID" != "1000" ]; then \
        if [ "$USER_GID" != "1000" ]; then sudo groupmod 1000 --gid $USER_GID; fi \
        && if [ "$USER_UID" != "1000" ]; then sudo usermod --uid $USER_UID 1000; fi; \
    fi \
    #
    # ***************************************************************
    # * Add steps for installing any other needed dependencies here *
    # ***************************************************************
    #
    # Clean up
    && sudo apt-get autoremove -y \
    && sudo apt-get clean -y \
    && sudo rm -rf /var/lib/apt/lists/*

# Uncomment to default to non-root user
# USER $USER_UID

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog
```

This retains its value as a sample but minimizes the number of actual build steps. This template would **evolve over time as new features are added**. For example, the above Dockerfile includes a way to make sure the user's GID/UID matches the local operating system - which is critical for Linux users. However, if we introduce a `user` concept in `devcontainer.json`, the template would no longer need to include this part of the file.

Referencing The MAJOR version in this Dockerfile allows us to push fixes or upstream updates that do not materially change the definition.

### Repository contents

Consequently, this user stub Dockerfile needs to be versioned with the `devcontainer.json` file and can technically version independently of the actual main Dockerfile and image. Given this tie, it makes sense to keep this file with `devcontainer.json` in the repository. The repository therefore would could contain:

```text
📁 .devcontainer
     📄 base.Dockerfile
     📄 devcontainer.json
     📄 Dockerfile
📁 test-project
📄 README.md
```

Here, `devcontainer.json` points to `base.Dockerfile`, but this is the Dockerfile used to generate the actual image rather than the stub Dockerfile. The stub that references the image is in `base.Dockerfile`.  To make things easy, we can also automatically generate this stub at release time if only a Dockerfile is present.

Testing, then, is as simple as it is now - open the folder in `vscode-dev-containers` in a container and edit / test as required. Anyone simply copying the folder contents then gets a fully working version of the container even if in-flight and there is no image for it yet.

In the vscode-dev-containers repo itself, the `FROM` statement in `Dockerfile` would always point to `latest` or `dev` since it what is in master may not have even been released yet. This would get dynamically updated as a part of the release process - which we will cover next.

```Dockerfile
FROM mcr.microsoft.com/vs/devcontainer/javascript-node:dev-10
```

#### Automated updates of other Dockerfiles

The process also automatically swaps out referenced MCR images for MAJOR versions of built images in any Dockerfile that is added to the package. This allows us to push break fix and or security patches as break fix releases and people will get them. The build supports an option to not update `latest` and MAJOR versions, so we can also rev old MAJOR versions if we have to, but normally we'd roll forward instead.

#### Common scripts

Another problem the build solves is mass updates - there's a set of things we want in every image and right now it requires ~54 changes to add things. With this new process, images use a tagged version of scripts in `script-library`. The build generates a SHA for script so they can be safely used in Dockerfiles that are not built into images while still allowing people to just grab `.devcontainer` from master and use it if they prefer.

#### Release process

When a release is cut, the contents of vscode-dev-containers repo would staged. The build process would then do the following for the appropriate dev containers:

1. Build an image using the `base.Dockerfile` and push it to a container registry with the appropriate version tags. If no `base.Dockerfile` is found, `Dockerfile` is renamed to `base.Dockerfile` and used instead.

2. After the image is built and pushed, `base.Dockerfile` is deleted from staging.

3. Next, `Dockerfile` is updated to point to the correct MAJOR version. If no Dockerfile is found, a stub is used based on the root image (Alpine vs Debian).

    ```Dockerfile
    # For information on the contents of the image referenced below, see the Dockerfile at
    # https://github.com/microsoft/vscode-dev-containers/tree/v0.35.0/containers/javascript-node-10/.devcontainer/base.Dockerfile
    FROM mcr.microsoft.com/vs/devcontainer/javascript-node:0-10
    ```

4. `devcontainer.json` is updated to point to `Dockerfile` instead of `base.Dockerfile` (if required) and add a comment to the source code README for this specific version.

    ```json
    // For format details, see https://aka.ms/vscode-remote/devcontainer.json or the definition README at
    // https://github.com/microsoft/vscode-dev-containers/tree/v0.35.0/containers/javascript-node-10
    {
        "name": "Node.js 10",
        "dockerFile": "Dockerfile",
        "extensions": [
            "dbaeumer.vscode-eslint"
        ]
    }
    ```

These modified contents are then archived in an npm package exactly as they are today and shipped with the extension (and over time we could dynamically update this between extension releases).

```text
📁 .devcontainer
     📄 devcontainer.json
     📄 Dockerfile
```
