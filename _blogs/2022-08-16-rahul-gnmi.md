---
published: false
date: '2022-08-16 13:08 -0500'
title: Rahul-gnmi
author: Rahul Sharma
tags:
  - iosxr
  - cisco
position: hidden
---

---
published: false
date: '2022-08-16 13:08 -0500'
title: OpenConfig - gNMI
author: Rahul Sharma
tags:
  - iosxr
  - cisco
position: hidden
---
# Background

Let’s first start with a little bit of background with why we need gRPC and why it is better than other available options.

# What is an RPC?

An RPC (Remote Procedure Call) is a mechanism in which a system 'A' (client) runs a program (or a process) on another system 'B' (server) and receives the output of this process as a response. It works on a client-server model and uses TCP protocols and JSON/XML as encodings to transmit request/response data.

# What the problem with traditional RPC?

The problem starts with JSON/XML data formats. These text-based formats aren't the best when it comes to data compression. These formats transfer additional data (basically semantics) along with the actual data and this leads to a huge amount of data being transferred between client and server.

This problem is addressed in gRPC, and therefore gRPC is preferred over RPC.

# What is gRPC?

gRPC is an open-source protocol lead by Google (and hence 'g' in 'gRPC'). Like traditional RPC, it also uses an RPC but combines it with a 'Protocol Buffer'. A Protocol buffers can describe the structure of data and the code can be generated from that description for generating or parsing a stream of bytes that represents the structured data. The binary data format slashes the actual data to around 10x times. This compressed data is transferred over HTTP V2 that can also be combined with TLS to make it secure.

Since we are discussing HTTP, you might be wondering why not REST instead of traditional RPC.

Answer is that even REST uses JSON data format and as discussed above, this text-based format doesn't compress well. Also, REST uses HTTP v1.1 which works on a request/response model where if a server receives requests from multiples clients simultaneously, each request is handled one at a time. Whereas gRPC uses HTTP v2 that comes along with multiplexing capabilities along with request/response model and can handle multiple requests simultaneously.

So far, we have read why is gRPC the best option we have. But how do we make use of gRPC? One way is to create an RPC on our own and then convert it into corresponding binary data format and transfer it to/from the server. Does this method seem interesting?? because creating is an RPC is 'very easy'. Another method is to have an interface that creates RPCs for us and handle their conversion into corresponding data bytes at the client as well as at server.

And that's when gNMI comes into the role.

'gNMI' or 'gRPC Network Management Interface' is an interface that uses gRPC to communicate with devices and manages RPCs and data conversions that take place during the communication (JSON to binary and vice-versa). It provides us following 4 functions that use gRPC at their base:

**1. Capabilities:** To retrieve all the YANG modules present on the device. We use these modules to use upcoming functions.

**2. Get:**  To retrieve a operational data corresponding to the YANG module.

**3. Set:**  To make configurational changes on the device using a YANG module. You should know the data structure of this YANG module to use this functionality.

**4. Subscribe:** To stream operational data corresponding to a particular YANG module at a given cadence. Ideally this cadence should not be less than 30 seconds. It might vary depending on the situation.

After reading the stuff above, you now have an overview of why gRPC has an edge over traditional RPC, why do we need gNMI and what are its functionalities. But as said, we learn more when we perform what we read. And now it's time to leverage what we have learned/read so far and get our hands dirty.

Since we are using gRPCs that uses HTTP2.0, we can make our connection secure using TLS. For now, let’s start without TLS. Also, we have 2 opensource gNMI clients namely '**pygnmi**' and '**gnmic**'. We will start with 'pygnmi' for now.

# 'pygnmi' without TLS:

We will use a python library called 'pygnmi' as one of the gNMI clients and as a server, we would have a router with Cisco IOS-XR version 7.4.2

Step1:  we need to configure grpc server on the router.

	Router#confifure terminal
	Router(config)#grpc
	Router(config-grpc)#port 57777

Step2: Install pygnmi on your system

	pip install pygnmi

Step3: Use following scripts as an example to leverage all the gNMI functionalities

## 1. Capabilities function

```python
from pygnmi.client import gNMIclient
import json

if __name__ == '__main__':

	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:
		
		capability_result = gc.capabilities()
	
	print(json.dumps(capability_result, indent=4))
```

