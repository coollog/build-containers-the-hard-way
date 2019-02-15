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



Becoming a super hero is a fairly straight forward process:

```
$ give me super-powers
```

{% hint style="info" %}
 Super-powers are granted randomly so please submit an issue if you're not happy with yours.
{% endhint %}

Once you're strong enough, save the world:

```
// Ain't no code for that yet, sorry
echo 'You got to trust me on this, I saved the world'
```



