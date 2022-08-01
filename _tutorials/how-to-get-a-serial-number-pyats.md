---
published: true
date: '2022-07-29 16:21 +0100'
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

Recently, I got a query from a Customer: how could I easily collect my device(s) serial number? 

At first, he question sounded silly: you could just do `show inventory all` on any IOS XR platform to get the platform serial number. What if you need to do it 100 times per day? What if you need to do it on 100 devices at once? The goal of this new series of article is to explain different ways to collect a serial number on a device. If you can do it with a serial number, you can do it with anything else! 

In thirs first episode, we will use **pyATS** (**Py**thon **A**utomated **T**est **S**ystems, to be pronounced "py A. T. S.") was first created as an internal project, to ease the validation of two OS versions. It has been made public in 2017 through **Cisco Devnet**.

pyATS is made of three main building blocks:
- **pyATS**, the core block of this ecosystem. It’s a Python framework which leverages multiple Python libraries such as [Unicon](https://pypi.org/project/unicon/), providing a simplified connection experience to network devices. It supports **CLI**, **NETCONF**, **RESTCONF** and **gRPC**. It enables network engineers and developers to start with small and simple test cases.
- **pyATS libraries** (also known as Genie) which provides everything you need for network testing such as parsers, triggers and APIs. 
- **XPRESSO**, the pyATS Web UI Dashboard.

If you are not already familiar with pyATS and you want to know how to install it and how to use it, have a look at my pyATS series below.
https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/
{: .notice--info}

The code for this series of posts will be published here:
https://github.com/AntoineOrsoni/how-to-get-serial-number/
{: .notice--info}

# Other "How to get a serial number" episodes

You’ve missed an episode? You would like to read more? Below the list of published episodes:

| Episode 	| URL                                                                                              	| What's covered                                        	|
|---------	|--------------------------------------------------------------------------------------------------	|-------------------------------------------------------	|
| **1 - Install and use pyATS**       	| [Link](https://xrdocs.io/programmability/tutorials/how-to-get-a-serial-number-pyats/){: .btn}  	|  Using pyATS to get a serial number on a given IOS XR device 	|

# Using the Devnet sandbox

In order for everyone to be able to run the code, we will use the [IOS XR always-on sandbox on Cisco Devnet](https://devnetsandbox.cisco.com/RM/Diagram/Index/e83cfd31-ade3-4e15-91d6-3118b867a0dd?diagramType=Topology). Below the sandbox information.

| Key               	| Value                    	|
|-------------------	|--------------------------	|
| IOS XRv 9000 host 	| sandbox-iosxr-1.cisco.com |
|     SSH Port      	|     22                 	|
| NETCONF port		 	| 		830 				|
|     Username      	|     admin                	|
|     Password      	|     C1sco12345           	|


# Collecting the serial number using CLI

To all make sure we are all on the same page, below is the command to collect the serial number with CLI on an IOS XR device and a sample output. In this case, the answer we want to get is `SN: 8F21767F3A3`.

<script src="https://gist.github.com/AntoineOrsoni/025aefa2afbeefd77d7b0a0f3ec909d1.js"></script>

# Getting your hands dirty -- Collecting the serial number using pyATS

Enough talking, let's code!

![keyboard cat_small2.png]({{site.baseurl}}/images/keyboard cat_small2.png){: .align-center}

## Testbed definition

The simplest way to connect to a device is through a pyATS testbed file, written in YAML. This information will be used by **Unicon** to connect to the device and send the requested commands.

You can find the complete documentation on how to build a testbed [here](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/connection.html).
{: .notice--info}

In a nutshell, we need to specify how to connect to our device:
- IP address or URL,
- Credentials,
- Type, the Operating System of our device, in our case IOS XR,
- How to connect to our device, in our case SSH on port 22.

 Our testbed look like the below example:

<script src="https://gist.github.com/AntoineOrsoni/c837b0cc0d49c5be0f18232689eedd3e.js"></script>

Testbed definition has been covered in more details in this post: https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/
{: .notice--info}

## Leveraging pyATS parsers to get a Python dictionary

The power of the **pyATS libraries**: converting **raw output** (what you would get in a CLI output, printed earlier in this post) into a **parsed output** (dictionary) where you can easily get a value by accessing a specific key. Once parsed by pyATS, the output would look to something like below:

<script src="https://gist.github.com/AntoineOrsoni/dc87b1259a5f811e4a9394d9aa4481ae.js"></script>

To better understand the difference between a raw output and a parserd output, you can refer to this article:
https://xrdocs.io/programmability/tutorials/pyats-series-parsing-like-a-pro/
{: .notice--info}

## Using Python to get the value of a specific key

In Python, you can see a Dicitonary as a set of `key: value` pairs. In `my_dict`, in order to retrieve `my_value` associated with a specific `my_key`, you should use `my_value = my_dict['my_key']`. 

A value can be a dictionary. In this case, we call it a `nested dictionary`. In our example, the key `"module_name"` is associated with a dictionary. In our example, we have multiple nested dictionaries.

Dictionary keys are case sensitive!
https://docs.python.org/3/tutorial/datastructures.html#dictionaries
{: .notice--warning}

In our case, the code to get the Serial Number out of the parsed output should look something like: `serial_number = my_output["module_name"]["Rack 0"]["sn"]`. 

You can read more about Python dictionaries in the documentation.
https://docs.python.org/3/tutorial/datastructures.html#dictionaries
{: .notice--info}

## 



# pyATS pro and con's to retrieve a serial number
