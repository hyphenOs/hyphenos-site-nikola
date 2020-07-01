.. title: Python Project Workflows - Part 2 (Pipenv)
.. slug: python-project-workflows-2
.. date: 2018-05-14 12:57:04 UTC+05:30
.. tags: Python
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: In the [first post](/python-dev-environment.html), we looked at what are typical issues in setting up Python project workflows and took an overview of the tools of the trade. In this post, we are going to be looking closely at `pipenv` a tool for managing Python project dependencies. In particular we are looking at how `pipenv` will help us solve the problems of reproducible builds and managing dev and production environments properly.

# Intended Audience

If you have not read [first part](/python-dev-environment.html) of this series, it might be a good idea to start there to understand the issues that we are trying to address here.

Some background working in Python would be certainly useful. This document tries to provide an overview of the role of `requirements.txt` and how it is used along with `pip` and `setuptools`, but is not a tutorial on `requirements.txt`.

# Introduction

As seen in Part 1 of the series of blog, `requirements.txt` file can be used to track external dependencies of our project. Often, one may not necessarily have a separate `requirements.txt` file, but the dependencies can be tracked directly in the `setup.py` if one is using `setuptools` for building, installing and publishing the project. The way this works is `setup` function in `setuptools` takes an argument called `install_requires` and typically this list is generated from the `requirements.txt` file above, with something very simple like `install_requires=open('requirements.txt', 'r').readlines()`. This will make sure whenever we are trying to `install` the project, these dependencies (and their dependencies) are installed as well.