**Output**
This is an omitted output.
```json
{
    "supported_models": [
        {
            "name": "Cisco-IOS-XR-man-netconf-cfg",
            "organization": "Cisco Systems, Inc.",
            "version": "2019-12-12"
        },
        {
            "name": "Cisco-IOS-XR-dot1x-oper",
            "organization": "Cisco Systems, Inc.",
            "version": "2021-03-31"
        },
        {
            "name": "Cisco-IOS-XR-dot1x-oper-sub1",
            "organization": "Cisco Systems, Inc.",
            "version": "2021-03-31"
        }
    ],
    "supported_encodings": [
        "json_ietf",
        "ascii",
        "proto"
    ],
    "gnmi_version": "0.7.0"
}
```
## 2. Get function
```python
from pygnmi.client import gNMIclient
import json

if __name__ == '__main__':

	path = ['openconfig-interfaces:interfaces/interface[name=Loopback0]','Cisco-IOS-XR-ipv4-bgp-oper:bgp/bpm-instances-table/bpm-instances']
	
	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:	
		get_result = gc.get(path=path,encoding='json_ietf',datatype='all',) 

	print(json.dumps(get_result, indent=4))	
```
 
Available path formats:
	
 - /yang-module:container/container[key=value]
 - /yang-module:container/container[key=value]
 - /yang-module:/container/container[key=value]
 - /container/container[key=value]
        
The datatype argument may have the following values per gNMI specification:

 - all
 - config
 - state
 - operational

The encoding argument may have the following values per gNMI specification:
 - proto
 - ascii
 - json_ietf
 
 **Output**
 ```json
 {
    "notification": [
        {
            "timestamp": 1660773182082429567,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "interfaces/interface[name=Loopback0]/config",
                    "val": {
                        "name": "Loopback0",
                        "type": "iana-if-type:softwareLoopback"
                    }
                },
                {
                    "path": "interfaces/interface[name=Loopback0]/state",
                    "val": {
                        "name": "Loopback0",
                        "enabled": true,
                        "type": "iana-if-type:softwareLoopback",
                        "admin-status": "UP",
                        "oper-status": "UP",
                        "mtu": 1500,
                        "counters": {
                            "carrier-transitions": "1"
                        },
                        "last-change": "1658346470937377952",
                        "loopback-mode": false,
                        "ifindex": 9,
                        "logical": true
                    }
                },
                {
                    "path": "interfaces/interface[name=Loopback0]/subinterfaces",
                    "val": {
                        "subinterface": [
                            {
                                "index": 0,
                                "openconfig-if-ip:ipv4": {
                                    "addresses": {
                                        "address": [
                                            {
                                                "ip": "1.1.1.1",
                                                "config": {
                                                    "ip": "1.1.1.1",
                                                    "prefix-length": 32
                                                },
                                                "state": {
                                                    "ip": "1.1.1.1",
                                                    "prefix-length": 32,
                                                    "origin": "STATIC"
                                                }
                                            }
                                        ]
                                    }
                                }
                            }
                        ]
                    }
                },
                {
                    "path": "bgp/bpm-instances-table/bpm-instances",
                    "val": {
                        "instance": [
                            {
                                "instance-identifier": 0,
                                "placed-group-id": 1,
                                "instance-name-str": "default",
                                "as-number": 1,
                                "number-of-vrfs": 2,
                                "af-array": [
                                    {
                                        "entry": true
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": true
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": true
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    },
                                    {
                                        "entry": false
                                    }
                                ],
                                "read-only-enabled": false,
                                "install-diversion-enabled": false,
                                "srgb-start-configured": 0,
                                "srgb-end-configured": 0
                            }
                        ]
                    }
                }
            ]
        }
    ]
}
 ```      
## 3. Set function

There are 3 types of operations defined in the ‘SetRequest’ message:

**1.Update:** To create new configurations(data-node) as per the given path-value combination. 

**2.Delete:** To delete the configuration(data-node) in the on the router(data-tree) provided the path is correct and data-node already exists in the tree. Otherwise, the entire transaction is rolled back to its initial state. 

**3.Replace:** To replace the current configuration with the new supplied path-value combination.

For both Update and Replace operations, If the path specified does not exist, the target MUST create the data tree element and populate it with the data in the Update message, provided the path is valid according to the data tree schema. If invalid values are specified, the target MUST cease processing updates within the SetRequest method, return the data tree to the state prior to any changes, and return a SetResponse status indicating the error encountered.

Update and Delete operations are performed on the specific data-node and remaining configurations stays as it is. Whereas Replace operation replaces all the current configurations with new configurations. If a path-value combination for an already existing data-node is not given with this operation, then that data-node will get deleted if it doesn’t have a default values. If it has a default value, the values will be set to default.

The following code snippet creates a new interface and then deletes it

