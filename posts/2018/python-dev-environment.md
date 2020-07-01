.. title: Python Project Workflows - Part 1
.. slug: python-project-workflows-1
.. date: 2018-05-08 12:57:04 UTC+05:30
.. tags: Python
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: In a Python based development, tools like `pylint`, `pipenv` and `virtualenv` become important pieces of your development workflow. In the first part of the series of blog posts, we look at overview of the issues that become important as the scope of the project and size of the team working on project increases. In later posts in these series, we would try to address these issues one by one, we look at starting a project, identifying and resolving dependencies 'correctly', why reproducible builds matter, how to separate development and production environment and eventually how to containerize the application.

# Intended Audience

If you have done some Python programming, but unsure of what are the best practices to follow in a slightly bigger project, this post will offer you some choices (since best choices are often very subjective there is no one good practice.)

Some knowledge of tools in Python ecosystem like `pip`, `virtualenv`, `pylint` is useful. If you have never heard about them, the discussion below might still be useful, but is definitely not an introduction or a tutorial on the usage of those tools.

Also, working knowledge of VCS like `git` might be useful because the discussion below assumes that you are developing project in a VCS like `git`. You should be using `git` even if you do not want to follow any of the workflow discussed here.

If you have forked a project and are unsure of what some of those files in the projects like `requirements.txt`, `Pipfile`, `Pipfile.lock`, `setup.py` are, this should help you develop some understanding of that.

# Introduction
Whenever I have an idea to try out, my simplest workflow is either open a Python file in `vim` and then simply run `python file.py` or sometimes just fire a Python interactive shell and do stuff. This is #1 among the things that I love about Python as it allows you to quickly iterate over ideas. However, when the scope of things you are trying to do goes beyond a single Python file and what is provided by Python standard library, the above workflow can start becoming at the very least inadequate and even full of unpleasant surprises in some cases. Things become even more challenging when there are multiple developers working on the project. So usually, it's probably a good idea to give some thoughts to the development setup early enough, rather than 'overlay' some workflow later. Some of the specific issues that we would like to address include -

1. Avoiding run-time errors that are caused by undefined variable etc, this is especially required in an interpreted language like Python.

2. Manage dependencies project uses, so that multiple projects can co-exists on a development machine. Chances are you are collaborating on more than one project which have conflicting dependencies.

3. Produce identical builds across time and developer's platform.

4. Be able to separate a development environment from a production environment, there are certain packages needed only on development and/or build setup but are not required on production setup (eg. unit-testing).

All the issues discussed above have caused unpleasant surprises in a deployed application and it's quite possible to avoid those or at-least substantially decrease the occurrences of such issues.

Since there are a number of tools out there, which in some combination will prove quite useful, what we discuss below are how some of those tools can be used to address the challenge. First let's look at the problems in little more details

# Dependencies

Every project is likely to have dependencies on some libraries that are developed separately and maintained by someone else. In fact effectively tracking dependencies, can become quite a challenging task. Let's look at some of the challenges -

## Conflicting Dependencies

While Python's standard library is quite extensive, almost always we need a functionality that is better provided by some packages. A very good example is the [requests](http://docs.python-requests.org/en/master/) package, which is a great substitute for the Python standard library's `httplib` or `urllib2`. It's very likely that you are simultaneously working on two projects where one project works with a specific version of `requests` (say version-a) and another works with another version of `requests` (say version-b) and unfortunately they are incompatible. The two code bases are not related to each other, so likely you would want to use both the versions with the respective code base. Python solves this problem using a tool called `virtualenv`. What [`virtualenv`](https://virtualenv.pypa.io/en/stable/) essentially does is creates a self contained Python environment in a single directory and does tricks with Python's system path (`sys.path`) such that the Python interpreter finds packages and modules from inside this directory. Think of this as a Python equivalent of `chroot`, loosely speaking. In fact, if a developed Python application is going to be containerized, `virtualenv` will almost be a required one. So it's always a recommended practice to start a project with it's own `virtualenv`.

Key Takeaway : *Every project should have it's on `virtualenv`.*


## Transitive Dependencies

We have looked at how `virtualenv` could help us solve the problem of conflicting dependencies on a developer's machine. However, now when we look at an individual project, how do we install the dependencies. One good thing about `virtualenv` is, when you create a virtual environment, it installs `pip` Python's recommended package manager inside the virtual environment. So one should always use this `pip` for installing additional dependencies.

