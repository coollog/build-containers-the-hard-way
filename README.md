# Building Containers the Hard Way

_Like_ [_Kubernetes the Hard Way_](https://github.com/kelseyhightower/kubernetes-the-hard-way)_, but for building containers._

This guide is geared towards anyone interested in learning the low-level details regarding how to build containers. This guide is _not_ intended for those who wish to learn to build container images with high-level tools like Docker.

At the end of this guide, you should understand the internals of a container image, how to construct a container image, and how to push a container image to a Docker container registry piece-by-piece.

## What is a container image?

A _container_ is a way of executing processes with isolation provided by 3 Linux technologies - `chroot`, `namespaces`, and `cgroups`.

`chroot` sets the directory to use as the root \(`/`\) of the file system the process sees, instead of the actual file system root.

`namespaces` group resources \(like network and process IDs\) so that only processes within a namespace can see the resources of that namespace.

`cgroups` set CPU and memory limits for processes.

The main goal of containers is to allow processes to run in isolation from other processes both on the file system level and on the resource utilization level.

A _container image_ a way to package up an application along with its runtime dependencies so that the package can run as a _container_. A container image is simply a directory of files along with metadata about how to run the container.

[Containers from Scratch](https://ericchiang.github.io/post/containers-from-scratch/) is a good article explaining how to run your own containers using simple Linux commands.

### Build a Docker container image with Docker

Docker is the most popular tool for working with container images. It has built-in support for building and running containers. Docker defines its own scripting language for defining how to build a container image.

For example, this Dockerfile builds a simple Docker image that serves any static files in the current working directory as webpages:

{% code-tabs %}
{% code-tabs-item title="Dockerfile" %}
```text
FROM python
COPY . /public
WORKDIR /public
ENTRYPOINT ["python3", "-m", "http.server"]
```
{% endcode-tabs-item %}
{% endcode-tabs %}

To build the Docker image, save the `Dockerfile` in the current working directory and tell Docker build it:

```bash
$ docker build .
```

```text
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM python
 ---> 338b34a7555c
Step 2/4 : COPY . /public
 ---> edcd805ec657
Step 3/4 : WORKDIR /public
 ---> Running in cd5f924a79fe
Removing intermediate container cd5f924a79fe
 ---> bba7b6ca42fc
Step 4/4 : ENTRYPOINT ["python", "-m", "http.server"]
 ---> Running in 4198534c1c9c
Removing intermediate container 4198534c1c9c
 ---> 5550043a7340
Successfully built 5550043a7340
```

The logs of the `docker build` command clearly show how Docker executes the build based on our Dockerfile.

The `docker build .` command tells Docker to use the local current working directory \(`.`\) as the _Docker context_. The Docker context is sent to the _Docker daemon_. The `docker` CLI is the client that you use to send commands and data to the Docker daemon. The Docker daemon stores and runs containers.

The commands in the Dockerfile are run in the given order. The Docker daemon runs a container to execute each command and generates a new container image at the end of each step.

The `FROM python` step tells Docker to build the new container starting with the [python container image from Docker Hub](https://hub.docker.com/_/python) as the base.

`COPY . /public` copies all the files in the Docker context into the `/public` directory on our container image. This includes all the files in our local current working directory that was sent over to the daemon as the Docker context.

`WORKDIR /public` sets the working directory for the container image we are building, so that when the container image is run, the current working directory for the process is `/public`.

The container that was created by the `WORKDIR` command is removed. Steps that do not change the container file system only modify the _container configuration_ and are removed. The container configuration is metadata that describes how to run the container entrypoint process.

`ENTRYPOINT ...` sets the command to run when the container is run, which, in this case, runs an HTTP server that serves all the files in `/public`. So, for example, if we had HTML files in our local current working directory, those would be served by this container.

The final image built has ID `5550043a7340`, which can be used in subsequent `docker` commands.

This container can then be run with:

```bash
$ docker run -p 8000:8000 5550043a7340
```

This runs the container and forwards the port to `localhost:8000`. If you had, say, an `index.html` in your local current working directory, going to `localhost:8000` would serve the contents of `index.html`.

Another common Dockerfile instruction is `RUN <command>`, which executes the `<command>` using whichever shell is present in the container, creating a new container following the result of executing that command.

### Advanced Docker Builds

Docker has its own caching mechanism that helps Dockerfile-based builds run faster. Essentially, Docker caches the container image generated after each Dockerfile step. If it determines that a step has not changed, it will use the container image cached for that step rather than run that step again. However, if a step changes, it will invalidate the cache for that step and all steps afterwards. The way the caching mechanism works in Docker merits some tips for writing a more efficient Dockerfile.

#### Frequently changing steps go last

Dockerfile builds can take quite a while, especially with heavy steps that download dependencies or compile your application. Since any step that changes invalidates steps afterwards, better Dockerfiles tend to place more frequently-changing steps last. Heavy steps that do not change much tend to be front-loaded so that subsequent builds can just use the cached result.

#### **Keep the number of steps minimal**

Since each step generates a separate cached container image, steps should be kept minimal to reduce the number of layers \(each cached step has its own overhead\). For example, a group of `RUN`s can usually be combined:

_Bad:_

```
RUN mkdir mydirectory
RUN touch mydirectory/myexecutable
RUN chmod +x mydirectory/myexecutable
```

_Better:_

```text
RUN mkdir mydirectory && \
    touch mydirectory/myexecutable && \
    chmod +x mydirectory/myexecutable
```

However, frequently-changing steps should not be combined with infrequently-changing heavy steps - rather, an efficient Dockerfile should try to separate these as much as possible to avoid running infrequently-changing steps.

#### **Use multistage builds for more flexibility than just a linear Dockerfile build**

> TODO

## **Container image format**

Container images can be built in a specific format. Different runtimes may support different formats. The most common formats are the Docker image format and Open Container Initiative \(OCI\) format.

> TODO: Support for formats, like in Kubernetes \(CRI\)

### Layers

As mentioned, a container image is simply a directory of files \(along with metadata\). In the image format, this directory of files is split up into layers that are compatible with AUFS.

Layers are independent, but are composed together with a defined order in the image format. The order of the layers in the image matters for files that may exist in multiple layers. The contents of a file that appears in a later layer replaces the contents of the same file in a prior layer. A file can also be deleted in a later layer by prepending its filename with `.wh.`.

Each layer is archived into a single tarball. In the Docker image format, this tarball must be compressed with GZIP compression.

Caveat: when run, Docker shares common layers between containers and creates another overlay layer for file system writes - any writes to existing files would be a copy-on-write. This overlay layer is deleted when the container is removed.

> TODO: Show some bash commands that can generate a layer

### Digests

BLOBs \(like layer archives\) in the image format are identified by their digests. These digests are generated by hashing the BLOB contents. The digest for a layer would be the hash of the compressed tarball archive. The hashing algorithm to use for Docker image format is SHA-256, but [OCI format supports SHA-512 as well](https://github.com/opencontainers/image-spec/blob/master/descriptor.md#registered-algorithms). The benefit of identifying BLOBs is that they become content-addressable. Uniqueness is provided by the collision-resistance of SHA-256.

> TODO: Add example descriptor digest

### Container configuration

The metadata for a container image is represented by its container configuration. This container configuration contains mainly information about how the container should be run. This includes fields such as for architecture \(GOARCH\) and OS \(GOOS\), exposed ports, volume mounts, and environment variables.

The most important of the runtime configuration found in the container configuration is the entrypoint, which is the command that is run when the container is run. This essentially defines the entrypoint process the container is isolating.

Note that all runtime configuration in the container configuration exist as defaults. When actually running the container either with Docker or with Kubernetes, any/all of these runtime configuration can be overridden.

> TODO: Might want to talk about reasonings for burning runtime config into an image vs. defining in a runtime spec

> TODO: Show example container configuration and running with Docker/Kubernetes and effective runtime configuration

