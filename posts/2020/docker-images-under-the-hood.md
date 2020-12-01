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


# Docker Registry API and Docker Images

Before we start making sense of so many `sha256` digests above, it's a good idea to pause a bit and look at some of the registry  APIs and leverage that to understand how the images can be downloaded. To better understand this, I looked at running `skopeo inspect` for the `fedora:docker` image as above. [`skopeo`](){:target="\_blank"} is a tool that can be used to experiment with Docker as well as OCI images. To actually understand some of the `sha256` hashes, we look at the debug output of following command.

```shell
$ skopeo inspect docker://docker.io/fedora:latest --debug
DEBU[0000] Loading registries configuration "/etc/containers/registries.conf"
DEBU[0000] Trying to access "docker.io/library/fedora:latest"
DEBU[0000] Credentials not found
DEBU[0000] Using registries.d directory /etc/containers/registries.d for sigstore configuration
DEBU[0000]  No signature storage configuration found for docker.io/library/fedora:latest
DEBU[0000] Looking for TLS certificates and private keys in /etc/docker/certs.d/docker.io
DEBU[0000] GET https://registry-1.docker.io/v2/
DEBU[0000] Ping https://registry-1.docker.io/v2/ status 401
DEBU[0000] GET https://auth.docker.io/token?scope=repository%3Alibrary%2Ffedora%3Apull&service=registry.docker.io
DEBU[0001] GET https://registry-1.docker.io/v2/library/fedora/manifests/latest
DEBU[0002] Content-Type from manifest GET is "application/vnd.docker.distribution.manifest.list.v2+json"
DEBU[0002] GET https://registry-1.docker.io/v2/library/fedora/manifests/sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80
DEBU[0003] Content-Type from manifest GET is "application/vnd.docker.distribution.manifest.v2+json"
DEBU[0003] Downloading /v2/library/fedora/blobs/sha256:b3048463dcefbe4920ef2ae1af43171c9695e2077f315b2bc12ed0f6f67c86c7
DEBU[0003] GET https://registry-1.docker.io/v2/library/fedora/blobs/sha256:b3048463dcefbe4920ef2ae1af43171c9695e2077f315b2bc12ed0f6f67c86c7
DEBU[0009] Credentials not found
DEBU[0009] Using registries.d directory /etc/containers/registries.d for sigstore configuration
DEBU[0009]  No signature storage configuration found for docker.io/library/fedora:latest
DEBU[0009] Looking for TLS certificates and private keys in /etc/docker/certs.d/docker.io
DEBU[0009] GET https://registry-1.docker.io/v2/
DEBU[0015] Ping https://registry-1.docker.io/v2/ status 401
DEBU[0015] GET https://auth.docker.io/token?scope=repository%3Alibrary%2Ffedora%3Apull&service=registry.docker.io
DEBU[0021] GET https://registry-1.docker.io/v2/library/fedora/tags/list
{
    "Name": "docker.io/library/fedora",
    "Digest": "sha256:aa889c59fc048b597dcfab40898ee3fcaad9ed61caf12bcfef44493ee670e9df",
    "RepoTags": [
        "20",
        "21",
        "22",
        "23",
        "24",
        "25",
        "26-modular",
        "26",
        "27",
        "28",
        "29",
        "30",
        "31",
        "32",
        "33",
        "34",
        "branched",
        "heisenbug",
        "latest",
        "modular",
        "rawhide"
    ],
    "Created": "2020-11-12T00:25:31.334712859Z",
    "DockerVersion": "19.03.12",
    "Labels": {
        "maintainer": "Clement Verna \u003ccverna@fedoraproject.org\u003e"
    },
    "Architecture": "amd64",
    "Os": "linux",
    "Layers": [
        "sha256:ae7b613df528a37664448affa6e52ff405701cda015a2a67301423bc20226b61"
    ],
    "Env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "DISTTAG=f33container",
        "FGC=f33",
        "FBR=f33"
    ]
}

```

We can now try to leverage the steps above to figure out what is happening here -

There are following API Calls made

