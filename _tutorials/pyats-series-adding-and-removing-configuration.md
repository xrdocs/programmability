---
published: true
date: '2023-05-04 11:07 +0200'
title: pyATS Series - Adding and removing configuration
author: Antoine Orsoni
excerpt: Adding and removing configuration from IOS XR device using pyATS
tags:
  - iosxr
position: hidden
---
{% include toc icon="table" title="Table of Contents" %}
![pyats_hello2.jpg]({{site.baseurl}}/images/pyats_hello2.jpg){: .align-center}

# Introduction

Ever dreamed of a test framework that could be used across multiple platforms, OS and vendors, which could do regression, sanity and feature testing; already used by thousands of engineers and developers worldwide? Guess what, **it exists, it’s free, and you can start using it right now!**

pyATS (**Py**thon **A**utomated **T**est **S**ystems, to be pronounced "py A. T. S.") was first created as an internal project, to ease the validation of two OS versions. It has been made public in 2017 through **Cisco Devnet**.

Many thanks to **Romain Cyrille**, Cisco CX Engineer, for his help writting this article!
{: .notice--info}

# Other pyATS episodes

You've missed the first episode? You would like to read more? Below the list of published episodes:

| Episode 	| URL                                                                                              	| What's covered                                        	|
|---------	|--------------------------------------------------------------------------------------------------	|-------------------------------------------------------	|
| **1 - Install and use pyATS**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/){: .btn}  	|  What's pyATS, Install pyATS, Collect a raw CLI output 	|
| **2 - Parsing like  a pro**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-parsing-like-a-pro/){: .btn} 	|  Explore pyATS libraries, Collect and parse a CLI output        	|
| **3 - Be a model**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-be-a-model/){: .btn} 	|  What a pyATS model and when to use it        	|
| **4 - Collecting many show commands**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-collecting-many-show-commands/){: .btn} 	|  How to collect many show commands on many devices? |
| **5 - Tips and Tricks**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-tips-and-tricks/){: .btn} 	|  pyATS Tips and Tricks |

# Abstract

In this new episode, we will see how to add configuration to a device, confirm the changes (looking at operational data). We will then compare three ways to remove configuration to go back to default. We will do all this changes using pyATS.

We will manipulate Segment Routing Traffic Engineering in today's use case. We will send traffic from a source to a destination and influence the traffic's path using Segment Routing Policies.

Manipulating configuration can be done with pyATS but it might not be the ideal tool for you. Here, we use pyATS because we are in a lab and we can't break anything (most important, we don't care if we do). Based on your use case, you might consider other tools like Crosswork, NSO or Ansible.
{: .notice--info}

# Network Topology

In today's article, we will use the Segment Routing network topology, based on IOS XRd. It's available [on this github repo](https://github.com/ios-xr/xrd-tools/blob/main/samples/xr_compose_topos/segment-routing/docker-compose.xr.yml).

The topology in the above repo has been slightly changed to adapt to our use case. For instance, we are using proper IOS XRd routers as source and destination (not linux hosts), so we can benefit from all IOS XR networking features.
You can find those changes (each node `startup configuration` and `docker-compose.xr.yaml`) [here](https://github.com/AntoineOrsoni/xrdocs-how-to-pyats/tree/master/5_config_builder/ios%20xrd).

You can read more on XRDocs about what's IOS XRd and how to use it [here](https://xrdocs.io/virtual-routing/tutorials/).
{: .notice--info}

The topology will look like below. We will send traffic between a source and a destination (linux hosts) and influence the traffic's path using Segment Routing Policies.

```
#                 xrd-7(PCE)
#                 /        \
#              xrd-3 --- xrd-4
#               / |        | \
#  src --- xrd-1  |        |  xrd-2 --- dst
#               \ |        | /
#              xrd-5 --- xrd-6
#                 \        /
#                 xrd-8(vRR)
#
#
# IP addresses
# source:            10.1.1.2
# xrd-1-GE2 (left ): 10.1.1.3
# xrd-2-GE2 (right): 10.3.1.2
# dest:              10.3.1.3
```

## Modifying the `docker-compose` File

To be able to connect via SSH to the two linux hosts (source and destination), I had to modify the `docker-compose` file. Below what the file looks like for the two hosts. What changed:
- Using the image `lscr.io/linuxserver/openssh-server:latest` where SSH server is already installed. Documentation [here](https://github.com/linuxserver/docker-openssh-server).
- In the `environment`, I set `PASSWORD_ACCESS=true` so I can access my device using a login/password. Otherwise I could only access it with a specific public RSA key.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
  dest:
    cap_add:
    - NET_ADMIN
    command: /bin/sh -c "ip route add 10.0.0.0/8 via 10.3.1.2 && /bin/sh"
    container_name: dest
    <mark>image: lscr.io/linuxserver/openssh-server:latest</mark>
    networks:
      xrd-2-dest:
        ipv4_address: 10.3.1.3
    stdin_open: true
    tty: true
    <mark>environment:</mark>
      <mark>- PASSWORD_ACCESS=true</mark>
      <mark>- USER_PASSWORD=cisco123</mark>
      <mark>- USER_NAME=cisco</mark>
  source:
    cap_add:
    - NET_ADMIN
    command: /bin/sh -c "ip route add 10.0.0.0/8 via 10.1.1.3 && /bin/sh"
    container_name: source
    <mark>image: lscr.io/linuxserver/openssh-server:latest</mark>
    networks:
      source-xrd-1:
        ipv4_address: 10.1.1.2
    stdin_open: true
    tty: true
    <mark>environment:</mark>
      <mark>- PASSWORD_ACCESS=true</mark>
      <mark>- USER_PASSWORD=cisco123</mark>
      <mark>- USER_NAME=cisco</mark>
</code>
</pre>
</div>
