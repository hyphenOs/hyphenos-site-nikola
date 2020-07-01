.. title: Python Logging Under The Hood
.. slug: python-logging-under-the-hood
.. date: 2018-07-08 12:57:04 UTC+05:30
.. tags: Python
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: Python Logging module is perhaps one of the most widely used and often not so well understood module. While most of the things are documented quite well, sometimes it's easy to miss out a few things. Here we take a look at Python's Logging module in a bit more details, trying to get under the hood to figure out what's really happening.

Note: This is an old posts and will be updated for Python 3 soon.

# Introduction

One of the reasons I got a little curious about logging module was thanks to `pylint`. While I was running `pylint` on some code, I saw at quite a few places the following warning message -
```
logging-not-lazy (W1201): *Specify string format arguments as logging function parameters*
```

While my code had a few instances of equivalent of the following -

```
log.info("Log Something val %d" % (val))
```
While the message was saying I should probably use something like
```
log.info("Log Something val %d" , val)
```

Now, the natural question is why should that matter? The answer is "While the former string substitution will be evaluated regardless of whether a particular error is enabled or not, ie to say this particular log record will be emitted on not, while the latter is a function call that happens only when the log record is 'emitted'.

Suggested different option of using `string.format` is also not a choice here as well for the same reason and you would see a similar warning as above.

```
:logging-format-interpolation (W1202): *Use % formatting in logging functions and pass the % parameters as arguments*
```

So this prompted in taking a more closer look at the documentation and figure out some of the subtleties. We discuss all the findings in little more details.

# A Bit More About These Warnings

As we have seen in the previous section, it is important to pass parameters as arguments to logging function rather than using '%' substitution or using `string.format`. Let's take a look at this in little more detail by profiling the code. Let's run this through our [context profiler](https://gabhijit.github.io/python-profiling-context-manager.html). Here's the relevant code and corresponding details -

```
import logging
import time

import profiler

iterations = 1000000
l = logging.getLogger(__name__)
l.addHandler(logging.NullHandler())

l.setLevel(logging.INFO)

class Foo(object):
    def __str__(self):
        return "%s" % self.__class__

with profiler.Profiler(enabled=True, contextstr="non lazy logging") as p:
    for i in range(iterations):
        l.debug("Hello There. %s" % Foo())

p.print_profile_data()

with profiler.Profiler(enabled=True, contextstr="lazy logging") as p:
    for i in range(iterations):
        l.debug("Hello There. %s", Foo())
p.print_profile_data()
```

And here is the output of running the above code

```
profile: non lazy logging: enter
         4000003 function calls in 1.058 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
  1000000    0.231    0.000    0.655    0.000 /usr/lib/python2.7/logging/__init__.py:1145(debug)
  1000000    0.272    0.000    0.424    0.000 /usr/lib/python2.7/logging/__init__.py:1360(isEnabledFor)
  1000000    0.385    0.000    0.385    0.000 lazy_logging.py:13(__str__)  <-------------- Check this line
  1000000    0.152    0.000    0.152    0.000 /usr/lib/python2.7/logging/__init__.py:1346(getEffectiveLevel)
        1    0.018    0.018    0.018    0.018 {range}
        1    0.000    0.000    0.000    0.000 /home/gabhijit/backup/personal-code/gabhijit-github-io-blog/sources/misc/python/logging/profiler.py:29(__exit__)
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}


profile: non lazy logging: exit

profile: lazy logging: enter
         3000003 function calls in 0.591 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
  1000000    0.195    0.000    0.580    0.000 /usr/lib/python2.7/logging/__init__.py:1145(debug)
  1000000    0.245    0.000    0.385    0.000 /usr/lib/python2.7/logging/__init__.py:1360(isEnabledFor)
  1000000    0.140    0.000    0.140    0.000 /usr/lib/python2.7/logging/__init__.py:1346(getEffectiveLevel)
        1    0.010    0.010    0.010    0.010 {range}
        1    0.000    0.000    0.000    0.000 /home/gabhijit/backup/personal-code/gabhijit-github-io-blog/sources/misc/python/logging/profiler.py:29(__exit__)
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}


profile: lazy logging: exit
```
So our `__str__` for the class Foo was called even when the `logging.debug`  is not enabled. This might look like a convoluted example, but this proves an important point, especially when we are logging in critical path, extra cycles are spent in evaluating the string even when the relevant level is not enabled. When the parameters to be evaluated are themselves more expensive this could be an un-necessary penalty, so this should be avoided.

Surely, it's important to understand logging module in more details as this is going to be widely used in any program or library that we develop.

Let's look at the logging architecture in little more details through use of some examples and what exactly is handling under the hood.

# Logging Architecture

The complete logger flow is described below, taken from the excellent [logging howto](https://docs.python.org/2/howto/logging.html)



We quickly go through the main classes `Logger` and `Handler` class, to provide a quick overview. Reading in more details about the HOWTO pointed above is highly recommended. We are not going to look at some of the other standard classes here a) as they are well covered in the documentation above and b) Typically you'd just use what the standard provides. The intent here is not to re-produce documentation, but look at gotchas that are worth keeping an eye about.

