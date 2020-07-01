.. title: Linux Virtual Interfaces
.. slug: linux-virtual-interfaces
.. date: 2018-02-23 12:57:04 UTC+05:30
.. tags: Linux Networking
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: Linux supports `veth` and `tun/tap` types of virtual interfaces, which are used by VMs, Containers for providing networking. This post summarizes certain findings as a result of experimenting these virtual interfaces along with Linux's Ethernet bridge.

## Background

Virtual interfaces in Linux are often used to provide networking to Containers and Virtual Machines. This is typically achieved by creating a virtual interface, assigning it to a VM/Container, connecting it to a bridge and in turn connecting it to a physical interface if required. Typically network namespaces are used to provide isolation between the VMs/Containers.
 While there's enough documentation around net, but there are subtleties.

## `veth` Virtual Interface

This is a simple Ethernet interface that is always created as a pair of Ethernet interfaces, so the idea here is very simple, packets sent on one of the pair are received on other and vice versa. A typical use-case for this is to establish connectivity between different namespaces or connecting a network namespace outside the host through a bridge. Some interesting observations here -

If we create a `veth` pair and keep both ends in the same network namespace and try to ping from one side to another ping works fine, but we don't see any packets on either of the interfaces, which is kind of surprising. The reason this happens is - any address assigned to Linux machine (to any interface) gets assigned to a local table and in the case of 'ping' above, since both the interfaces are in the same namespace (and hence same local table), the packets don't flow through the `veth`s but through `lo` interface.

```

# create veth pair and assing IP address.

ip link add veth0 type veth peer name veth1
ip addr add 10.1.0.1/24 dev veth0
ip addr add 10.1.0.2/24 dev veth1

ping 10.1.0.2 -I 10.1.0.1

PING 10.1.0.2 (10.1.0.2) from 10.1.0.1 : 56(84) bytes of data.
64 bytes from 10.1.0.2: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 10.1.0.2: icmp_seq=2 ttl=64 time=0.043 ms
64 bytes from 10.1.0.2: icmp_seq=3 ttl=64 time=0.040 ms

# list all interfaces

ip link

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp3s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 1000
    link/ether 88:d7:f6:93:2b:00 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
    link/ether 88:78:73:97:c6:88 brd ff:ff:ff:ff:ff:ff
27: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 66:01:80:3c:7d:70 brd ff:ff:ff:ff:ff:ff
28: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f6:a7:c5:e4:5a:7b brd ff:ff:ff:ff:ff:ff

# Strange thing above is - both veth0 and veth1 are down and yet the ping works
# This is because of the 'local routing table' see below.

ip route show table local
local 10.1.0.1 dev veth0  proto kernel  scope host  src 10.1.0.1
local 10.1.0.2 dev veth1  proto kernel  scope host  src 10.1.0.2
<snipped>

# bring 'up' the interfaces
ip link set veth0 up
ip link set veth1 up

# No packets on veth0 and veth1 but packets are seen on lo

tcpdump -i veth0 -n icmp

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel

tcpdump -i veth1 -n icmp

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel

tcpdump -i lo -n icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
15:52:38.940699 IP 10.1.0.1 > 10.1.0.2: ICMP echo request, id 23800, seq 126, length 64
15:52:38.940717 IP 10.1.0.2 > 10.1.0.1: ICMP echo reply, id 23800, seq 126, length 64

```

To be able to bring the interface up in a `veth` pair, both the interfaces should be set up using `ip link set vethX up`.

If we have to force packets to go through the `veth` interface, we have to assign one of them to a different network namespace. This ensures that local routing table lookup is bypassed and then packets flow through the `veth` interfaces. This can be achieved as follows -


