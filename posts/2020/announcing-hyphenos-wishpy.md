.. title: Announcing :  hyphenOs - wishpy
.. slug: announcing-hyphenos-wishpy
.. date: 2020-08-13 10:27:00 UTC+05:30
.. tags: Python, Networking, Wireshark, libpcap
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status:
.. summary: 'wishpy' or Python bindings for Wireshark and libpcap is an open source library, that brings Wireshark's dissectors and libpcap's packet capturing to Python as Pythonic APIs. This can then be used along with Python's rich data analysis and visualization ecosystem.

**Wi**re**sh**ark and libpcap meet **Py**thon! A developer preview is [now available](https://github.com/hyphenOs/wishpy){:target="\_blank"}

# Why 'wishpy'?

Wireshark is an excellent tool for analyzing network traces mainly thanks to a rich set of Protocol dissectors it supports - nearly **1500** protocols. Utilities like `tshark` exploit this feature set. While Wireshark is great at doing the dissection part, programmatically using these dissectors outside wireshark's codebase is a kind of an involved task. There are a few issues -

1. The APIs are provided in C only - So those APIs are available only in the C world.
2. The APIs are hardly documented and do suffer from being quite tightly coupled with the way they are used in Wireshark.
3. To overcome this `tshark` provides a number of command line switches, but even there the functionality is available only if it is available in `tshark` (eg. `tshark` has two switches for json one `-T json` and `-T ek`, one for getting the json output and another for getting [`Elastic`](https://www.elastic.co/){:target="\_blank"} compliant output!!)

It will help a lot if this dissection functionality is available -

1. Outside `wireshark`'s code base
2. Through a set of programmer friendly APIs that can be leveraged to build other solutions.
3. In a more high level language, popular language - say Python

Enter **'wishpy'**! 'Wishpy' also stands for - I **wish** such a functionality was available in **Py**thon world!

There are other projects notably [`scapy`](https://scapy.net/){:target="\_blank"}, but in terms of support for Protocol dissectors they are considerably limited. So clearly, there is a merit in bringing the Wireshark's dissectors to the Python World.

# Wishpy - The Basics

Actually 'wishpy' is not the first attempt at providing Python bindings for Wireshark. There is a project called [`wirepy`](https://github.com/lukaslueg/wirepy){:target="\_blank"}, but it is not maintained any more and even the APIs that are supported are not very Pythonic. 'Wishpy' is not for writing Protocol dissectors in Python but instead use the Protocol dissectors written in C in Python with more Pythonic APIs. We'll come to that in a bit.

'Wishpy' provides the extensions to Wireshark library using [`cffi`](https://cffi.readthedocs.io/en/latest/){:target="\_blank"} module for Python. One advantage of this approach is - it is easier to track the development of upstream `Wireshark` and make the features supported in the upstream easily available to `wishpy`.

'Wishpy' also wraps [`libpcap`](https://tcpdump.org/){:target="\_blank"} library for packet capture.

'Wishpy' works with Python queues (and for that matter any queues which have simple `get`, `put` abstractions), thus making Packet processing functionality a composable Pipeline.


# Features

Wishpy is in active development and right now it primarily works on Linux, but we have a proof of concept working on Windows for dissecting pcap files. This is an early release and supports following features -

1. Dissected Packets are available as `json`. The output is compatible with what `tshark` provides. 'Wishpy' also treats the field types semantically, that is the fields that are integers, floats or booleans are not just wrapped into a `json` string, the way it is done by `tshark`.

2. Supports pretty printing of `json`.

3. Supports version 2.6 and 3.2 of Wireshark. The actual version that is used is determined on the target machine where wishpy is installed.

4. Supports a GCD of APIs supported by `libpcap` versions 1.7, 1.8 and 1.9.

5. Wishpy also provides [`pcapy`](https://github.com/helpsystems/pcapy){:target="\_blank"} compatible APIs, thus it's a drop in replacement for `pcapy`, at-least for the main APIs.

6. Wishpy works with `pypy3` as well. See below for more details.

7. Supports simple Pythonic `Dissector` and `Capturer` [APIs](https://wishpy.readthedocs.io/en/latest/api.html){:target="\_blank"}.

8. Clean separation between application and library. [This example](https://github.com/hyphenOs/wishpy-examples){:target="\_blank"} demonstrates how json outputs of Packets can be dumped to a Redis Queue.

# Early Performance

We have conducted some profiling and performance experiments. These experiments suggest that using Wishpy with CPython may be about 3-4 times slower than the `tshark` that is built in C. This does not look very promising, _prima facie_. However, when we switch to **`pypy3`** using it's `jit` capabilities, we often do **as well as if not better than `tshark`** that ships with Wireshark! This makes 'wishpy' very promising because one can get a performance at par (or better) with `tshark` and offering all the flexibility and rich ecosystem of Python's data processing capabilities.

See below for performance results from a PCAP file with large packets (data download), but not a very deep dissection tree per packet.

Summary: **Pypy3: fastest --> 'tshark': little more than twice slower --> 'CPython': Nearly 4 times slower**.

```bash


# 'wishpy' with CPython 3.5
$ venv/bin/python examples/tshark_timed.py  ~/pcaps/Capture1.pcap
Running dissector for 50000 Packets.
Total time taken in seconds: 25.690130949020386

# 'wishpy' with PyPy3 (Python 3.6)
$ venv-pypy/bin/python examples/tshark_timed.py  ~/pcaps/Capture1.pcap
Running dissector for 50000 Packets.
Total time taken in seconds: 6.10959005355835

# 'tshark'
$ echo "Running 'tshark' for 50000 packets." && \
	date +%H:%M:%S.%N && tshark -T json -r ~/pcaps/Capture1.pcap -c 50000 > /dev/null \
	&& date +%H:%M:%S.%N
Running 'tshark' for 50000 packets.
11:03:36.033859533
11:03:49.346423480

```

See below for performance results from a PCAP file with small packets (protocol data), but with a deep dissection tree per packet.

Summary: **Pypy3: fastest -> 'tshark': nearly 90% slower --> 'CPython': Little more than 4 times slower**.
```bash

# 'wishpy' with CPython 3.5
$ venv/bin/python examples/tshark_timed.py  ~/pcaps/Capture2.pcap
Running dissector for 50000 Packets.
Total time taken in seconds: 42.661524534225464

# 'wishpy' with PyPy3 (Python 3.6)
$ venv-pypy/bin/python examples/tshark_timed.py  ~/pcaps/Capture2.pcap
Running dissector for 50000 Packets.
Total time taken in seconds: 9.754892110824585

# 'tshark'
$ echo "Running 'tshark' for 50000 packets." && \
	date +%H:%M:%S.%N && tshark -T json -r ~/pcaps/Capture2.pcap -c 50000 > /dev/null \
	&& date +%H:%M:%S.%N
Running 'tshark' for 50000 packets.
11:14:32.583997783
11:14:50.733474335

```

This is just a quick experiment to understand the performance of 'wishpy'.

Honestly, it started as how bad the performance is compared with `tshark`. However, thinking more about it, I realized, this particular problem might indeed be a 'sweet spot' for `pypy`'s `jit` capabilities. I won't go on and claim that it runs **faster** than the native `tshark` as yet, because to be fair in the comparison above, `tshark`'s output is redirected to `/dev/null`, so it has to take efforts to generate the output which is then ignored, thus not making this a fair comparison. However, I didn't come across an option where I could run `tshark` in something like `--silent` mode. Which again suggests, why having something like `wishpy` makes sense as one is not limited by CLI switches that are offered by `tshark` alone.

In future, we'll talk about a detailed comparison of all the three approaches.

# Help Welcome

'Wishpy' is still an early project and focus so far has been on getting an early preview version out, with a set of basic capture and dissection features. Help is surely needed in terms of trying it out on different Linux distributions. Also MacOS support might work, but is not tested at the moment.

Also, if you would like to use 'Wishpy' for a use-case that you have in mind where 'Wishpy' might be well suited, but is missing some features, please file an issue on the [github](https://github.com/hyphenOs/wishpy/){:target="\_blank"}.

Wishpy is available at -

 - [Repository](https://github.com/hyphenOs/wishpy){:target="\_blank"}
 - [Documentation](https://wishpy.readthedocs.io/){:target="\_blank"} - (early!)

Happy protocol dissecting!! :-)
