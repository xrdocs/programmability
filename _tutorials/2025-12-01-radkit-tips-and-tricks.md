---
published: false
date: '2025-12-01 11:39 +0100'
title: Radkit - Tips and tricks
author: Antoine Orsoni
tags:
  - cisco
  - Radkit
position: hidden
---
{% include toc icon="table" title="Table of Contents" %}

# Introduction

RADKit is a Software Development Kit (SDK): a set of ready-to-use tools and Python modules allowing efficient and scalable interactions with local or remote equipment. RADKit is available at no additional cost with your existing Support Contracts.

# Useful links

| What?         | Link                                             |
|---------------|--------------------------------------------------|
| Documentation | https://radkit.cisco.com/docs/index.html         |
| Download      | https://radkit.cisco.com/downloads/              |
| Installation  | https://radkit.cisco.com/docs/install/index.html |

# Tips and tricks

## Printing device information

```python
service.inventory['device-name']
```

It will return device parameters, internal attributes, metadata and APIs (ex: NETCONF). Example below.

```python
Object parameters
--------------------  ------------------
identity              None              
serial                None              
name                  device-name
service_display_name  device-name
--------------------  ------------------

Internal attributes
key                    value                                         
---------------------  ------------------------------------------
description            NCS 540                                   
device_type            IOS_XR                                    
forwarded_tcp_ports                                              
host                   10.1.1.1                               
http_config            False                                     
netconf_config         False                                     
snmp_version           False                                     
swagger_config         False                                     
terminal_capabilities  ['UPLOAD', 'INTERACTIVE', 'EXEC' + 1 more]
terminal_config        True                                      

Metadata

APIs
-------  -------
Netconf  UNKNOWN
Swagger  UNKNOWN
-------  -------
```

## Filtering the inventory

You can filter the inventory on any attribute (description, device_type...). Example for `device_type` attribute. On the below example, we will filter only the `IOS_XR` device type.

```python
service.inventory.filter("device_type", "IOS_XR")
```

First the attribute name is looked for in the device parameters, then in the internal attributes, then in the metadata

[Documentation](https://radkit.cisco.com/docs/client_api/client_api.html#radkit_client.sync.DeviceDict.filter)
