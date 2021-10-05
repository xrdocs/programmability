---
published: true
date: '2021-09-19 10:39 +0200'
title: YANG Suite - Do you speak YANG?
position: hidden
author: Antoine Orsoni
excerpt: >-
  Presentation of YANG Suite tool, to retrieve and observe YANG models from your
  network devices.
tags:
  - NETCONF
  - YANG
  - YANG Suite
  - Automation
  - Python
---

You have been talking SNMP for years with your network devices and you've faced many limitations as discussed in [RFC3535](https://datatracker.ietf.org/doc/html/rfc3535#page-4)? You understand YANG could definitely help you tackle these challenges but you don't yet speak YANG? Then YANG Suite is the right tool for you!

YANG Suite was first developped as an internal Cisco project. In February 2021, the wait was over and it has been made available for everyone.

YANG Suite provides network operators with a common tool to interact with Cisco IOS XE, IOS XR, and the NX-OS Network Operating Systems as they look to modernize their network management and migrate from traditional network management tools. Its core features include YANG model browsing and exploring, as well as device management. On top, you can add many plugins such as NETCONF, gRPC and Diff to enrich its capabilities.

![English YANG translate_4.png]({{site.baseurl}}/images/English YANG translate_4.png){: .align-center}

# What is a model?

TODO TODO TODO

# Installing YANG Suite

First, let's install YANG Suite. Let's start by cloning the github repo where the code is stored.

<code>
git clone https://github.com/CiscoDevNet/yangsuite
</code>

To setup YANG Suite (generate a certificate, choose a login/password), you will need to run the below commands. It's only required for the **first** utilisation. 

<code>
cd yangsuite/docker/ 
./start_yang_suite.sh
</code>

Then, start YANG Suite by entering the below commands:

<code>
cd yangsuite/docker/ 
docker compose up
<code>

The **nginx** container (web server) redirects port 80 to port 8433 which is used to interface with the YANG Suite core. You can now connect to http://localhost or https://localhost:8443 to access YANG Suite.
  
You will need to install Docker in order to use YANG Suite. You can find more information on how to get Docker and how to install it [here](https://docs.docker.com/get-docker/).
{: .notice--info}

The complete documentation on how to install YANG Suite is available [here](https://github.com/CiscoDevNet/yangsuite).
{: .notice--info}
  
# Learning YANG models
  
Let's say your entire backbone is running **IOS XR 7.3.1**. You're trying to find a way to collect the serial number of all devices on your backbone. In this second section, we are goign to see how we can download all YANG models from a remote repository, find the ritht model to use in order to collect the serial number.
  
## Adding a new YANG repository
  
To add a new YANG repository, from which we can sync our YANG models, here are the steps to follow:
  1. Go to **Setup > YANG Files and repositories**
  2. Click **New repository** in order to create a new local folder where we will add our YANG models
  3. Give it a name
  4. Click **Create repository**
  
  ![Add remote repository.jpg]({{site.baseurl}}/images/Add remote repository.jpg){: .align-center}

## Cloning YANG models from a remote repository
  
In this first scenario, we are going to clone a remote repository. Today, we are going to use this remote repository: https://github.com/YangModels/yang. Feel free to use another one.
  
Did you know that all YANG models for all Cisco IOS for all versions are stored on [https://github.com/YangModels/yang](https://github.com/YangModels/yang) ?
{: .notice--info}
  
To clone a remote repository, follow the below steps:
  1. Click on **Git** to use it as our way to collect YANG models
  2. Add the **Repository URL** (https://github.com/YangModels/yang), the **Git branch** (master) and the **directory** where the YANG models are stored (vendor/cisco/xr/731). You can nagivate in the repository and use the directory you like; for example if you need to collect YANG models for another IOS XR version.
  3. Click **Import YANG files** to start cloning the repository.
  
![Add remote repository 2.jpg]({{site.baseurl}}/images/Add remote repository 2.jpg){: .align-center}

This could take a few minutes, depending on how many models are in the repository. For me, it took around 5 minutes. Once done, you should see YANG models from the remote repository appear on the box on the left (4).
  
## Cloning YANG models from a device
 
Alternatively, you can also clone YANG models directly from a device. When a client (your device) and a server (YANG Suite) initiate a NETCONF session, they exchange **Hello messages** listing the set of capabilities they support.
  
For example, below is an example of capabilities of a device running IOS XR 6.5.3.
  
<script src="https://gist.github.com/AntoineOrsoni/f401cb979a81b62798c8f022b8f064d6.js"></script>
  
You can get a similar output using CLI with the command: `ssh username@host -p 830 -s netconf` where 830 is the NETCONF port on the device.
{: .notice--info}
  
* GET from repo
* Add and GET from node
* Explore YANG
* Understand dependencies
* Diff: what changed between two versions of a model?
* Generate and play RPC
* Export the call as Python script
