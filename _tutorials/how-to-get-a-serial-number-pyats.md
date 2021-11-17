---
published: true
date: '2021-11-17 16:21 +0100'
title: How to get a serial number - pyATS
author: Antoine Orsoni
excerpt: How to get a serial number on an IOS XR device using pyATS?
tags:
  - Programmability
  - Python
  - pyATS
  - Automation
position: hidden
---
{% include toc icon="table" title="Table of Contents" %}

Recently, I got a query from a Customer: how could I easily collect my device(s) serial number? At first, he question sounded silly: you could just do `show inventory all` on any IOS XR platform to get the platform serial number. What if you need to do it 100 times per day? What if you need to do it on 100 devices at once? The goal of this new series of article is to explain different ways to collect a serial number on a device. If you can do it with a serial number, you can do it with anything else! In thirs first episode, we will use **pyATS**.

If you are not already familiar with pyATS and you want to know how to install it and how to use it, have a look at my pyATS series below.
https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/
{: .notice--info}

# Using the Devnet sandbox

In order for everyone to be able to run the code, we will use the [IOS XR always-on sandbox on Cisco Devnet](https://devnetsandbox.cisco.com/RM/Diagram/Index/e83cfd31-ade3-4e15-91d6-3118b867a0dd?diagramType=Topology). Below the sandbox information.

| Key               	| Value                    	|
|-------------------	|--------------------------	|
| IOS XRv 9000 host 	| sandbox-iosxr-1.cisco.com |
|     SSH Port      	|     22                 	|
|     Username      	|     admin                	|
|     Password      	|     C1sco12345           	|


# Collecting the serial number using CLI

To all make sure we are all on the same page, below is the command to collect the serial number with CLI on an IOS XR device and a sample output. In this case, the answer we want to get is `SN: 8F21767F3A3`.

<script src="https://gist.github.com/AntoineOrsoni/025aefa2afbeefd77d7b0a0f3ec909d1.js"></script>

# Collecting the serial number using pyATS

## Testbed definition

First, we need to create a testbed. 

Testbed definition has been covered in more details in this post: https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/
{: .notice--info}