## `Logger` Class

From the HOWTO above -

> Logger objects have a threefold job. First, they expose several methods to application code so that applications can log messages at runtime. Second, logger objects determine which log messages to act upon based upon severity (the default filtering facility) or filter objects. Third, logger objects pass along relevant log messages to all interested log handlers.

So basically, Logger classes are main entry point into the logging system. A more simple description of `Logger` class can be, they create `LogRecord` objects depending upon current configuration and handle them. The main function that does this looks like following (taken from `/usr/lib/python-2.7/logging/__init__.py`)

```
    def _log(self, level, msg, args, exc_info=None, extra=None):
        """
        Low-level logging routine which creates a LogRecord and then calls
        all the handlers of this logger to handle the record.
        """
        if _srcfile:
            #IronPython doesn't track Python frames, so findCaller raises an
            #exception on some versions of IronPython. We trap it here so that
            #IronPython can use logging.
            try:
                fn, lno, func = self.findCaller()
            except ValueError:
                fn, lno, func = "(unknown file)", 0, "(unknown function)"
        else:
            fn, lno, func = "(unknown file)", 0, "(unknown function)"
        if exc_info:
            if not isinstance(exc_info, tuple):
                exc_info = sys.exc_info()
        record = self.makeRecord(self.name, level, fn, lno, msg, args, exc_info, func, extra)
        self.handle(record)

    def handle(self, record):
        """
        Call the handlers for the specified record.

        This method is used for unpickled records received from a socket, as
        well as those created locally. Logger-level filtering is applied.
        """
        if (not self.disabled) and self.filter(record):
            self.callHandlers(record)

```