A small digression here before we look more at dependencies and transitive dependencies. Typically if you are using Linux platform for development, your distribution will also provide the distribution specific versions of Python packages (like `rpm` or `deb`) supported by the package manager of your distribution (like `yum` or `apt`). Whenever you are developing using Python you should *never* use these packages - often these packages are outdated, second their dependencies come as distribution specific packages like `rpm` or `deb` (and not as pip packages) and they are not so straight forward to use in a virtual environment.

One of the advantages of using `pip` is, if the package you are using has dependencies itself, they are also installed recursively till all dependencies are resolved.

Also, one of the recommended practice is to use your distributions packages for `pip` and `python` and then use that `pip` to install your project's dependencies.

Key Takeaway : *Always use `pip` to install dependencies in a virtual environment created by `virtualenv` for the project*

## Deterministic Builds (or Build Reproducibility)

Once we start with a virtual environment and use `pip` to install packages, often we have a pretty good starting point for the Project's development. It may not though be enough or always optimal. For example a question one might want to ask is - should the virtual environment itself be maintained inside `git`? It's not a very bad idea, but probably not a recommended one. A natural question then is how can a team collaborate effectively? One of the ways to solve that problem is by maintaining a `requirements.txt` file, that lists down your dependencies and their respective versions and instead maintain that file in a `git` repository. `pip` allows installing packages listed in a file using a command like `pip -r requirements.txt`. Pip also allows a command called `pip freeze` that looks at currently installed packages in a `virtualenv` and generates a list of packages with their installed version. (Note: strictly speaking pip can also look at packages installed in the standard system path on the machine, but as discussed already, we will typically use `pip` that `virtualenv` installs, we are only looking at dependencies installed in a `virtualenv`.) So something like `pip freeze > requirements.txt` would help you generate the `requirements.txt` and then this file can be tracked in `git`. Someone cloning (or forking) the repository can simply do a `pip -r requirements.txt` after cloning the repository (and creating the `virtualenv`.) and would have identical versions of packages installed (well almost - we'd look at a subtle issue and how to fix that later.)

Key Takeaway : *Use a `requirements.txt` file to track your dependencies and generate it using `pip freeze`.*

# Separate Environments

What we have described so far should be 'good enough' when starting a project. However when the scope of the project starts improving, unit tests are added, coding guidelines are to be enforced, there might be more needed to be done than what we have discussed so far. Let's look at some of the challenges. What `pip freeze` does is it lists down all the packages (with their installed versions) that are installed by pip. But let's consider this - You want to run some unit tests while building a project and you are using tools like `nose` to do so and have installed it using `pip`, `pip freeze` will catch that for you as well. In a development environment you want to run certain sanity checks etc and are using tools like `pylint` (see below for more about `pylint`), but may be you don't want those in a production environment. So you want to kind of keep the installed dependencies in a development environment different from those in production environment, tools like `pipenv` help yo fix that problem. In the next article of this series, we are going to take a closer look at `pipenv`.

# Code Quality

Often as the size of a team working on a project grows, it is often not sufficient to simply document 'recommended' practices and conventions, there needs to be a way to enforce some. (for instance you might want your code to strictly adhere to `pep8` and code not conforming to `pep8` is not admissible). Python also being a weakly typed and interpreted languages, a number of errors show up at the run-time, so it's a better practice to actually use some code linting tools that will analyze your code (often without running it) and highlight potential errors that can be easily fixed during the development itself. In a subsequent post we will take a more detailed look at `pylint` and how it can be integrated into the development workflow to ensure certain code quality.

# Summary

In this part, we discussed typical challenges in developing a Python project from an ecosystem perspective and provided an overview of some tools that can help address. In summary, following just the three simple practices should start as a good starting point

1. Every Project should have it's own `virtualenv`.

2. Always use `pip` to install dependencies in a virtual environment created by `virtualenv`.

3. Use `requirements.txt` file to track your dependencies and generate it using `pip freeze`.

In remaining parts we would look at how to use `pipenv` to setup separate Development and Production environments, how to use `pylint` and integrate it as a `git pre-commit hook` to enforce certain coding standards and automatically check for errors in Python code without waiting for them to show up at run-time and how a Python project can be containerized.

