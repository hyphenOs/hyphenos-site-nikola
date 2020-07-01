.. title: OpenAirInterface with Linux Network Namespaces
.. slug: oai-oaisim-enb-epc-netns
.. date: 2018-05-27 12:57:04 UTC+05:30
.. tags: OAI
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary:  <a href="http://www.openairinterface.org/">OpenAirInterface</a> aims to develop open source software solutions for 4G and 5G cellular network, that would run on COTS hardware (typically x86/ARM based CPUs). The project provides an Air Interface Simulator (OAISIM) and implementations of eNodeB and the EPC. OAISIM allows one to test the software implementations of the Network components, without actually having access to the RF part. This blog post explores, how this can be done on a single decently high end desktop.



*Note: This is an old post and a lot has changed in the OAI Since. In a future post, I will update the approach with latest versions of OAI.*

# Intended Audience

A familiarity with LTE network architecture and some knowledge about different interfaces is certainly useful. Some background about Linux network namespaces is quite useful as well.

# Background

While there are many tutorials that allow you to connect OAISIM + eNodeB on one machine and connect it with EPC (MME/SGW and PGW) running on another machine, I was not able to find any good tutorial that allowed to connect these devices on the same machine. While, it's quite possible to do so by using two VMs on the same machine, two VMs still looked like sub-optimal as in theory it looked quite possible to do this using Network Namespaces to provide isolations for individual nodes. Further, using network namespaces would allow to containerize this quite easily. Since the development platform I am using is not exactly identical to the one for which documentation is available, I had to do some tweaking to build the components as well.

A bit of a background - OAI Wiki has [instructions to connect eNodeB with EPC](https://gitlab.eurecom.fr/oai/openairinterface5g/wikis/T/howtoconnectoaisimwithoaiepc) where both of them are running on separate machines. The instructions were for a rather old release ( v0.4). The instructions are for Ubuntu 14.04. My development machine is Ubuntu 16.04, so I wanted something that would work on Ubuntu 16.04, which means I had to rebuild the software on Ubuntu 16.04. Then we could leverage network namespaces to create a topology inside the Linux box to implement the [LTE Network Architecture](https://www.tutorialspoint.com/lte/images/lte_epc.jpg). The final goal is to be able to ping Internet from the emulated UE on the OAISIM.

# Building The Stack

The OAI codebase is actually made up of two repositories available from Eurocomm Gitlab -
1. The RAN side is implemented in the [openairinterface5g repository](https://gitlab.eurecom.fr/oai/openairinterface5g)
2. The CN side is implemented in the [openair-cn repository](https://gitlab.eurecom.fr/oai/openair-cn)


## Build RAN Side Components

I didn't follow the `-I` option available with the build scripts. The reason being, some of them are patching older versions of software, so just wanted to see how it goes without having to do that. Only a couple of dependencies were obtained from the source - asn1c and free diameter (see below), but everything else is a package available from a standard repository of Ubuntu 16.04 and nothing is patched. I particularly don't like patching older versions of dependencies with some non-standard sources, because that makes the build very fragile and only works on a very limited set of environments.

Did face a few issues to get the build successful -

1. The head of the develop branch was having issue with libconfig, so it was not possible to use the head of the develop branch.

2. The original version for which documentation was available for setting up OAISIM with EPC, was 0.4, however it was not building on Ubuntu 16.04. The problem was some of the ASN.1 files for RRC were patched with an older version of `asn1c` which was available for Ubuntu 14.04, so it was not possible to patch those files with `asn1c` available for Ubuntu 16.04. This required a little bit of figuring around to find out what was the latest enough and well tagged version. Version 0.6.1 works fine, that uses asn1c hosted on [eurocomm gitlab](https://gitlab.eurecom.fr/oai/asn1c).

3. Since both eNodeB and EPC will be running on same Kernel, a kernel version that supports GTP is required. GTP is available in the mainline Linux tree since Kernel version 4.7.0. My running kernel version was 4.15.0, so it should be possible to use the GTP module available in the kernel. However, the UE IP module inside the openairinterface5g repository that emulates the UE side was not getting built on the 4.15.0 kernel, this is because there was a slight change in the timer interface in the 4.15.X kernel. So I [patched](https://github.com/gabhijit/oai5g/blob/gabhijit-v0.6.1/openair2/NETWORK_DRIVER/UE_IP/device.c) the UE-IP module inside my repository, so that it got successfully compiled.

4. There was another issue - `libnettle` available on Ubuntu 14.04 was major version 2.X. The one available on Ubuntu 16.04 was 3.X and signatures of some of the functions were changed, so that was required to be fixed.

5. Also had to use the Free Diameter hosted on [eurocomm gitlab](https://gitlab.eurecom.fr/oai/freediameter) to build the CN side.

All of the changes are maintained in a [repository on github](https://github.com/gabhijit/oai5g). One of the reasons I am doing this and not up streaming those changes yet is - eventually my repository will have breaking changes from the original repository - especially the build system and the `-I` flag. Once I am able to fix it reasonably well, it might be a good time to up-stream those changes.

## Building EPC Components

Building of EPC components was quite straight forward. Just following the build scripts was enough. I at-least don't remember having to do a lot of tweaking to get it working.


# Running OAI

Once the required software components are built, a network topology as shown in figure below is created.

![OAI Topology](/blog/images/oai-sim-epc-topology.png "OAI Topology")


The scripts used to create this topology are [maintained in this repository](https://github.com/gabhijit/networking-experiments/tree/master/oai-oaisim-enb-epc). The details of how to setup and run all the nodes are specified in the README file in the location above.

In trying to run OAI inside the same Linux machine, we are making use of network namespaces to achieve that. So basically each network namespace provides isolation between the nodes and the nodes are connected using `veth` pairs and/or `bridge` interfaces wherever required. Figure below explains the topology and the networks we are using -

1. The eNodeB MME - S1-MME interface is implemented as a 10.0.0.0/24 network
2. The MME-HSS - S6A interface is implemented as a 10.0.1.0/24 network
3. The eNodeB-SGW - S1-U interface is implemented as a 10.0.2.0/24 network
4. The MME-SGW - S11 interface is implemented as a 10.0.3.0/24 network
5. The HSS-Host implements another interface for communication between the HSS and MySQL running in default (host) namespace.

Once all the setup is done, it should be possible to ping Internet from the simulated UE.

## Miscellaneous Notes

Some of the things in the above setup are less than ideal. Documenting those here for reference -

1. MySQL database is used by HSS. It is possible to run the database in the HSS namespace. However, the above setup does not do so. Once this is done, no additional configuration requirements for connecting to the database will be required.

2. On the egrees ie. SGi interface we are using MacVTAP to connect to external world. MacVTAP interfaces do not work so well with WLAN interfaces. So it is recommended to use a wired Ethernet interface to use MacVTAP on the egress. Right now, not entirely sure of what's the best way to deal with that problem, one choice would be to use separate network and bridge it to external world and apply some kind of SNAT. This needs some more work.

3. eNodeB does not run in it's own namespace. To be able to do that, the UE-IP driver should be patched to work in any network namespace. Right now it uses - `netlink_kernel_create` along with `&init_net` passed to it, which is the default namespace. This needs some more investigation. Update: This is fixed in `ue_ip.ko` kernel module and the latest changes are [updated in this repository](https://github.com/gabhijit/oai5g). This needs some more testing to say this works.


## Summary and Next Steps

Since we are able to run individual nodes inside separate network namespace, containerizing this should be kind of straight forward. Will explore this in a follow up.

If anyone wants to try this out on any platform, please do so and if you face any problems, please create issues in the repositories, so that I can fix those.

