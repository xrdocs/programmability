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

Manipulating configuration can be done with pyATS but it might not be the ideal tool for you. Here, we use pyATS because we are in a lab and we can't break anything (most important, we don't care if we do). Based on your use case, you might consider other tools like Ansible or NSO.
{: .notice--info}

# Network Topology

In today's article, we will use the Segment Routing network topology, based on XRd. It's available [on this github repo](https://github.com/ios-xr/xrd-tools/blob/main/samples/xr_compose_topos/segment-routing/docker-compose.xr.yml).

You can read more on XRDocs about what's XRd and how to use it [here](https://xrdocs.io/virtual-routing/tutorials/).
{: .notice--info}