One of the challenges with `requirements.txt`, when using [semantic versioning](https://semver.org/) (which is always recommended) is whether to specify exact versions of dependencies (using eg. say `requests==2.18.4`) or using the versions that are compatible with our Project. While it may be desirable to use latest compatible versions, it might result in different builds (where the build checksum does not match) at different times of the same project (even for the same commit) using different versions of dependencies and hence the builds are no longer deterministic or reproducible, so it probably is a good idea to use fixed versions of dependencies and whenever the dependencies are updated, they undergo a proper testing and then a newer version of project can use newer version of dependencies.

As we have seen `pip freeze` can be used to generate exact version of dependencies installed. However this is still not most ideal, the reason being `pip freeze` will collect all dependencies (which is good), but won't tell us which dependencies were installed to satisfy which dependencies or in other words, it doesn't show dependency graphs and how they are resolved. This means even 'dev' dependencies are collected by `pip freeze`, which is something we don't want.

We might require separate environment for development and separate environment for deployment. Some of the dependencies that were installed during development are often not required in a deployment environment (eg. tools like `pylint` that are used for code linting, or `mock`, `unittest` that are used for unit-testing etc.) Usually it's not a harm to install those in a deployment environment as well, but more software on the production machine means bigger attack surface, which usually is best avoided.

So how do we solve this issue? In comes `pipenv`. Next we'll look at some sample use of `pipenv` and how the problem described above can be addressed and some things to keep in mind.

# A Quick Overview of 'pipenv'

[`pipenv`](https://github.com/pypa/pipenv) is a recommended tool for Python packaging and it tries to bring the features available for packaging in things like `npm`, `cargo`, `bundler` etc. A project's dependencies are provided in a file called `Pipfile`. `pipenv` then along with `Pipfile` and companion `Pipfile.lock` provides the necessary tooling. A more detailed list of `pipenv` [features is available here](https://github.com/pypa/pipenv#-features). Also, with `pipenv`, there is no need to manage `virtualenv` for the project. It is managed automatically by the `pipenv`. It's highly recommended to read more about it's documentation above. Just like `pip`, it's possible to provide dependencies from a VCS or your local file-system. Also, it's quite easy to manage the development environment as most of the `pipenv` commands support an option called `--dev`, that is extremely useful. Next we'd take a look at a simple `pipenv` based workflow that I usually follow.

# A Simple workflow for 'pipenv'

This section explains how `pipenv` manages your environment, for a quick usable workflow go directly to [summary section below](#summary)

Usually, your project may already have a `requirements.txt`, so the first step in getting started with `pipenv` is generating a `Pipfile`. This can be done as follows

```bash
gabhijit@dev:~/foo/pipenv-foo$ pipenv install
Creating a virtualenv for this project…
Using /usr/bin/python (2.7.12) to create virtualenv…
⠋Already using interpreter /usr/bin/python
New python executable in /home/gabhijit/.local/share/virtualenvs/pipenv-foo-Y9xwmvqn/bin/python
Installing setuptools, pip, wheel...done.

Virtualenv location: /home/gabhijit/.local/share/virtualenvs/pipenv-foo-Y9xwmvqn
Creating a Pipfile for this project…
Pipfile.lock not found, creating…
Locking [dev-packages] dependencies…
Locking [packages] dependencies…
Updated Pipfile.lock (dfae9f)!
Installing dependencies from Pipfile.lock (dfae9f)…
  🐍   ▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉ 0/0 — 00:00:00
To activate this project's virtualenv, run the following:
 $ pipenv shell
gabhijit@dev:~/foo/pipenv-foo$

```
This created empty `Pipfile` and `Pipfile-lock`. It also created a virtual environment for us. Next is to add some dependencies to your project. Let's say we want to add `requests` as a dependency on our project. Also, while developing we are going to use `pylint` for code checking/verification. So we specify these dependencies as follows in the `Pipfile`.

```
gabhijit@dev:~/foo/pipenv-foo$ cat Pipfile
[[source]]
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"

[dev-packages]
pylint = "*"

[packages]
requests = "*"

[requires]
python_version = "2.7"
```
and then we simply say `pipenv update` and this updates the `Pipfile.lock` as follows (output truncated.)
```
gabhijit@dev:~/foo/pipenv-foo$ cat Pipfile.lock

{
    "_meta": {
        "hash": {
            "sha256": "a91b34418a47e89f73a94acd1be60e7a60c9dace59cd436ddbe25fb315cb3fb8"
        },
        "pipfile-spec": 6,
        "requires": {
            "python_version": "2.7"
        },
        "sources": [
            {
                "name": "pypi",
                "url": "https://pypi.org/simple",
                "verify_ssl": true
            }
        ]
    },
    "default": {
        "certifi": {
            "hashes": [
                "sha256:13e698f54293db9f89122b0581843a782ad0934a4fe0172d2a980ba77fc61bb7",
                "sha256:9fa520c1bacfb634fa7af20a76bcbd3d5fb390481724c597da32c719a7dca4b0"
            ],
            "version": "==2018.4.16"
        },
        "chardet": {
            "hashes": [
                "sha256:84ab92ed1c4d4f16916e05906b6b75a6c0fb5db821cc65e70cbd64a3e2a5eaae",
                "sha256:fc323ffcaeaed0e0a02bf4d117757b98aed530d9ed4531e3e15460124c106691"
            ],:
            "version": "==3.0.4"
        },
        "idna": {
            "hashes": [
                "sha256:2c6a5de3089009e3da7c5dde64a141dbc8551d5b7f6cf4ed7c2568d0cc520a8f",
                "sha256:8c7309c718f94b3a625cb648ace320157ad16ff131ae0af362c9f21b80ef6ec4"
            ],
            "version": "==2.6"
        },
        "requests": {
            "hashes": [
                "sha256:6a1b267aa90cac58ac3a765d067950e7dbbf75b1da07e895d1f594193a40a38b",
                "sha256:9c443e7324ba5b85070c4a818ade28bfabedf16ea10206da1132edaa6dda237e"
            ],
            "index": "pypi",
            "version": "==2.18.4"
        },
        "urllib3": {
            "hashes": [
                "sha256:06330f386d6e4b195fbfc736b297f58c5a892e4440e54d294d7004e3a9bbea1b",
                "sha256:cc44da8e1145637334317feebd728bd869a35285b93cbb4cca2577da7e62db4f"
            ],
            "version": "==1.22"
        }
    },
    "develop": {
        "astroid": {
            "hashes": [
                "sha256:35cfae47aac19c7b407b7095410e895e836f2285ccf1220336afba744cc4c5f2",
                "sha256:38186e481b65877fd8b1f9acc33e922109e983eb7b6e487bd4c71002134ad331"
            ],
            "version": "==1.6.3"
        },
        "backports.functools-lru-cache": {
            "hashes": [
                "sha256:9d98697f088eb1b0fa451391f91afb5e3ebde16bbdb272819fd091151fda4f1a",
                "sha256:f0b0e4eba956de51238e17573b7087e852dfe9854afd2e9c873f73fc0ca0a6dd"
            ],
            "markers": "python_version == '2.7'",
            "version": "==1.5"
        },
        ....
        ....
    },
}
```
It has three sections `_meta` that information is about the `virtualenv`, Python Version, Pypi source etc. Then there are sections `default`, this corresponds to Project's dependencies and then there is section `develop`, which corresponds to Project's dependencies in development environment. Separating Project's dependencies from the development dependencies is a really neat thing. Also, notice each dependency has got a list of hashes and the current version that is installed. We'll see how this can be later used for generating `requirements.txt` file. Note here, all the standard `pip` commands like `pip install`, `pip freeze` etc. work as it is in the `pipenv`. But they are generally not required to be run (during development at-least).

`pipenv` also supports a command called `pipenv graph` that shows how each dependencies are resolved (from both `develop` and `default`). For our particular project, this looks like -

```
gabhijit@dev:~/foo/pipenv-foo$ pipenv graph
pylint==1.8.4
  - astroid [required: >=1.6,<2.0, installed: 1.6.3]
    - backports.functools-lru-cache [required: Any, installed: 1.5]
    - enum34 [required: >=1.1.3, installed: 1.1.6]
    - lazy-object-proxy [required: Any, installed: 1.3.1]
    - singledispatch [required: Any, installed: 3.4.0.3]
      - six [required: Any, installed: 1.11.0]
    - six [required: Any, installed: 1.11.0]
    - wrapt [required: Any, installed: 1.10.11]
  - backports.functools-lru-cache [required: Any, installed: 1.5]
  - configparser [required: Any, installed: 3.5.0]
  - isort [required: >=4.2.5, installed: 4.3.4]
    - futures [required: Any, installed: 3.2.0]
    - futures [required: Any, installed: 3.2.0]
  - mccabe [required: Any, installed: 0.6.1]
  - singledispatch [required: Any, installed: 3.4.0.3]
    - six [required: Any, installed: 1.11.0]
  - six [required: Any, installed: 1.11.0]
requests==2.18.4
  - certifi [required: >=2017.4.17, installed: 2018.4.16]
  - chardet [required: >=3.0.2,<3.1.0, installed: 3.0.4]
  - idna [required: >=2.5,<2.7, installed: 2.6]
  - urllib3 [required: <1.23,>=1.21.1, installed: 1.22]
```

This manages dependencies well, but to be able to generate reproducible builds we need 'fixed' versions of packages to be installed (or known - how to install), or in other words, we do require `requirements.txt` file (and `dev-requirements.txt` file). `pipenv` makes generating these files extremely easy with `pipenv lock` command as shown below -

```
gabhijit@dev:~/foo/pipenv-foo$ pipenv lock -r
-i https://pypi.org/simple
certifi==2018.4.16
chardet==3.0.4
idna==2.6
requests==2.18.4
urllib3==1.22
  warnings.warn(warn_message, ResourceWarning)
gabhijit@dev:~/foo/pipenv-foo$ pipenv lock -r --dev
-i https://pypi.org/simple
astroid==1.6.3
backports.functools-lru-cache==1.5; python_version < '3.4'
configparser==3.5.0; python_version == '2.7'
enum34==1.1.6; python_version < '3.4'
futures==3.2.0
isort==4.3.4
lazy-object-proxy==1.3.1
mccabe==0.6.1
pylint==1.8.4
singledispatch==3.4.0.3; python_version < '3.4'
six==1.11.0
wrapt==1.10.11
```

Note above, `pipenv` also added the source from which these versions are to be donwloaded, which is kind of very useful (so during build some different source won't be accidentally used.)
You might see a warning as below, but it can be safely ignored -
```
/home/gabhijit/.local/lib/python2.7/site-packages/pipenv/utils.py:1288: ResourceWarning: Implicitly cleaning up <TemporaryDirectory '/tmp/pipenv-EMQAPo-requirements'>
```

So what we do now is - used the output generated above to populate `requirements.txt` and `dev-requirements.txt` respectively.

# <a name="summary"></a> Workflow Summary for Pipenv

All of this can be summarized as follows (assuming you are starting for a brand new-project) -

1. Start with `pipenv install`

2. Add necessary project requirements.

3. `pipenv update`

4. `pipenv lock -r > requirements.txt`

5. `pipenv lock -r --dev > dev- requirements.txt`

6. In your build system use `pip` to install dependencies like `pip install -r requirements.txt`. This would avoid installing dependencies from `dev-requirements.txt` during build. Since all the versions of packages are fixed, it will always generate reproducible builds.

7. Later, whenever the dependencies are to be updated (for security fixes say), go to step 3. and follow again.


# Few More Points

* VCS repositories - `pipenv` allows to use dependencies from the VCS directly (something that `pip` supports as well.) The exact syntax for this is as follows - In the `Pipfile` mention the dependency as follows
```
tickerplot = { git = "https://github.com/gabhijit/tickerplot.git", ref="v0.0.4", editable="True" }
```
Note: here it is required to give `editable=True` or else `pipenv lock -r` won't list the subsequent dependencies of the dependencies taken from the `git` repository above.

* Starting with existing `requirements.txt` file - If there's an already exisiting `requirements.txt` file, `Pipfile` will pick up the exact versions from the `requirements.txt` and you won't be able to `upgrade` to newer versions without fixing this first.

