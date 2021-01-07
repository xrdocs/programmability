---
published: true
date: '2021-01-07 22:40 +0100'
title: pyATS series - Install and use pyATS
author: Antoine Orsoni
excerpt: Installation of pyATS and collecting your first CLI output
tags:
  - iosxr
  - cisco
  - pyATS
position: hidden
---

# pyATS overview

Ever dreamed of a test framework that could be used across multiple platforms, OS and vendors, which could do regression, sanity and feature testing; already used by thousands of engineers and developers worldwide? Guess what, it exists, it’s free, and you can start using it right now!

pyATS was first created as an internal project, to ease the validation of two OS versions. It has been made public in 2017 through Cisco Devnet.

This blog post will be the first one of a series on pyATS. Today, we will explain what’s pyATS, install pyATS and cover a basic use case (getting a CLI output from a XR device). More use cases are going to be covered in the next posts. 

The code used for each blog post can be found [here](https://github.com/AntoineOrsoni/xrdocs-how-to-pyats). This link will include the code for all posts. Today’s part will refer to the `0_get_cli_show` folder of the repo.

## Building blocks

pyATS is made of three main building blocks:
- **pyATS**, the core block of this ecosystem. It’s a Python framework which leverages multiple Python libraries such as [Unicon](https://pypi.org/project/unicon/), providing a simplified connection experience to network devices. It supports CLI, NETCONF and RESTCONF. It enables network engineers and developers to start with small and simple test cases 
- **pyATS libraries** (also known as Genie) which provides everything you need for network testing such as parsers, triggers and APIs. 
- **XPRESSO**, the pyATS Web UI Dashboard.

![pyATS ecosystem]({{site.baseurl}}/https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/_images/layers.png){: .align-center}

You can read more about pyATS ecosystem in the [official documentation](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/intro/introduction.html).

## Supported OS

This solution is built and thought from the ground up to be an agnostic ecosystem. As of January 2021, it comes out of the box with libraries for the below OS:

- IOS,
- IOS XE,
- IOS XR,
- NXOS,
- ASA,
- Linux,
- JUNOS,
- SROS,
- BIGIP,
- Viptela OS,
- DNA Center.

# Getting your hands dirty

Enough talking, how do YOU start using pyATS?

## pyATS requirements

Full requirements can be found in the [official documentation](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/prereqs/prerequisites.html).

### Hardware

pyATS is lightweight and scalable. As per the documentation, you need:
- 1GB of RAM,
- 1 vCPU.

### Operating System
 
It rusn in a Linux and Linux-like environments, such as Ubuntu, CentOS, Fedora and macOS. The pyATS ecosystem does **not** support Windows.

### Python version

As of January 2021, it requires Python version between 3.5 and 3.8. Version 3.9 is NOT yet supported.

![pyats_ready.png]({{site.baseurl}}/images/pyats_ready.png){: .align-center}

## pyATS installation

pyATS ecosystem can be installed in two ways: in a docker container or in a virtual environment. Today, we will focus on the second option. Remember, you need a Python version between 3.5 and 3.8 to use pyATS.

Full installation documentation for Docker and Virtual Environment can be found [here](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/install/installpyATS.html).

Let’s start! Open a bash terminal and run the below three commands. It will:
- Create a virtual environment.
- Activate the virtual environment.
- Install pyATS and its dependencies.

**From your bash terminal**
<div class="highlighter-rouge">
<pre class="highlight">
<code>
python -m venv venv
source venv/bin/activate
pip install pyats
</code>
</pre>
</div>

## Your first pyATS use case: getting a CLI output from a device

In this first use case, we are going to see step by step how we can get a **simple CLI output** (`show ip interface brief`) from a IOS XR device. This first use case do **not** demonstrates the full power of pyATS but should be a good example to cover the basics.

In order for everyone to be able to run the code, we will use the [IOS XR always-on sandbox on Cisco Devnet](https://devnetsandbox.cisco.com/RM/Diagram/Index/e83cfd31-ade3-4e15-91d6-3118b867a0dd?diagramType=Topology). Feel free to adapt the code to use your own device(s). Below the sandbox information.

| Key               	| Value                    	|
|-------------------	|--------------------------	|
| IOS XRv 9000 host 	| sbx-iosxr-mgmt.cisco.com 	|
|     SSH Port      	|     8181                 	|
|     Username      	|     admin                	|
|     Password      	|     C1sco12345           	|

## Building a testbed

The simplest way to connect to a device is through a pyATS testbed file, written in YAML. This information will be used by **Unicon** to connect to the device and send/get the requested commands.