```python
from pygnmi.client import gNMIclient
import json

if __name__ == '__main__':
	
	## update
	
	cisco_update = [
				("openconfig-interfaces:interfaces/interface[name=Loopback3]",
					{
						"config":
						{
							"name":"Loopback3",
							"enabled": False,
							"type": "iana-if-type:softwareLoopback",     ## mandatory field as per YANG module
							"description":"testing pygnmi to create an interface"
						}

					}
				)

			] 
	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:
		update_result = gc.set(update=cisco_update,encoding='json_ietf')
	
	print(json.dumps(update_result, indent=4))
	
	## get
	
	path = ['openconfig-interfaces:interfaces/interface[name=Loopback3]']

	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:	
		get_result = gc.get(path=path,encoding='json_ietf',datatype='all',) 

	print(json.dumps(get_result, indent=4))	

	## delete	

	cisco_delete = ['openconfig-interfaces:interfaces/interface[name=Loopback3]']
	
	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:
		delete_result = gc.set(delete=cisco_delete,encoding='json_ietf')
	
	print(json.dumps(delete_result, indent=4))	
	
```
	
 **Output**
 ```json
 {
    "timestamp": 1660768044642964688,
    "prefix": null,
    "response": [
        {
            "path": "interfaces/interface[name=Loopback3]",
            "op": "UPDATE"
        }
    ]
}
{
    "notification": [
        {
            "timestamp": 1660768051355819859,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "interfaces/interface[name=Loopback3]",
                    "val": {
                        "config": {
                            "name": "Loopback3",
                            "type": "iana-if-type:softwareLoopback",
                            "enabled": false,
                            "description": "testing pygnmi to create an interface"
                        },
                        "state": {
                            "name": "Loopback3",
                            "enabled": false,
                            "type": "iana-if-type:softwareLoopback",
                            "admin-status": "DOWN",
                            "oper-status": "DOWN",
                            "mtu": 1500,
                            "counters": {
                                "carrier-transitions": "2"
                            },
                            "last-change": "1660768025944368364",
                            "loopback-mode": false,
                            "description": "testing pygnmi to create an interface",
                            "ifindex": 124,
                            "logical": true
                        }
                    }
                }
            ]
        }
    ]
}
{
    "timestamp": 1660768052452119615,
    "prefix": null,
    "response": [
        {
            "path": "interfaces/interface[name=Loopback3]",
            "op": "DELETE"
        }
    ]
}
 ```

## 4. Subscribe function

When a client wants a router to send a particular data (operational) at a given time interval, it sends a subscription list which contains a list of sensor paths and time interval. 

```python 

from pygnmi.client import gNMIclient,telemetryParser
import json

if __name__ == '__main__':
	
	subscribe_request = {

		'subscription':[
			{
				'path':'openconfig-interfaces:interfaces/interface[name=Loopback0]',
				'mode':'sample',
				'sample_interval':10000000000
			},
			{
				'path':'Cisco-IOS-XR-ipv4-bgp-oper:bgp/bpm-instances-table/bpm-instances',
				'mode':'sample',
				'sample_interval':10000000000 ## in nanoseconds
			}
		],
		'mode':'stream',
		'encoding':'json_ietf'

	}

	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:
		telemetry_stream = gc.subscribe(subscribe=subscribe_request)

		for data in telemetry_stream:
			print(json.dumps((telemetryParser(data)), indent=4))
```
**Output**

```json
{
    "update": {
        "update": [
            {
                "path": "interfaces/interface[name=Loopback0]",
                "val": {
                    "state": {
                        "name": "Loopback0",
                        "type": "iana-if-type:softwareLoopback",
                        "mtu": 1500,
                        "loopback-mode": false,
                        "enabled": true,
                        "ifindex": 9,
                        "admin-status": "UP",
                        "oper-status": "UP",
                        "last-change": "1658346470937377952",
                        "logical": true,
                        "counters": {
                            "carrier-transitions": "1"
                        }
                    }
                }
            }
        ],
        "timestamp": 1660773333831000000,
        "prefix": ""
    }
}
{
    "update": {
        "update": [
            {
                "path": "interfaces/interface[name=Loopback0]",
                "val": {
                    "subinterfaces": {
                        "subinterface": {
                            "index": 0,
                            "openconfig-if-ip:ipv4": {
                                "addresses": {
                                    "address": {
                                        "ip": "1.1.1.1",
                                        "state": {
                                            "ip": "1.1.1.1",
                                            "prefix-length": 32,
                                            "origin": "STATIC"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        ],
        "timestamp": 1660773333843000000,
        "prefix": ""
    }
}
{
    "update": {
        "update": [
            {
                "path": "bgp/bpm-instances-table/bpm-instances",
                "val": {
                    "instance": [
                        {
                            "instance-identifier": 0,
                            "placed-group-id": 1,
                            "instance-name-str": "default",
                            "as-number": 1,
                            "number-of-vrfs": 2,
                            "af-array": [
                                {
                                    "entry": true
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": true
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": true
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                },
                                {
                                    "entry": false
                                }
                            ],
                            "read-only-enabled": false,
                            "install-diversion-enabled": false,
                            "srgb-start-configured": 0,
                            "srgb-end-configured": 0
                        }
                    ]
                }
            }
        ],
        "timestamp": 1660773333887000000,
        "prefix": ""
    }
}
{
    "sync_response": true
}

```