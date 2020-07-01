.. title: Using Python Context Manager for Profiling
.. slug: python-profiling-context-manager
.. date: 2017-08-24 12:57:04 UTC+05:30
.. tags: Python
.. link:
.. description:
.. type: text
.. status:
.. type:
.. author: Abhijit Gadgil
.. summary: Profiling your code to identify hotspots or potential performance issues can prove quite useful. Python provides a `cProfile` package in the standard library provides this functionality for deterministic profiling. Usually, one might want to have an ability to turn profiling on and off at the run-time if possible. We explore a mechanism based on context managers in Python (Python `with` syntax) to be able to do so.

*Note: References to Python 2.7 need to be updated to latest version of Python. This post is about 3 years old. Will be updating it soon.*

# Introduction

Recently, while implementing a class for a project, we needed to profile parts of the code as some of the observations [were quite un-intuitive](/blog/2017/vector-operations-are-fast-right/). It was also important to identify the parts of code that were expensive and the kind of time they would take (strictly speaking profiling using `cProfile` won't be able to help much here.). Broadly the requirements can be summarized as follows -

1. Ability to add profiling information in several places in code. (Something like `cProfile.Profile().enable()` and `cProfile.Profile.disable()` done frequently.)

2. Ability to enable profiling globally or not, so when globally disabled, it should add a minimum overhead.

3. Being able to keep related profiling information with the code itself.

Initial idea was to have a class wide `cProfile.Profile()` object and keep enabling and disabling it for the parts of the code that was to be profiled. However, this approach was a bit problematic and didn't look very elegant as -

1. The code gets littered with `enable`/`disable` code.

2. Also, to meet requirement 2. above, this would have been wrapped in an `if` statement

3. Dumping stats only for some part of the code - in a section would become a little irritating to handle (see below).

For this type of problems, Context Managers in Python look like a good choice. All the `enable`/`disable` logic, `if` conditions etc. can be neatly wrapped inside the `__enter__` and `__exit__` methods and then selective profiling can simply be used as

```python
with Profiler(contextstr="----- foo profiling ----", enabled=True) as p:
     # Run the profiled code here

```
As can be seen, the readability of such a code is way better than say -

```python

if self.profiling_enabled:
	self.cprofiler.enable()

# do something

if self.profiling_enabled:
  self.cprofiler.disable()

```

# Implementation

The complete implementation for the `class Profiler` looks like following taken from [tickerplot utils](https://github.com/gabhijit/tickerplot/blob/master/tickerplot/utils/profiler.py).

```python

import cProfile
import StringIO
import pstats

class Profiler(object):

    def __init__(self, enabled=False, contextstr=None, fraction=1.0,
                 sort_by='cumulative', parent=None, logger=None):
        self.enabled = enabled

        self.contextstr = contextstr or str(self.__class__)

        if fraction > 1.0 or fraction < 0.0:
            fraction = 1.0

        self.fraction = fraction
        self.sort_by = sort_by

        self.parent = parent
        self.logger = logger

        self.stream = StringIO.StringIO()
        self.profiler = cProfile.Profile()

    def __enter__(self, *args):

        if not self.enabled:
            return self

        # Start profiling.
        self.stream.write("\nprofile: {}: enter\n".format(self.contextstr))
        self.profiler.enable()

        return self

    def __exit__(self, exc_type, exc_val, exc_tb):

        if not self.enabled:
            return False

        self.profiler.disable()

        sort_by = self.sort_by
        ps = pstats.Stats(self.profiler, stream=self.stream).sort_stats(sort_by)
        ps.print_stats(self.fraction)

        self.stream.write("\nprofile: {}: exit\n".format(self.contextstr))

        return False

    def get_profile_data(self):

        value = self.stream.getvalue()
        if self.logger is not None:
            self.logger.info("%s", value)

        return value


if __name__ == '__main__': # pragma: no cover

    import re

    with Profiler(enabled=True, contextstr="test") as p:
        for i in range(1000):
            r = re.compile(r'^$')

    print(p.get_profile_data())


    try:
        with Profiler(enabled=True, contextstr='exception') as p:
            raise ValueError("Error")
    except ValueError:
        print(p.get_profile_data())


    profiling_enabled = False
    with Profiler(enabled=profiling_enabled, contextstr='not enabled') as p:
        for i in range(1000):
            r = re.compile(r'^$')

    print(p.get_profile_data())

```

The code above is quite simple and extremely readable. Also as can be seen quite easy to plug-in in a running code. There are some important things to be kept in mind -


1. A quirk of `pstats.Stats` Constructor. If we pass a `Profile` object to the `pstats.Stats` Constructor as above, unless the `Profile` object is `enable`d, the Constructor raises an Exception that looks like -
```bash

  File "/usr/lib/python2.7/pstats.py", line 81, in __init__
    self.init(arg)
  File "/usr/lib/python2.7/pstats.py", line 95, in init
    self.load_stats(arg)
  File "/usr/lib/python2.7/pstats.py", line 124, in load_stats
    % (self.__class__, arg))
TypeError: Cannot create or construct a <class pstats.Stats at 0x7f683bec19a8> object from <cProfile.Profile object at 0x7f683bf1a6e0>
```

2. Just like the normal `__exit__` mechanism, returning `False` would make sure, if the running code within a `with` block raised an exception, that gets re-raised (as we often do not want to interfere with the Application's exception handing.).

3. This works quite well in a nested 'context' (ie. a `with Profiler()` inside a `with Profiler()`), which is kind of cool.

# Summary

This is quite a handy utility class, that I often plugin whenever I want to add some profiling to the code on the fly.
