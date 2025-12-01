---
published: true
date: '2025-12-01 11:39 +0100'
title: Radkit - Tips and tricks
author: Antoine Orsoni
tags:
  - cisco
  - Radkit
position: top
---
{% include toc icon="table" title="Table of Contents" %}

# Introduction

RADKit is a Software Development Kit (SDK): a set of ready-to-use tools and Python modules allowing efficient and scalable interactions with local or remote equipment. RADKit is available at no additional cost with your existing Support Contracts.

This article combines a list of useful tips and tricks I discovered. I will update it with every new finding. Do you know a cool trick? Feel free to share in the comments.

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

You can chain multiple filters.

```python
service.inventory.filter('device_type', 'IOS_XR').filter('description', 'NCS 540')
```

It also accepts regex to filter, for example on the device name.

[Documentation](https://radkit.cisco.com/docs/client_api/client_api.html#radkit_client.sync.DeviceDict.filter)

## Sending a command on the (filtered) inventory

Once filtered (if needed), you can send a command on the inventory. Radkit client will take care of the parallelization for you. `wait` method allows to wait for all subsequent RPC to be completed.

```python
iosxr = service.inventory.filter("device_type", "IOS_XR")
software_version = iosxr.exec("show version").wait()
```

When printing `software_version`, it will look like below. Note the result is `PARTIAL_SUCCESS` meaning we had an error collecting the command for some devices.

```
[PARTIAL_SUCCESS] <ExecResponse_ByDevice_ToSingle {3 entries}>
status    index    service_id      device                   command       sudo    data                                                      
                                      
--------  -------  --------------  -----------------------  ------------  ------  ----------------------------------------------------------
------------------------
FAILURE   0        service1  device1  show version  False   (error: Device action failed: Permission error while prepa
ring connection)        
SUCCESS   1        service1  device2      show version  False   RP/0/RP0/CPU0:device1#show version\nMon Dec  1
 13:19:05.317 CET\nCi...
SUCCESS   2        service1  device3       show version  False   RP/0/RP0/CPU0:device2#show version\nMon Dec  1 
13:19:05.308 CET\nCis...
```

## Extracting and filtering outputs

You can filter outputs with each RPC's status. For example, here we will only print the `success` entries.

```python
success_replies = sotware_version.by_status['SUCCESS']
```

Sample output
```python
[SUCCESS] <ExecResponse_ByDevice_ToSingle {2 entries}>
status    index    service_id      device                   command       sudo    data                                                      
                                      
--------  -------  --------------  -----------------------  ------------  ------  ----------------------------------------------------------
------------------------
SUCCESS   1        service1  device1      show version  False   RP/0/RP0/CPU0:device1#show version\nMon Dec  1
 13:19:05.317 CET\nCi...
SUCCESS   2        service1  device2      show version  False   RP/0/RP0/CPU0:device2#show version\nMon Dec  1 
13:19:05.308 CET\nCis...
```

You can print the associated device names like below.

```python
list(success.by_device.keys())
```

Sample output
```python
['device1', 'device2']
```

## Iterating through results

Now that we have collected the command, we might want to iterate through results to do something (here, we will just print the output). Yet, in Radkit 1.9.0, there is no elegant want to do it. The easiest way I found is this one:

```python
for name, result in software_version.items():
    print(f"{name}: {result.data}")
```

## Combining with Genie parsers

From Radkit 1.9.0, you can combine with Genie parsers.

```python
import radkit_genie as rkg

software_version_parsed = rkg.parse(software_version)
for result in software_version_parsed.values():
    print(f'{result.device.name}: {result.data['version']['version']}')
```


