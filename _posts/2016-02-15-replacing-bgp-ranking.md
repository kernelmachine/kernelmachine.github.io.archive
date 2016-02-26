---
layout: post
title: IPV4
---

Here at Rapid7 we have incredible access to internet-scale data via sonar^1 and heisenberg^2.
Investigating these data gives us a birds eye view of the current state of the internet, public scanning activity, and the network security landscape.

Recently, I've become interested in extending these resources to include maps of IPV4 allocation and network providers.

But first, a primer on IPv4, the fourth version of the Internet Protocol.

The birth of the Internet, nicely documented by various books^3, involved the allocation of a fixed number of IP addresses. Because IPV4 addresses are 32-bit, there are:

$$ 2^32 = 4, 294, 967, 296 \text{ IP addresses in IPV4. }

Public demand for IP addresses lead to the exhaustion of allocable IPv4 addresses in 2011, something thought impossible at the Internet's inception.

In fact, the Internet is a large marketplace of territory. Service providers sell blocks of IP addresses at fixes prices that depend on the number of IP addresses in the block. Let's take a look at how this works.

An IP block has two components: a base address and a prefix. The base address is fixed, and the prefix is a mask that determines how many variants of the base address exist in the block.

For example, an IP address: 4.3.2.1/24 is denoted by the following:

$$ \text { 1 1 1 1 1 }  \text { 1 1 1 1 1 } \text { 1 1 1 1 1 } \text { 1 0 0 0 0 } $$
$$ \text { 1 1 0 1 1 }  \text { 1 0 1 1 1 } \text { 1 1 1 0 1 } \text { 0 0 0 0 0 } $$

Security is one of few disciplines that has a *lot* of associated data


^1: https://sonar.labs.rapid7.com/
^2:https://community.rapid7.com/community/infosec/blog/2016/01/05/12-days-of-haxmas-beginner-threat-intelligence-with-honeypots
