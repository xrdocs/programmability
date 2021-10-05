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

# Installing YANG Suite

First, let's install YANG Suite. Let's start by cloning the github repo where the code is stored.

You will need to install Docker in order to use YANG Suite. You can find more information on how to get Docker and how to install it [here](https://docs.docker.com/get-docker/).
{: .notice--info}

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
  
# Learning YANG models from a remote repository
  
Let's say your entire backbone is running IOS XR 7.3.1. You're trying to find a way to collect the serial number of all devices on your backbone. In this second section, we are goign to see how we can download all YANG models from a remote repository, find the ritht model to use in order to collect the serial number.
  


* GET from repo
* Add and GET from node
* Explore YANG
* Understand dependencies
* Diff: what changed between two versions of a model?
* Generate and play RPC
* Export the call as Python script
