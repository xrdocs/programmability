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
| **6 - Pushing and removing configuration**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-adding-and-removing-configuration/){: .btn} 	|  pyATS Tips and Tricks |

# Abstract

In this new episode, we will see how to add configuration to a device, confirm the changes (looking at operational data) and then clean the configuration. We will then compare three ways to remove configuration to go back to default. We will do all this changes using pyATS. In the last part, we will combine the different building blocks to have a complete script.

We will manipulate Segment Routing Traffic Engineering in today's use case. We will send traffic from a source to a destination and influence the traffic's path using Segment Routing Policies.

Manipulating configuration can be done with pyATS but it might not be the ideal tool for you. Here, we use pyATS because we are in a lab and we can break anything (most important, we don't care if we do). Based on your use case, you might consider other tools like Crosswork, NSO or Ansible.
{: .notice--info}

# Network Topology

In today's article, we will use the Segment Routing network topology, based on IOS XRd. It's available [on this github repo](https://github.com/ios-xr/xrd-tools/blob/main/samples/xr_compose_topos/segment-routing/docker-compose.xr.yml).

The topology in the above repo has been slightly changed to adapt to our use case. For instance, we are using proper IOS XRd routers as source and destination (not linux hosts), so we can benefit from all IOS XR networking features.
You can find those changes (each node `startup configuration` and `docker-compose.xr.yaml`) [here](https://github.com/AntoineOrsoni/xrdocs-how-to-pyats/tree/master/5_config_builder/ios%20xrd).

You can read more on XRDocs about what's IOS XRd and how to use it [here](https://xrdocs.io/virtual-routing/tutorials/).
{: .notice--info}

The topology will look like below. We will send traffic between a source and a destination (linux hosts) and influence the traffic's path using Segment Routing Policies.

![Topology_XRd_1.jpg]({{site.baseurl}}/images/Topology_XRd_1.jpg)

## IP Addressing

**Management IP** addresses are in the range `172.40.0.0/24`.
`101` to `108` are respectively associated from `xrd-1` to `xrd-8`
`200` and `201` are respectively associated to `xrd-source` and `xrd-dest`.

**Adjacency links** addresses are in the range `100.0.0.0/8`.
Second and third byte are associated with the two routers on each side of the link. Fourth byte represents the node.
Ex: `100.106.108.0/24` represents the link between `xrd-6` and `xrd-8`.
`100.106.108.108` belong to an interface on `xrd-8`.

**Source and Destination** addresses are respectively `10.1.1.2` and `10.3.1.3`.

# Pushing Configuration to a device

In this first step, we are going to see that pushing configuration with pyATS is easy. I would not suggest using this solution when pushing a complex end-to-end service where you would need advanced features such as rollback capabilities. In a lab environment, it's good enough and it has the advantage to be very simple.

Manipulating configuration can be done with pyATS but it might not be the ideal tool for you. Here, we use pyATS because we are in a lab and we can break anything (most important, we don't care if we do). Based on your use case, you might consider other tools like Crosswork, NSO or Ansible.
{: .notice--info}

Once connected to a device using pyATS, you can use the `configure()` method to push configuration. It takes a `string` as parameter, which is the configuration to be pushed. Below an example for a segment-routing policy. You can pass multiple lines of configuration to the `configure()` method.

<script src="https://gist.github.com/AntoineOrsoni/56032ff89ed5ca6de7ab836b09bdb72d.js"></script>

You can read more about how to configure devices with pyATS in the [documentation](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/quickstart/configuredevices.html)
{: .notice--info}

# Removing configuration from a device

Removing configuration can be slightly more complex than pushing configuration. In this part, we are going to explore 3 differerents ways to achieve that goal:
- Downloading a `base configuration template` on the device, and loading it.
- Doing a configuration rollback to a previous commit.
- Using `configure()` method to unconfigure what we have done (i.e. adding `no` at the beginning of each configuration section).

In all three scenarios, we will remove the segment routing policy configuration that we pushed in the previous part. We will compare the pros and cons for each option.

## Loading a configuration on the device

First option is to have a base configuration file saved on the device harddisk and `load` the configuration on the device. The IOS XR command to do it is `load harddisk:filename` where `filename` is the configuration file. Then, you have to commit the change using `commit replace` to first wipe out the configuration and then apply the fresh one.

To download the configuration on the device, I like to use SCP. On IOS XR, the command is: `scp username@server:/path/to/file harddisk:/filename` where:
- `username` is the username to connect to the `server`,
- `server` is the server where you want to download the configuration file,
- `/path/to/file` is the path to your configuration file on the `server`,
- `filename` is the filename you would like to use on the device.

If the configuration file is not on the device or if the `filename` has a typo, the `commit replace` will just wipe out the configuration and you will end with a factory default configuration. 
Double check that the file has been pushed on the device before executing the command.
{: .notice--warning}

You can find below an example of loading a configuration file from the device. For safety reason, we first use the **pyATS Library** to parse the `dir harddisk:` output to verify the file exists.

<script src="https://gist.github.com/AntoineOrsoni/7899d47846d73acf1fc3ad21c03af88e.js"></script>

What I like with the solution is the simplicity. Once the base configuration file is on the device, you can just wipe out the configuration and start from a clean base. No chance to have residual configuration. You can have as many base configuration files as you want, according to your use case. For example one base configuration for **segment routing**, one for **telemetry** etc.

I can see two cons for this solution. If your base configuration file changes, then you have to upload it again on each device. As we saw, if the file doesn't exist on the device, IOS XR will just wipe out the configuration. It's on you to double check the file is there and you can use **pyATS Library** to do so.

## Configuration rollback

Another way is to commit your base configuration with a **label** then use `configuration rollback to-exclude label-name` command to rollback the configuration. In this case `label-name` is the label you used in the commit with the command `commit label label-name`. 

It takes only one line of Python to do so.

<script src="https://gist.github.com/AntoineOrsoni/7bb17e80c75f352cf8a17311659c2a3b.js"></script>

This approach is efficient as it takes only one line of Python but it can be painful to manage commit labels on each device.

## Unconfiguring line by line


Last way that we can consider is to remove the configuration line by line: adding a `no` in front of each line we pushed. It takes more lines and probably more chances to forget to remove something but it can be handy if you need to partially change the configuration.

Here is an example:

<script src="https://gist.github.com/AntoineOrsoni/36dea353f00336425c1358b0e6b88c6f.js"></script>

# Combining all building blocks together

Now that we explored each building block, we can combine everything to have an example of end-to-end solution. The below script will be divided in three parts:
1. Verifying the traffic from `source` to `destination` follows the IGP and is not influenced by any policy.
2. Pushing the **Segment Routing Traffic Engineering policy** configuration and verifying the traffic path is following the policy.
3. Removing the configuration and getting back to default configuration. Verifying the traffic follows the IGP again.

To keep this example simple, we will only `execute()` a traceroute to check the path from `source` to `destination` and print the output (i.e. we will not parse it, as we don't have a parser today for traceroute on IOS XR). We will see in the next episode what you can do in this case.

Here is the code:

<script src="https://gist.github.com/AntoineOrsoni/be6c3b6ce29c382111e63d6842239158.js"></script>

# Conclusion

In this sixth episode of the pyATS series, saw how to push and remove configuration. We took a **Segment Routing Traffic Engineering policy** as example.

The code used for each blog post can be found [here](https://github.com/AntoineOrsoni/xrdocs-how-to-pyats). This link will include the code for all posts.
{: .notice--info}

# Resources

Below a few useful pyATS resources.

- [List of supported pyATS parsers](https://pubhub.devnetcloud.com/media/genie-feature-browser/docs/#/),
- [The official pyATS documentation](https://pubhub.devnetcloud.com/media/pyats/docs/getting_started/index.html),
- [List of Unicon supported platforms](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/supported_platforms.html),
- [Devnet code exchange](https://developer.cisco.com/codeexchange/),
- [Join the Webex space with the pyATS community](https://eurl.io/#r18UzrQVr).