1. For Getting auth token for the repository `library/fedora` for `image pull` scope. `
```
https://auth.docker.io/token?scope=repository%3Alibrary%2Ffedora%3Apull&service=registry.docker.io
```

2. Later A call is made to obtain a list of all manifests.

```
https://registry-1.docker.io/v2/library/fedora/manifests/latest
```

3. Then a call is made to obtain the details of the manifest -

```
https://registry-1.docker.io/v2/library/fedora/manifests/sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80
```

4. Finally a call is made to obtain the details of the image -
```
https://registry-1.docker.io/v2/library/fedora/blobs/sha256:b3048463dcefbe4920ef2ae1af43171c9695e2077f315b2bc12ed0f6f67c86c7
```

5. In the end another API call is made to obtain a list of all tags
```
https://registry-1.docker.io/v2/library/fedora/tags/list
```

Note: There are quite a few differences between the `skopeo inspect` and `docker inspect` output viz. -

1. `docker inspect` works with locally downloaded docker images
2. `skopeo inspect` does not require the image is to be downloaded.
3. The outputs of the two commands are quite different with some common parts.

What we'll do next is run these `HTTP GET`s ourselves to make sense of what exactly is happening. This should help us understand some of the `sha256`s as well.

1. Let's first obtain the Token for downloading all the data -
```
$ curl 'https://auth.docker.io/token?scope=repository%3Alibrary%2Ffedora%3Apull&service=registry.docker.io' -vvv | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 18.232.227.119...
* Connected to auth.docker.io (18.232.227.119) port 443 (#0)
* found 138 certificates in /etc/ssl/certs/ca-certificates.crt
* found 557 certificates in /etc/ssl/certs
* ALPN, offering http/1.1
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* SSL connection using TLS1.2 / ECDHE_RSA_AES_128_GCM_SHA256
*		 server certificate verification OK
*		 server certificate status verification SKIPPED
*		 common name: *.docker.io (matched)
*		 server certificate expiration date OK
*		 server certificate activation date OK
*		 certificate public key: RSA
*		 certificate version: #3
*		 subject: CN=*.docker.io
*		 start date: Sat, 23 May 2020 00:00:00 GMT
*		 expire date: Wed, 23 Jun 2021 12:00:00 GMT
*		 issuer: C=US,O=Amazon,OU=Server CA 1B,CN=Amazon
*		 compression: NULL
* ALPN, server did not agree to a protocol
> GET /token?scope=repository%3Alibrary%2Ffedora%3Apull&service=registry.docker.io HTTP/1.1
> Host: auth.docker.io
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Date: Tue, 01 Dec 2020 07:59:00 GMT
< Transfer-Encoding: chunked
< Strict-Transport-Security: max-age=31536000
<
{ [4364 bytes data]
100  4351    0  4351    0     0   3134      0 --:--:--  0:00:01 --:--:--  3136
* Connection #0 to host auth.docker.io left intact
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvZmVkb3JhIiwiYWN0aW9ucyI6WyJwdWxsIl0sInBhcmFtZXRlcnMiOnsicHVsbF9saW1pdCI6IjEwMCIsInB1bGxfbGltaXRfaW50ZXJ2YWwiOiIyMTYwMCJ9fV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTYwNjgwOTg0MCwiaWF0IjoxNjA2ODA5NTQwLCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6ImNYVWxlTXlqWjFjSmRKMFREb202IiwibmJmIjoxNjA2ODA5MjQwLCJzdWIiOiIifQ.fCghcdjMGKxNZct1IiOqW5s5yAnDcUkD-4Z2zK0M8ioi8GGMT85yesLx4DnCqe7tqH96STp3MSMd-_t1aIjEAQggQfbkWiPaCib1ybQFY6RhQ6YGVGa0uhFullHPSOsa2MEyqtOSMfMDH_rJSMn7XwduvQv6hmQspqFzELCUn0dRwxdmzelVy_3KQd6O4TZGsvTROpCfe8dqfL8xRYd0UqlPWk6bHKIcUSNFm3t3KjI2R841NH9PsX0sW8LiOL5zlSC_9s12kAgbRHxh32uDc4r_aT-YN1B8fIcwK7ZHjr6XQdddDU-n9-GnQiG3Y4WtG9CqvRuSXNDaMO89bR55VQ",
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvZmVkb3JhIiwiYWN0aW9ucyI6WyJwdWxsIl0sInBhcmFtZXRlcnMiOnsicHVsbF9saW1pdCI6IjEwMCIsInB1bGxfbGltaXRfaW50ZXJ2YWwiOiIyMTYwMCJ9fV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTYwNjgwOTg0MCwiaWF0IjoxNjA2ODA5NTQwLCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6ImNYVWxlTXlqWjFjSmRKMFREb202IiwibmJmIjoxNjA2ODA5MjQwLCJzdWIiOiIifQ.fCghcdjMGKxNZct1IiOqW5s5yAnDcUkD-4Z2zK0M8ioi8GGMT85yesLx4DnCqe7tqH96STp3MSMd-_t1aIjEAQggQfbkWiPaCib1ybQFY6RhQ6YGVGa0uhFullHPSOsa2MEyqtOSMfMDH_rJSMn7XwduvQv6hmQspqFzELCUn0dRwxdmzelVy_3KQd6O4TZGsvTROpCfe8dqfL8xRYd0UqlPWk6bHKIcUSNFm3t3KjI2R841NH9PsX0sW8LiOL5zlSC_9s12kAgbRHxh32uDc4r_aT-YN1B8fIcwK7ZHjr6XQdddDU-n9-GnQiG3Y4WtG9CqvRuSXNDaMO89bR55VQ",
  "expires_in": 300,
  "issued_at": "2020-12-01T07:59:00.946986182Z"
}
```

2. Use the token obtained above to download the manifest -
```
$ curl -H 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvZmVkb3JhIiwiYWN0aW9ucyI6WyJwdWxsIl0sInBhcmFtZXRlcnMiOnsicHVsbF9saW1pdCI6IjEwMCIsInB1bGxfbGltaXRfaW50ZXJ2YWwiOiIyMTYwMCJ9fV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTYwNjgwOTg0MCwiaWF0IjoxNjA2ODA5NTQwLCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6ImNYVWxlTXlqWjFjSmRKMFREb202IiwibmJmIjoxNjA2ODA5MjQwLCJzdWIiOiIifQ.fCghcdjMGKxNZct1IiOqW5s5yAnDcUkD-4Z2zK0M8ioi8GGMT85yesLx4DnCqe7tqH96STp3MSMd-_t1aIjEAQggQfbkWiPaCib1ybQFY6RhQ6YGVGa0uhFullHPSOsa2MEyqtOSMfMDH_rJSMn7XwduvQv6hmQspqFzELCUn0dRwxdmzelVy_3KQd6O4TZGsvTROpCfe8dqfL8xRYd0UqlPWk6bHKIcUSNFm3t3KjI2R841NH9PsX0sW8LiOL5zlSC_9s12kAgbRHxh32uDc4r_aT-YN1B8fIcwK7ZHjr6XQdddDU-n9-GnQiG3Y4WtG9CqvRuSXNDaMO89bR55VQ' https://registry-1.docker.io/v2/library/fedora/manifests/latest  -H 'Accept: application/vnd.docker.distribution.manifest.list.v2+json' -vvv | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:04 --:--:--     0*   Trying 52.4.20.24...
  0     0    0     0    0     0      0      0 --:--:--  0:00:05 --:--:--     0* Connected to registry-1.docker.io (52.4.20.24) port 443 (#0)
  0     0    0     0    0     0      0      0 --:--:--  0:00:05 --:--:--     0* found 138 certificates in /etc/ssl/certs/ca-certificates.crt
* found 557 certificates in /etc/ssl/certs
* ALPN, offering http/1.1
* SSL connection using TLS1.2 / ECDHE_RSA_AES_128_GCM_SHA256
*		 server certificate verification OK
*		 server certificate status verification SKIPPED
*		 common name: *.docker.io (matched)
*		 server certificate expiration date OK
*		 server certificate activation date OK
*		 certificate public key: RSA
*		 certificate version: #3
*		 subject: CN=*.docker.io
*		 start date: Sat, 23 May 2020 00:00:00 GMT
*		 expire date: Wed, 23 Jun 2021 12:00:00 GMT
*		 issuer: C=US,O=Amazon,OU=Server CA 1B,CN=Amazon
*		 compression: NULL
* ALPN, server did not agree to a protocol
  0     0    0     0    0     0      0      0 --:--:--  0:00:06 --:--:--     0> GET /v2/library/fedora/manifests/latest HTTP/1.1
> Host: registry-1.docker.io
> User-Agent: curl/7.47.0
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvZmVkb3JhIiwiYWN0aW9ucyI6WyJwdWxsIl0sInBhcmFtZXRlcnMiOnsicHVsbF9saW1pdCI6IjEwMCIsInB1bGxfbGltaXRfaW50ZXJ2YWwiOiIyMTYwMCJ9fV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTYwNjgwOTg0MCwiaWF0IjoxNjA2ODA5NTQwLCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6ImNYVWxlTXlqWjFjSmRKMFREb202IiwibmJmIjoxNjA2ODA5MjQwLCJzdWIiOiIifQ.fCghcdjMGKxNZct1IiOqW5s5yAnDcUkD-4Z2zK0M8ioi8GGMT85yesLx4DnCqe7tqH96STp3MSMd-_t1aIjEAQggQfbkWiPaCib1ybQFY6RhQ6YGVGa0uhFullHPSOsa2MEyqtOSMfMDH_rJSMn7XwduvQv6hmQspqFzELCUn0dRwxdmzelVy_3KQd6O4TZGsvTROpCfe8dqfL8xRYd0UqlPWk6bHKIcUSNFm3t3KjI2R841NH9PsX0sW8LiOL5zlSC_9s12kAgbRHxh32uDc4r_aT-YN1B8fIcwK7ZHjr6XQdddDU-n9-GnQiG3Y4WtG9CqvRuSXNDaMO89bR55VQ
> Accept: application/vnd.docker.distribution.manifest.list.v2+json
>
< HTTP/1.1 200 OK
< Content-Length: 1201
< Content-Type: application/vnd.docker.distribution.manifest.list.v2+json
< Docker-Content-Digest: sha256:aa889c59fc048b597dcfab40898ee3fcaad9ed61caf12bcfef44493ee670e9df
< Docker-Distribution-Api-Version: registry/2.0
< Etag: "sha256:aa889c59fc048b597dcfab40898ee3fcaad9ed61caf12bcfef44493ee670e9df"
< Date: Tue, 01 Dec 2020 08:03:30 GMT
< Strict-Transport-Security: max-age=31536000
< RateLimit-Limit: 100;w=21600
< RateLimit-Remaining: 98;w=21600
<
{ [1201 bytes data]
100  1201  100  1201    0     0    175      0  0:00:06  0:00:06 --:--:--   360
* Connection #0 to host registry-1.docker.io left intact
{
  "manifests": [
    {
      "digest": "sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      },
      "size": 529
    },
    {
      "digest": "sha256:12cea180e80e7f5b8847a82d35b4bd6a8089703f4c17d662512e49884146fa45",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "platform": {
        "architecture": "arm",
        "os": "linux",
        "variant": "v7"
      },
      "size": 529
    },
    {
      "digest": "sha256:6678ea27e7c7e3fce4adbc08d84cfef3d04211ee8e14e128acf6571b331a068a",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "platform": {
        "architecture": "arm64",
        "os": "linux",
        "variant": "v8"
      },
      "size": 529
    },
    {
      "digest": "sha256:2237bb2b8b11cb0858e9c3c791ce1cf45035122ad7f065fb2d723ca2ad0a822d",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      },
      "size": 529
    },
    {
      "digest": "sha256:153fa742cea71512a14224856538bcba53bb201f630304fd82abd78806fd1cf1",
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "platform": {
        "architecture": "s390x",
        "os": "linux"
      },
      "size": 529
    }
  ],
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
  "schemaVersion": 2
}
```

3. Look at the first entry above - This is the manifest that corresponds to `X64` and `linux`. Let's download that manifest and look at it's sha256 sum

```
$ curl -H 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvZmVkb3JhIiwiYWN0aW9ucyI6WyJwdWxsIl0sInBhcmFtZXRlcnMiOnsicHVsbF9saW1pdCI6IjEwMCIsInB1bGxfbGltaXRfaW50ZXJ2YWwiOiIyMTYwMCJ9fV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTYwNjgxMDMxNCwiaWF0IjoxNjA2ODEwMDE0LCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6IklDek9LWGYzbEduUlJKbDFjOGlZIiwibmJmIjoxNjA2ODA5NzE0LCJzdWIiOiIifQ.bzDj-7jp47cbKU36TxAHGeNgWdCF6AdPJR3hq9I4pFc4sYdwWO_75C9blL3Jj32qYGg-Uq1Vu6w6Hv8V7XbYqNq31Nv0BFjNai3A4ri96pI4Tj26h0JB2PdiBZFKvQURAL-qCTIzgia_Ie5MIYn25lS_4UzfQDQ7ik_a51IBDxkEm1nR5Qz0iDYZSqe_OIX59MXdsI11hAivyxlURL7LU1-8nxaEWWtjpvQViOsbz4OrrHKNAnHVdgnX7v7_ywHPRRWanG9BRmorPfgHskOqteGiudur5t-6gntfZswUZ9u3RPWvE8ypGe9DCdyZ_L70scI2exzrTB_XNdTDEwOAyQ' https://registry-1.docker.io/v2/library/fedora/manifests/sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80  -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' -vvv > manifest
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 54.85.56.253...
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to registry-1.docker.io (54.85.56.253) port 443 (#0)
* found 138 certificates in /etc/ssl/certs/ca-certificates.crt
* found 557 certificates in /etc/ssl/certs
* ALPN, offering http/1.1
* SSL connection using TLS1.2 / ECDHE_RSA_AES_128_GCM_SHA256
*		 server certificate verification OK
*		 server certificate status verification SKIPPED
*		 common name: *.docker.io (matched)
*		 server certificate expiration date OK
*		 server certificate activation date OK
*		 certificate public key: RSA
*		 certificate version: #3
*		 subject: CN=*.docker.io
*		 start date: Sat, 23 May 2020 00:00:00 GMT
*		 expire date: Wed, 23 Jun 2021 12:00:00 GMT
*		 issuer: C=US,O=Amazon,OU=Server CA 1B,CN=Amazon
*		 compression: NULL
* ALPN, server did not agree to a protocol
> GET /v2/library/fedora/manifests/sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80 HTTP/1.1
> Host: registry-1.docker.io
> User-Agent: curl/7.47.0
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDK1RDQ0FwK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakJHTVVRd1FnWURWUVFERXpzeVYwNVpPbFZMUzFJNlJFMUVVanBTU1U5Rk9reEhOa0U2UTFWWVZEcE5SbFZNT2tZelNFVTZOVkF5VlRwTFNqTkdPa05CTmxrNlNrbEVVVEFlRncweU1EQXhNRFl5TURVeE1UUmFGdzB5TVRBeE1qVXlNRFV4TVRSYU1FWXhSREJDQmdOVkJBTVRPMVZCVVRjNldGTk9VenBVUjFRek9rRTBXbFU2U1RWSFN6cFNOalJaT2xkRFNFTTZWMVpTU3pwTlNUTlNPa3RZVFRjNlNGZFRNenBDVmxwYU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcnh5Mm9uSDBTWHh4a1JCZG9wWDFWc1VuQVovOUpZR3JNSXlrelJuMTRsd1A1SkVmK1hNNUFORW1NLzBYOFJyNUlIN2VTWFV6K1lWaFVucVNKc3lPUi9xd3BTeFdLWUxxVnB1blFOWThIazdmRnlvN0l0bXgxajZ1dnRtVmFibFFPTEZJMUJNVnY2Y3IxVjV3RlZRYWc3SnhkRUFSZWtaR1M5eDlIcnM1NVdxb0lSK29GRGwxVVRjNlBFSjZVWGdwYmhXWHZoU0RPaXBPcUlYdHZkdHJoWFFpd204Y3EyczF0TEQzZzg2WmRYVFg3UDFFZkxFOG1jMEh4anBGNkdiNWxHZFhjdjU5cC9SMzEva0xlL09wRHNnVWJxMEFvd3Bsc1lLb0dlSmdVNDJaZG45SFZGUVFRcEtGTFBNK1pQN0R2ZmVGMWNIWFhGblI1TkpFU1Z1bFRRSURBUUFCbzRHeU1JR3ZNQTRHQTFVZER3RUIvd1FFQXdJSGdEQVBCZ05WSFNVRUNEQUdCZ1JWSFNVQU1FUUdBMVVkRGdROUJEdFZRVkUzT2xoVFRsTTZWRWRVTXpwQk5GcFZPa2sxUjBzNlVqWTBXVHBYUTBoRE9sZFdVa3M2VFVrelVqcExXRTAzT2toWFV6TTZRbFphV2pCR0JnTlZIU01FUHpBOWdEc3lWMDVaT2xWTFMxSTZSRTFFVWpwU1NVOUZPa3hITmtFNlExVllWRHBOUmxWTU9rWXpTRVU2TlZBeVZUcExTak5HT2tOQk5sazZTa2xFVVRBS0JnZ3Foa2pPUFFRREFnTklBREJGQWlFQXl5SEpJU1RZc1p2ZVZyNWE1YzZ4MjhrQ2U5M2w1QndQVGRUTk9SaFB0c0VDSURMR3pYdUxuekRqTCtzcWRkOU5FbkRuMnZ2UFBWVk1NLzhDQW1EaTVudnMiXX0.eyJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6ImxpYnJhcnkvZmVkb3JhIiwiYWN0aW9ucyI6WyJwdWxsIl0sInBhcmFtZXRlcnMiOnsicHVsbF9saW1pdCI6IjEwMCIsInB1bGxfbGltaXRfaW50ZXJ2YWwiOiIyMTYwMCJ9fV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTYwNjgxMDMxNCwiaWF0IjoxNjA2ODEwMDE0LCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6IklDek9LWGYzbEduUlJKbDFjOGlZIiwibmJmIjoxNjA2ODA5NzE0LCJzdWIiOiIifQ.bzDj-7jp47cbKU36TxAHGeNgWdCF6AdPJR3hq9I4pFc4sYdwWO_75C9blL3Jj32qYGg-Uq1Vu6w6Hv8V7XbYqNq31Nv0BFjNai3A4ri96pI4Tj26h0JB2PdiBZFKvQURAL-qCTIzgia_Ie5MIYn25lS_4UzfQDQ7ik_a51IBDxkEm1nR5Qz0iDYZSqe_OIX59MXdsI11hAivyxlURL7LU1-8nxaEWWtjpvQViOsbz4OrrHKNAnHVdgnX7v7_ywHPRRWanG9BRmorPfgHskOqteGiudur5t-6gntfZswUZ9u3RPWvE8ypGe9DCdyZ_L70scI2exzrTB_XNdTDEwOAyQ
> Accept: application/vnd.docker.distribution.manifest.v2+json
>
< HTTP/1.1 200 OK
< Content-Length: 529
< Content-Type: application/vnd.docker.distribution.manifest.v2+json
< Docker-Content-Digest: sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80
< Docker-Distribution-Api-Version: registry/2.0
< Etag: "sha256:fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80"
< Date: Tue, 01 Dec 2020 08:07:36 GMT
< Strict-Transport-Security: max-age=31536000
< RateLimit-Limit: 100;w=21600
< RateLimit-Remaining: 97;w=21600
<
{ [529 bytes data]
100   529  100   529    0     0    405      0  0:00:01  0:00:01 --:--:--   405
* Connection #0 to host registry-1.docker.io left intact

$ sha256sum manifest
fdf235fa167d2aa5d820fba274ec1d2edeb0534bd32d28d602a19b31bad79b80  manifest

$ cat manifest
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1997,
      "digest": "sha256:b3048463dcefbe4920ef2ae1af43171c9695e2077f315b2bc12ed0f6f67c86c7"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 63374419,
         "digest": "sha256:ae7b613df528a37664448affa6e52ff405701cda015a2a67301423bc20226b61"
      }
   ]
}

```
Observe that the `sha256sum` of the content downloaded (and saved as a file `manifest`) is same as the URL used to access the manifest. Note: here that we are adding a specific type of `Accept: application/vnd.docker.distribution.manifest.v2+json` header, the purpose of this header is to tell the API that we are interested in the manifest. A complete list of media types for different manifests is provided [here](){:target="_blank}

Also, the manifest contains references to two 'blobs' whose digest and size is available above -

1. `config` object is a blob, which we'll soon look at by downloading the config object
2. There is only one 'layer' with a `sha256` sum as above.

Next we are going to download the above two 'blobs' and inspect them in more details.

```
# image-config downloaded using the Registry API
$ cat image-config
{"architecture":"amd64","config":{"Hostname":"","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","DISTTAG=f33container","FGC=f33","FBR=f33"],"Cmd":["/bin/bash"],"Image":"sha256:3b1b0c55a47e10ea93d904fc20c39d253f9e1ad770922e8fb4af93dcec6691ce","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":{"maintainer":"Clement Verna \u003ccverna@fedoraproject.org\u003e"}},"container":"50cf73b69958473ab2f9a10d3249df073c99b7767ec7f1ff5ffd56da4f35397b","container_config":{"Hostname":"50cf73b69958","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","DISTTAG=f33container","FGC=f33","FBR=f33"],"Cmd":["/bin/sh","-c","#(nop) ","CMD [\"/bin/bash\"]"],"Image":"sha256:3b1b0c55a47e10ea93d904fc20c39d253f9e1ad770922e8fb4af93dcec6691ce","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":null,"Labels":{"maintainer":"Clement Verna \u003ccverna@fedoraproject.org\u003e"}},"created":"2020-11-12T00:25:31.334712859Z","docker_version":"19.03.12","history":[{"created":"2019-01-16T21:21:55.569693599Z","created_by":"/bin/sh -c #(nop)  LABEL maintainer=Clement Verna \u003ccverna@fedoraproject.org\u003e","empty_layer":true},{"created":"2020-04-30T23:21:44.324893962Z","created_by":"/bin/sh -c #(nop)  ENV DISTTAG=f33container FGC=f33 FBR=f33","empty_layer":true},{"created":"2020-11-12T00:25:30.976066436Z","created_by":"/bin/sh -c #(nop) ADD file:240dde03c4d9f0ad759f8d1291fb45ab2745b6a108c6164d746766239d3420ab in / "},{"created":"2020-11-12T00:25:31.334712859Z","created_by":"/bin/sh -c #(nop)  CMD [\"/bin/bash\"]","empty_layer":true}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:ed0c36ccfcbe08498869bb435711b2657b593806792e29582fa90f43d87b2dfb"]}}gabhijit@gabhijit-GL553VD:~/backup/personal-code/hyphenos-site-nikola$ sha256sum image-config
b3048463dcefbe4920ef2ae1af43171c9695e2077f315b2bc12ed0f6f67c86c7  image-config

# Similarly we download the layer.tar.gz and it's sha256sum is computed

$ sha256sum layer.tar.gz
ae7b613df528a37664448affa6e52ff405701cda015a2a67301423bc20226b61  layer.tar.gz

```

As can be seen above, the digests referred in the manifest can be downloaded as 'blobs' referenced by their `sha256sum`. This is called content addressable storage being used by Docker. We'll look at the downloaded `image-config` and we can see that this is identical to the one reported by `docker inspect` at the beginning of this post.

```shell
$ cat image-config | jq
{
  "architecture": "amd64",
  "config": {
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
  "container": "50cf73b69958473ab2f9a10d3249df073c99b7767ec7f1ff5ffd56da4f35397b",
  "container_config": {
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
  "created": "2020-11-12T00:25:31.334712859Z",
  "docker_version": "19.03.12",
  "history": [
    {
      "created": "2019-01-16T21:21:55.569693599Z",
      "created_by": "/bin/sh -c #(nop)  LABEL maintainer=Clement Verna <cverna@fedoraproject.org>",
      "empty_layer": true
    },
    {
      "created": "2020-04-30T23:21:44.324893962Z",
      "created_by": "/bin/sh -c #(nop)  ENV DISTTAG=f33container FGC=f33 FBR=f33",
      "empty_layer": true
    },
    {
      "created": "2020-11-12T00:25:30.976066436Z",
      "created_by": "/bin/sh -c #(nop) ADD file:240dde03c4d9f0ad759f8d1291fb45ab2745b6a108c6164d746766239d3420ab in / "
    },
    {
      "created": "2020-11-12T00:25:31.334712859Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/bash\"]",
      "empty_layer": true
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:ed0c36ccfcbe08498869bb435711b2657b593806792e29582fa90f43d87b2dfb"
    ]
  }
}
```

Thus we have managed to identify almost all the `sha256` digests that we saw initially in `docker inspect`. One thing that remains is the last `rootfs.diff_ids` part. This `sha256sum` is the checksum of the layer that is unzipped. We can verify that as follows -

```
$ gunzip layer.tar.gz
$ sha256sum layer.tar
ed0c36ccfcbe08498869bb435711b2657b593806792e29582fa90f43d87b2dfb  layer.tar
```

The main idea here is as follows - When a docker image is created, the configuration used for creating that image takes into account the actual 'contents' (`layer.tar` without the gzipped part) and the content hash is included in the image config and is thus available. Using this it is indeed possible to verify that the contents of the images. For example, after downloading the image configuration, one can verify the `sha256sum` indeed matches the object URL from where it was downloaded. Once that is verified, individual layers can also be verified as above, further those layers can be unzipped and their``sha256sum` can also be verified as matching in the `rootfs.diff_ids` in the image configuration. If all the digests (checksums) match, we can indeed verify that the correct image is downloaded.

This completes our basic discussion of the Docker Images. What we have looked at - how the images can be obtained by the docker engine from the docker repository. Next we will look at how those images are stored on the disk and how a container is created from those images.
