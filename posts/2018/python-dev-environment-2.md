.. title: Python Project Workflows - Part 3 (pylint)
.. slug: python-project-workflows-3
.. date: 2018-05-16 12:57:04 UTC+05:30
.. tags: Python
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: In the [first part](/python-dev-environment.html) we looked at a few challenges involved when developing a Python project in a collaborative environment. In the [second part](/python-dev-environment-2.html) we looked at how `Pipenv` addresses some of those issues. In this part of the series we are going to take a closer look at how one can use code linting tools. Specifically we are going to be looking in details at using `pylint`.

# Introduction

Python being a dynamically typed and interpreted language, there is no such thing as 'compile time' and most of the errors (even related to syntax) show up only at the run-time and that's kind of not good,  which can and should be easily avoided. So it's very important to have your code checked by some linting tools before it goes into the repository. Further, often, it's a good practice that certain coding guidelines are followed by the development team, in fact it's even better if they can be enforced using tools, so there is some level of consistency in the way a project is developed. Early is better than late. [`pylint`](https://www.pylint.org/) is an excellent tool to achieve both of these objectives (and more). In the discussion that follows, we are going to take a closer look at how `pylint` can be integrated into your Python development workflow.

# Python linting tools overview

There are a number of syntax and error checker tools for Python and [this](https://stackoverflow.com/questions/1428872/pylint-pychecker-or-pyflakes) discussion on SF compares them in great details, so won't repeat here. Choice of `pylint` was somewhat influenced by these discussions, but it was more like, used it, found it useful and just started using it.

# A simple `pylint` Workflow

It's easiest to get started with `pylint` by something as simple as -

`pylint modulename.py` or `pylint packagename` and if one is running `pylint` for the first time, chances are the code will get rated at a very low value (don't feel terrible if it reports a number less than 5.0 out of 10.0). `pylint` categorizes issues it observes with the code into five categories -

1. Convention : The code is not following the [pep8](https://www.python.org/dev/peps/pep-0008/) convention and some more.

2. Refactoring : Possible refactoring possible (identifying duplicate code etc.)

3. Warnings : Something that's bad - which is going to make the code smell bad eventually, but not necessarily an error.

4. Error: Syntax error or undefined variable (something you just cannot and should not ignore).

5. Fatal: For some reasons, `pylint` reported a fatal error and couldn't continue.

`pylint` being highly configurable uses a default configuration about the kind of messages it reports and kind of messages it ignores. A [detailed list of pylint messages](http://pylint-messages.wikidot.com/all-messages) is available here. What defaults `pylint` uses can be found out by running `pylint --generate-rcfile`. The details of configuration file can be read about in the [documentation](http://pylint.pycqa.org/en/1.8/). Often though, it's probably a good idea to tweak the configuration file as we'll see in the next section. Based upon the number of lines analyzed and the total message count then `pylint` assigns a 'score' to the code. `pylint` also generates a detailed report for the errors and can track a bit of history (current score compared to previous score) of executions.

# A detailed `pylint` Workflow

One of the best ways to start using `pylint` is to generate our own `pylint` configuration file and then tweak it to our own needs. We'd go through a simple workflow about how to do it, but this is something that is best tailored for individual project.

First start with a default `pylintrc` file generated through `pylint --generate-rcfile` and then tweak it to your own environment, preferences or coding standards. Instead of looking at all the possible tweaking options, we are mainly going to look in subsequent sections about options or messages that affect your 'score' and are likely to be potential cause of trouble in future if ignored.

## Enabling and Disabling Certain messages

There are many ways how one could enable or disable which `pylint` messages are reported - 1) Through command-line, 2) Through configuration file and 3) through editor directives. What I usually follow is to do it through configuration and only occasionally resort to inline directives, but avoid using command-line directives except when one wants to experiment with which directives to use (and finally add them to the configuration file.).

Here are some of the guidelines that I follow -

1. Start with enabling everything

2. Disable certain types of Convention and Refactor messages (this is usually a matter of taste and is a function of individual project, so probably not a good idea to recommend which are recommended ones.).

3. Enable ALL Warnings, Errors and Fatal messages. A note: I always disable `fixme` warnings because I almost always have a few `FIXME`s in the code and they shouldn't unnecessarily lower the score. Another nagging warning is a `global-statement`, which I often don't disable in the `pylintrc` file, but sometimes may disable it in a given file using the editor directive. As something like global statements should best be left to the decision of code reviewer upon whether (s)he is fine with it or not.

Once this is setup, it's probably a better idea to track the `pylintrc` file in the VCS, so subsequent invocations can use this file and there will be consistency in the way the code is analyzed. We'll look at an example of integrating this with your git based repository to ensure that the code gets checked by `pylint` every time you make a commit.

## A bit more about `pylint` Score

`pylint` provides a score to your Python code out of 10. This code is based on a formula from the `pylintrc` file that can be customized if you want to. The default formula looks something like -

```
evaluation=10.0 - ((float(5 * error + warning + refactor + convention) / statement) * 10)
```
However, coming from `gcc` `-Werr` world, it's better to treat warnings just as badly as errors as warnings are often a size of accumulating technical debt in your codebase. So I often use a slightly modified evaluation formula that looks like -

```
evaluation=10.0 - ((float(5 * error + 5 * warning + refactor + convention) / statement) * 10)
```
A side note - There might be certain warnings which we might want to disable (e.g. `fixme` warnings) as discussed above, but there are certain warnings worth paying attention to. For instance there is a warning about logging 'logging-not-lazy' - in fact a little curiosity about this specific warning after running `pylint` on one of my python projects, lead me to actually look at `pylint` little more seriously than I would have. A side side note - There are a number of things about `logging` module that should really qualify a separate blog post, but more about it some other time.

When we look at integrating with the `git commit`, we'll also look at how warnings can be treated as errors.

## Usage Summary `pylint`

1. Start with default configuration and save your `pylintrc` file in the VCS.

2. Enable / disable messages as per your particular workflow. Recommended: do not disable errors and treat almost all warnings as errors.

3. If required customize 'evaluation' to treat warnings as errors.

4. Recommended is to integrate `pylint` checking with commit to the VCS repository.

# Integrating `pylint` with git

`git` provides hooks in the workflow, which can be used for various purposes. We will make use of a `git pre-commit hook` to make sure that we run `pylint` on all modified Python files in a project. A hook can run any program and if returns an error (non-zero exit status), the particular action doesn't proceed. So we want to run `pylint` and allow it to go ahead if there are zero warnings and zero errors and zero fatal messages. `pylint` return status is a bit weird, it basically returns a value where a particular bit is set if there were messages of that type. A sample shell script below can be used to achieve this.

```bash
#!/bin/bash

PYLINT=venv/bin/pylint

TOPLEVEL=`git rev-parse --show-toplevel`

PYLINTRC=${TOPLEVEL}/.pylintrc

PYLINT_OPTS="--rcfile=${PYLINTRC}"

PYTHON_FILES=$(git diff --name-only --cached --diff-filter=ACM | grep '\.py$')
echo "Running Pylint ...."
echo "${PYLINT} ${PYLINT_OPTS} ${PYTHON_FILES}"
${PYLINT} ${PYLINT_OPTS} ${PYTHON_FILES}
RESULT=$?

ALL_RESULT=$(( $((${RESULT}&4)) || $((${RESULT}&2)) || $((${RESULT}&1)) ))
if [[ ${ALL_RESULT} -eq 0 ]]; then
    echo "pylint: Looks Okay."
    exit 0
else
    echo "pylint: Errors or Warning. Fix them first..."
    exit -1
fi

```

Note: in the example script above, we check for only modified Python files, since we do not want to run `pylint` if we only modify `README.md` file. Since, it's a better idea to track this file in the VCS as well.

## Batteries included in `pylint`

We have only scratched the surface of `pylint` usage, `pylint` invocation can be customized in many ways. For instance there are a number of configuration options that can be tweaked to a particular environment. Another important feature of `pylint` that we have not looked at so far is `pylint` plugins. Plugins provide a mechanism to extend the kind of checking `pylint` can perform on your code. A very good example of `pylint` plugins is [pylint django plugin](https://pypi.org/project/pylint-django/). It is highly recommended to use this plugin if you are working on Django project.

# Summary

So far we looked at `pylint` as a code linting tool and have seen it's usage and integration with `git`. For ensuring and enforcing code quality of Python codebase, this is an excellent tool. What we have discussed so far only covers a broad overview and certain ways of integrating `pylint` in your development workflows. This should serve as a good starting point for a deeper dive into `pylint`.
