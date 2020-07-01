.. title: Linux VRF with MPLS for L3-VPN
.. slug: linux-vrf-mpls
.. date: 2018-02-27 12:57:04 UTC+05:30
.. tags: Linux Networking
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: VRF support for Linux was added in kernel 4.5. In the Linux netdev 1.1 conference, there was a talk about this support, which showed one of the use-case as MPLS-VPN. This blog post tries to re-create the setup from demo on a kernel 4.15.

## Intended Audience

Some background about MPLS networking and VRF is definitely useful. Some understanding of Linux net-namespaces is definitely useful.

## VRF Background

Typically VRF is used in VPN implementations, where it is possible to have overlapping address-spaces between one or more customers, in such situations, the isolation required between customers is achieved using VRF. This can as well be achieved using VRF. This is explained in excellent details in [this blog post by cumulus networks](https://cumulusnetworks.com/blog/vrf-for-linux/).

## Re-producing Setup

One of the use-cases for VRF presented during [netdev 1.1]() was using VRFs with MPLS feature (also from Cumulus Networks). One of the things that I wanted to be able to do this is - to re-create the setup, so that it works on any Linux with kernel having the supported features. This exercise turned out to be slightly more involved than I thought before.

### The Topology

This setup is explained in the [document](https://www.netdevconf.org/1.1/proceedings/slides/ahern-vrf-tutorial.pdf) and is reproduced below

![VRF MPLS Topology](/blog/images/vrf-mpls-topology.png "Topology")


1. Each of the device (Host, CE, PE or P) lives in it's own `netns`. `veth` and `bridge` links are used as appropriate to connect the hosts (making sure also that their end-points are in the right `netns`).
2. Linux implementation of `vrf` is a `netdevice`. So one VRF `netdevice` is created for each of the customer in the same PE router, thus there are two VRF `netdevice`s in the PE router.
3. The physical devices (`veth` endpoints) that connect to the customer edge router (CE Router) are enslaved to the VRF `netdevice` corresponding to that customer.
4. For each of the VRF `netdevice`, there's a separate routing table setup for lookup and corresponding `l3mdev` rules are added for making sure the packet lookup happens in the correct routing table.

## Issues / Problems faced

While setting up most of the things was relatively easier, there were a few gotchas

1. It was not obvious, but MPLS labels can be setup only in the main routing table hence specifying table parameter throws `EINVAL` and as with many `RTNETLINK` errors, this one is a bit hard to figure out. Just figured this out by reading the appropriate code.
2. Unless you actually setup using `sysctl -w net.mpls.platform_labels=10000`, setting up label values throws an `EINVAL` error, so one has to first set this up. Basically the default value of `platform_labels` that is the max allowed label is 0. Hence we need to set it up to some high value, so label numbers we use will work. Note: maximum value for the label is 2^20.
3. The way I was going about creating most of the interfaces was, I was first creating an interface in the default namespace and then assigning the `netns` using `ip link set <dev> netns <nsname>`. While this works for `veth` devices, this doesn't work for `vrf` devices. My first impression was that setting VRF in a non-default `netns` is not allowed, while that is not the case, moving VRF from one netns to another is not allowed. So the best way to go about creating that is - to actually create the VRF device in the correct `netns` first time itself something like - `ip netns exec pe1 ip link add vrf-pe1-ce1 type vrf table 10`.
4. After everything was setup fine, the ping from c1 to c3 (see figure above) was not still working. While the setup looked all fine, was not able to ping from the hosts across MPLS network. The packets were not getting forwarded on one of the PE routers. A debugging using `tcpdump` showed that while the route was properly setup, the packets didn't show up on the corresponding interface (pe-p core). So I contacted original author of the tutorial (Dave Ahern of Cumulus Networks). He suggested to try and trace trace fib events `perf record -efib:* ` and then `perf script` to see for any clues. This actually helped. While, the lookup was working fine but what was observed was a subsequent `fib_validate_source` after the lookup was failing. This was because the VRF routing table didn't have a default route (which is always recommended) and it didn't have a route to the source subnet. Due to absence of this route, the source was treated as `martian`. Verified this using `sysctl -w net.ipv4.conf.all.log_martian=1`. Finally after adding the route to the source subnet the issue was resolved.
5. After all the above issues were resolved a rather odd issue showed up, while it was possible to ping across hosts, it was not possible to ping the host itself. This is because, when we create a `netns`, the loopback interface `lo` in that network namespace is not set as up by default. This is fixed by setting `ip netns exec netnsname ip link setup lo up` to be done in every network namespace created.

## Demo

Following [github repository](https://github.com/gabhijit/networking-experiments) has complete setup. Please give it a try and report if any issues.

## References

Some references used for this experiment -

1. [Tutorial PDF](https://www.netdevconf.org/1.1/proceedings/slides/ahern-vrf-tutorial.pdf)
2. [Tutorial Video](https://www.youtube.com/watch?v=zxPFFdRN_x4) - See VRF with MPLS use-case
3. Stack-Exchange discussion related to [MPLS labels](https://unix.stackexchange.com/questions/401719/rtnetlink-answers-invalid-argument-mpls-on-mininet)

## Summary

Network namespaces, virtual Ethernet and bridge interfaces allow creation of complex topologies within a single Linux box and hence can become an excellent tools for creating test-beds. Discussed in this blog was one application of such a test bed. As a next step, it would make a lot of sense to actually automate (or automate as far as possible) creation of complex topology starting from something like a 'dot' file.