A couple of points to note here - `_log` function will be called looking at the current logging `LEVEL` of the logger and the handlers are called only if this is logger is `enabled` and any of the 'filters' associated with this logger allow the record to be passed. [Question - why do all the work - when the logger is disabled?](https://stackoverflow.com/questions/50453121/logger-disabled-check-much-later-in-python-logging-module-whats-the-rationale). Actually, the above was acknowledged [as an issue](https://bugs.python.org/issue33606) and is fixed [for Python 3.8](https://github.com/python/cpython/commit/6e3ca645e71dd021fead5a70dc06d9b663612e3a).

## `Handler` Class

This class actually 'handles' the LogRecords emitted. Typically by calling the 'Formatter' for the LogRecord and then `emit`ing the record. A word about `handle` here is important - The Handler's `handle` method calls `emit` holding the Handler's lock (see below), so it's important to pay attention to `emit` and ensure as far as possible that it is not a blocking one. Some of the implementation of handlers actually have an `emit` function that is blocking. Python 3.2 onwards there is a QueueHandler that implements a non-blocking `emit`. So it might be a good idea to consider using that especially if you are logging in a critical path. Here is the Handler's `handle` function (taken from `/usr/lib/python2.7/logging/__init__.py`)

```
    def handle(self, record):
        """
        Conditionally emit the specified logging record.

        Emission depends on filters which may have been added to the handler.
        Wrap the actual emission of the record with acquisition/release of
        the I/O thread lock. Returns whether the filter passed the record for
        emission.
        """
        rv = self.filter(record)
        if rv:
            self.acquire()
            try:
                self.emit(record)
            finally:
                self.release()
        return rv
```
# Creating Loggers

## What happens upon `import logging`

When you do `import logging`, there are a number of things that happen under the hood. For example a `RootLogger` instance is created when the `logging` module is imported. Actually all the loggers created in a Python process form a hierarchy with `RootLogger` being at the 'root' of the hierarchy. We will look at more about this a bit later, in a somewhat convoluted example, but helps us understand what is really happening.

## Logging instead of 'print'

If you often `print` for debugging and providing some information to the user, it's often a good idea to simply create a logger and do `basicConfig`, so moving to a proper logging later on becomes easier and you don't have to manually move every `print` call to a logging call. So something like -

```
import logging
logger = logging.getLogger() # This will return the `RootLogger` above
logging.basicConfig() # output to stderr
logger.setLevel(logging.INFO)

# And then in the code -

logger.info("Doing something...")
```

Typically `print` should be used when you are developing a command line utility and want to output to `stdout`. Note above the `logging.basicConfig` default configuration is to configure output (it's actually an internally a `StreamHandler`) to ~`stdout`~ `stderr`.

## Creating Logger for Package and Subpackage

A Logger can be created by -

```
logger = logging.getLogger("something")
```

Usually it's a good idea to pass the `__name__` of the current module to the logger. This has an important benefits that loggers of a package are arranged in a hierarchy. Let's say you have a package structure that looks like `foo.bar.baz` then inside each of the `__init__.py` if the logger is created through `logging.getLogger(__name__)` then the logger corresponding to package `foo` will be parent of the logger corresponding to package `foo.bar` which in turn will be parent of `foo.bar.baz` logging. Some of the advantages of that are - when actually the log records are emitted one precisely knows which module/package the log came from and one doesn't have to worry about what name to assign to the logger. There's another detail here, each `Logger` has got a property called `propagate` (default is `True`). If this property is `True` then for a given logger whenever a `LogRecord` is to be handled handlers attached to that particular logger as well as those attached to the 'parent' are called as well, (Note: here handlers) regardless of the `level` of the parent logger(s). Thus what one can do is at a package level, create a handler and all the sub-packages and modules within that package would just use this handler and no separate handler is required to be created and assigned to logger. This serves as a convenience and helps when defining loggers for a library (see below for details about it).

### Loggers for libraries

All the discussion above for the package and sub-packages applies here as well, another thing to keep in mind when working on a library which will be potentially used by someone else, it's a good idea to leave the 'actual logging configuration' to the application that will be using this library, so what we should really do is - simply create loggers and don't attach any handlers, whatever handlers are required to be attached, will be done by the application code that will utilize the function of the library. To avoid an Exception related to 'no handlers attached', it's a good idea to attach a `NullHandler` to the logger created by the main package.

In Summary - following is a good example of creating a logger for a library you are developing

Let's say you are developing a library that will be available as a package `foo` with sub packages `bar` and `baz` inside the bar. Following simple logging setup should be good.

```
# Inside foo/__init__.py

import logging
logger = logging.getLogger()
logger.addHandler(logging.NullHandler())

# Inside foo/bar/__init__.py

import logging
logger = logging.getLongger()


# Inside foo/bar/baz/__init__.py

import logging
logger = logging.getLogger()

```

And then inside your Application, you can simply -

```
# Get access to RootLogger

_logger = logging.getLogger()
_logger.addHandler(logging.StreamHandler()) # Handler is attached to root logger.

our_logger = logging.getLogger("app")

```

It's possible to have a much more [detailed configuration](https://docs.python.org/2/library/logging.config.html) of the logging. But something above should often serve good enough a lot of times.

What this will do is - it will create a structure like the following (and default `propagate=True` will make sure the `RootLogger`'s Handler gets called.

```


RootLogger (StreamHandler)
\
 \_ AppLogger

\
 \_ Foo.logger
   \_ Bar.logger
     \_ Baz.logger
```

# Summary

We explored some of the details of Python's `logging` module and looked at some good practices worth following while implementing logging for your application. This should serve handy while reading and understanding the documentation and perhaps dealing with sometimes unexpected behaviors.

