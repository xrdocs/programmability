---
published: true
date: '2023-07-13 19:56 -0400'
title: Untitled
position: hidden
author: Rahul Sharma
---


# Background

<div style="text-align: justify">
A lot has been heard about 'Streaming Telemetry' in recent times, and it is known to be used to manage network devices. However, the question arises: Why is it needed when SNMP is already available to monitor and configure network devices?

<br>
  
Knowing SNMP, its architecture, basic commands, and how it works is common knowledge. However, it's also essential to be aware of certain limitations associated with SNMP, which can act as bottlenecks for network device management. 

</div> 
<hr>
<div style="text-align: justify"> 
  
Here are some of these limitations:

</div> 

<div style="text-align: justify"> 

  
**1. Limited Data Granularity:** SNMP's limited data coverage and predefined polling intervals hinder real-time monitoring and analysis of network conditions, as it may not capture comprehensive data or transient events occurring between intervals.

</div> 
<hr>  
<div style="text-align: justify"> 
  
  
**2. Polling Overhead:** SNMP's polling mechanism, involving periodic data requests from the management system, adds network traffic and overhead, impacting performance, especially in large-scale networks with numerous devices.

</div> 
<hr>  
<div style="text-align: justify"> 
  
**3 .Unreliable Transport:** SNMP traps use UDP for transport. UDP is inherently unreliable. If a trap doesn't reach a data collector, the information will be lost.

</div>   
<hr>  
<div style="text-align: justify"> 
  
**4. Scalability Challenges:** SNMP encounters scalability challenges as the number of managed devices grows, requiring the management system to handle connection maintenance, polling intervals, and data processing, which can strain resources and hinder efficient management of large networks.

</div> 
<hr>  
<div style="text-align: justify"> 
  
**5. Limited Event-Driven Monitoring:** Due to its reliance on polling, SNMP is less adept at capturing and responding to event-driven conditions, resulting in potential delays in detecting and reporting critical network events or anomalies unless they coincide with polling intervals.

</div> 
<hr>  
<div style="text-align: justify">   
**6. Lack of Flexibility:** Extending or modifying the hierarchical data model of SNMP, defined by MIBs, involves complex and time-consuming updates to both the management system and network devices. This process limits flexibility in adding new metrics or adapting to evolving network requirements.

</div> 
<hr>  
<div style="text-align: justify"> 
  
These limitations are manageable for small number of devices, but when it comes to managing a network with 1000s of devices, it becomes challenging. There is a need to have an alternative that addresses these limitations. And that's is when 'Streaming Telemetry' comes into the role.

</div>   
<hr>

# What is Streaming Telemetry?

Streaming telemetry represents a modern and efficient approach to network monitoring and data collection, where network devices autonomously **push** real-time operational data to a central management system (or collector) using frameworks like **gRPC** .

This continuous streaming enables granular visibility into network behavior, facilitates prompt anomaly detection, optimization of resources, and informed decision-making for network management and troubleshooting.

![SNMPvsST.png]({{site.baseurl}}/images/SNMPvsST.png)

![SNMP-vs-ST-tabular.png]({{site.baseurl}}/images/SNMP-vs-ST-tabular.png)

**gRPC** is an open-source protocol led by Google, combines Remote Procedure Calls (RPC) with Protocol Buffers to enable efficient communication between systems. Protocol Buffers define the structure of data, allowing for the generation of code to create or parse byte streams representing the structured data. The binary data format employed by gRPC reduces the size of the actual data by approximately 10 times. This compressed data is transmitted over HTTP/2, which supports multiplexing, facilitating concurrent handling of multiple requests using a request/response mechanism. Additionally, gRPC can utilize TLS encryption to ensure secure communication over HTTP/2.


![gRPC-client-server.png]({{site.baseurl}}/images/gRPC-client-server.png)
# YANG Models + Streaming Telemetry = Model-Driven Telemetry

**YANG Models**

YANG model is a hierarchical data structure (like tree) that consists of nodes that can be managed/monitored on network device.

These are two classes of YANG Models:

1. Cisco Native Models: Cisco creates and manages these models. They are fairly comprehensive and cover nearly everything that can be configured via a CLI. There are over 1300 native YANG models as of July 2023. 

These models are Cisco-exclusive and can only be used with Cisco devices.These models are further categorised into different types:

 - Configurational: These models are used to modify network device configurations.
        
 - Operational: These models are used to retrieve a network device's operational state. These models are read-only, which means you cannot change their configuration.
 
 - Cisco has a third model type known as the Unified model. These are similar to config models, but they share the same abstraction layer as CLI, making them CLI friendly. Someone who understands the IOS-XR CLI will find it easier to understand these models than the config model, which has a different abstraction layer than the CLI and thus is more difficult to understand its hierarchy.

2. OpenConfig Models: OpenConfig models are created by the OpenConfig forum, led by Google and consisting of companies like Meta, Apple, Microsoft, AT&T, Comcast, and more. These models serve as a common baseline for all network vendors, such as Cisco, Juniper, Arista, and others. 

They are designed to be vendor-neutral, meaning they can work with any network device regardless of the manufacturer. However, it's important to note that these models have limited coverage in terms of the data they can manage.


**Model-Driven Telemetry**

When streaming telemetry is combined with YANG models as a data definition language, it’s known as MDT. In order to establish an MDT, a gRPC channel has to be created between network device and collector.

There are two types of MDT with respect to the network device. 

1. When the gRPC channel is initiated by collector or when network device ‘gets in’ the gRPC channel request, it known as **Dial-in** MDT.

2. When the gRPC channel is initiated by collector or when network-device ‘sends out’ the gRPC channel request, it is known as **Dial-out** MDT.

![dial-in-dial-out.png]({{site.baseurl}}/images/dial-in-dial-out.png)

In order to establish MDT, we need to create sensor-group, destination-group and subscription. Let's discuss more about them in the following lines:

**Sensor-group** is a group of sensor-paths that corresponds to the data that network device sends to the collector. This can be summarised as **What data to send** 

**Destination-group** is a group of collectors that collect data from network devices. It comprises of parameters such as collector IP address, username and password and encoding (one out of JSON-IETF, GPB etc) for each of the collector. This can be summarised as **Where to send the data**

**Subscription** binds a sensor-group to a destination-group. Once this is created, an MDT is established where the network device starts streaming 'sensor-group' to 'destination-group'.

Let's now implement all the we have learned so far.