```
# create veth pair

ip link add veth0 type veth peer name veth1

# create a network namespace

ip netns add test

# attach one of the veth interfaces to the network namespace

ip link set veth0 netns test

# bring up the interfaces (note below we've to now run ip netns exec test for veth0
ip link set veth1 up
ip netns exec test ip link set veth0 up

# assign IP addresses
ip addr add 10.1.0.2/24 dev veth1

ip netns exec test ip addr add 10.1.0.1/24 dev veth0

# now ping (this pings from veth1 end to veth0). If we want to ping from veth0 to veth1 we've to run ping inside 'ip netns exec test'
ping -I 10.1.0.2 10.1.0.1

# check tcpdump on veth0 and veth1

tcpdump -i veth1 -n

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
16:14:59.228785 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 21, length 64
16:14:59.228817 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 21, length 64
16:15:00.252694 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 22, length 64
16:15:00.252722 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 22, length 64
16:15:01.276645 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 23, length 64
16:15:01.276674 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 23, length 64
16:15:02.300690 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 24, length 64
16:15:02.300721 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 24, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel


ip netns exec test tcpdump -i veth0 -n

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
^C16:15:45.308779 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 66, length 64
16:15:45.308807 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 66, length 64
16:15:46.332711 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 67, length 64
16:15:46.332739 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 67, length 64
16:15:47.356645 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 68, length 64
16:15:47.356670 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 68, length 64
16:15:48.380769 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 69, length 64
16:15:48.380798 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 69, length 64
16:15:49.404682 IP 10.1.0.2 > 10.1.0.1: ICMP echo request, id 24880, seq 70, length 64
16:15:49.404714 IP 10.1.0.1 > 10.1.0.2: ICMP echo reply, id 24880, seq 70, length 64

10 packets captured
10 packets received by filter
0 packets dropped by kernel

```

Typically the way `veth` interfaces are used is to connect one end to a network namespace and another end to a bridge. Interesting observation here though is - If we do this and assign IP address to one of the `veth` pairs say `veth1`, then the ping doesn't work. When looked in details why this happens, it is not clear. However if we assign the IP address to the br0 interface, then the ping works. Why this happens is not quite clear.

### `veth` Interfaces summary

1. Use them to connect two network namespaces
2. Always one has to bring both ends of the `veth` pair up for the interface to be up.
3. If we attach one of the interface to a bridge, then the IP address needs to be attached to the bridge interface and not the attached interface.

## `tun/tap` Virtual Interface

Mainly `tun` and `tap` interfaces are used to inject IP packets to/from kernel from userspace. The way this typically works is a `/dev` entry is created when a process binds to an interface and process can simply `read/write` from the `/dev/` for the packet transfer. A more detailed look at how tun/tap interfaces work is given in [this link](http://backreference.org/2010/03/26/tuntap-interface-tutorial/) and it is worth following that for details. `tap` interfaces are typically used by Virtual Machines, where full Layer 2 headers are also used. For other applications like `openvpn`, typically make use of `tun` interface type.

To be able to use `tap` interface, one has to bind to the tap interface. The link above shows one example or a similar example from `qemu` source is [availble here](https://github.com/qemu/qemu/blob/08a63553161d9d12fba217f07af49f986b487e9a/net/tap-linux.c#L41)

## References

Following is a list of links on the web that I referred to while experimenting with Virtual Interfaces.

1. [http://www.naturalborncoder.com/virtualization/2014/10/17/understanding-tun-tap-interfaces/]()
2. [https://lists.linuxfoundation.org/pipermail/bridge/2011-June/007711.html]()
3. [https://stackoverflow.com/questions/25641630/virtual-networking-devices-in-linux#34773334]()
4. [http://www.opencloudblog.com/?p=66]()
5. [https://unix.stackexchange.com/questions/122468/how-does-one-capture-traffic-on-virtual-interfaces]()
6. [https://serverfault.com/questions/585246/network-level-of-veth-doesnt-respond-to-arp?newreg=fa2af6ce40ad43318ac4b32054741cd7]()
7. [http://backreference.org/2013/06/20/some-notes-on-veth-interfaces/]()
8. [https://serverfault.com/questions/743466/tcpdump-on-bridge-interface-virbr-does-not-receive-any-packets-destined-for-on]()

