.. title: Docker Images Under the Hood
.. slug: docker-images-under-the-hood
.. date: 2020-12-01 10:07:00 UTC+05:30
.. tags: Docker, Containers
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status: draft
.. summary: This post looks at the details of the docker images and how they work with containers.

# Introduction

Recently, I started looking at containers (again) more from the infrastructure developer's point of view rather than an application developer's point of view. What this concretely means is - I am trying to figure out how the containers, specifically in the context of Kubernetes work. While, I understood that at a high-level, containers make use of certain Linux kernel features, like `cgroups` and `namespaces`, from the infra developer's point of view, there are a lot of details that need to be understood. One piece of that puzzle is container images. Often container images have been synonymous with Docker Images since docker has been the de facto standard for managing containers in the Kubernetes world and even otherwise. Part of the reason has been, recently we started using docker for deployment for one of the projects primarily to support multi-tenancy in the application.

In the present blog post, I am trying to compile my understanding of Docker Images, so that it's easier to refer to again in future, if required. While this post will provide an overview of what docker images are, it does not go into great details about how they are created. The blog post is more about - how docker engine works with docker repository and a root file system is provided for the container. The motivation for this comes from the fact that - what if I have to 'run a container by hand' using a docker image?

# Docker Images - An Overview

A docker image provides the root file system for the container. At a high level, it's basically a collection of tar files, organized as layers, which can be 'overlay'ed over each other. Starting with base layer and subsequent layers added as a `diff` to the previous layer. For the rest of the discussion, we'll be exploring the following image `fedora:latest` from the docker registry.

Let's start looking at what a docker image is using the docker cli.

A list of locally available docker images can be obtained using the following command -

```shell

# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
fedora              latest              b3048463dcef        2 weeks ago         175MB
alpine              latest              d6e46aa2470d        5 weeks ago         5.57MB

```

Let's look at little more details about the `fedora:latest` image above. This can be done using `docker inspect`.

```shell

# docker inspect fedora:latest
[
    {
        "Id": "sha256:b3048463dcefbe4920ef2ae1af43171c9695e2077f315b2bc12ed0f6f67c86c7",
        "RepoTags": [
            "fedora:latest"
        ],
        "RepoDigests": [
            "fedora@sha256:aa889c59fc048b597dcfab40898ee3fcaad9ed61caf12bcfef44493ee670e9df"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2020-11-12T00:25:31.334712859Z",
        "Container": "50cf73b69958473ab2f9a10d3249df073c99b7767ec7f1ff5ffd56da4f35397b",
        "ContainerConfig": {
            "Hostname": "50cf73b69958",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "DISTTAG=f33container",
                "FGC=f33",
                "FBR=f33"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"/bin/bash\"]"
            ],
            "Image": "sha256:3b1b0c55a47e10ea93d904fc20c39d253f9e1ad770922e8fb4af93dcec6691ce",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "maintainer": "Clement Verna <cverna@fedoraproject.org>"
            }
        },
        "DockerVersion": "19.03.12",
        "Author": "",
        "Config": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "DISTTAG=f33container",
                "FGC=f33",
                "FBR=f33"
            ],
            "Cmd": [
                "/bin/bash"
            ],
            "Image": "sha256:3b1b0c55a47e10ea93d904fc20c39d253f9e1ad770922e8fb4af93dcec6691ce",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "maintainer": "Clement Verna <cverna@fedoraproject.org>"
            }
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 174996402,
        "VirtualSize": 174996402,
        "GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/49fc1d5368593b27f5d3e38306c9e6ee1d76869a310879b406daaaadd04b4ef0/merged",
                "UpperDir": "/var/lib/docker/overlay2/49fc1d5368593b27f5d3e38306c9e6ee1d76869a310879b406daaaadd04b4ef0/diff",
                "WorkDir": "/var/lib/docker/overlay2/49fc1d5368593b27f5d3e38306c9e6ee1d76869a310879b406daaaadd04b4ef0/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:ed0c36ccfcbe08498869bb435711b2657b593806792e29582fa90f43d87b2dfb"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]

```

Let's Look at Some of the interesting details -

1. `Id` - Image ID - Soon we'll look at how this is created.
2. `RepoTags` and `RepoDigests` - Tag for the image and Digest for the Image manifest (we'll look at what this means).
3. `ContainerConfig` - The container that was used to create this image.
4. `Config` - The actual configuration of the image (Note: this looks similar to `ContainerConfig` above, please look at this [stackoverflow question](){:target="\_blank"} to understand the difference).
5. `GraphDriver` - Details of the Storage Driver used to use this Image with container. Please look at [Docker Storage Drivers documentation](){:target="\_blank"} for further details.
6. `RootFS` - The layers that make up root file system for the image.
7. `Metadata` - Any additional metadata for the image.

As one can observe above, there are quite a few `sha256` digests present, let's look at what each of those are generated.
