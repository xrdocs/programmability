---
published: true
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

# What's the problem with traditional RPC?

The problem starts with JSON/XML data formats. These text-based formats aren't the best when it comes to data compression. These formats transfer additional data (basically semantics) along with the actual data and this leads to a huge amount of data being transferred between client and server.

Other solutions also available such as Apache RPC, but for this article we will focus on gRPC.

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

<details>
  <summary><b>Output</b></summary>

 This is an omitted out1.  

```yaml
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

</details>

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

 <details>
  <summary><b>Output</b></summary>

This is an omitted output.  
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
</details>

 
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
 
 
> **_NOTE:_** When using ‘get’ or ‘set’ functions, some parameters are mandatory to pass as a parameter. To find mandatory parameters for a YANG module, we need to go through that module.

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
<details>
  <summary><b>Output</b></summary>
 
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
  
</details>


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
<details>
  <summary><b>Output</b></summary>
  
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
</details>

# 'pygnmi' with TLS:

In order to have TLS encryption to secure the data, we need an ems.pem certificate. This certificate is available at '/misc/config/grpc' directory on the router and can be copied into the same folder as the scripts above. Once it is copied, you can replace the line the line:
	
	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:
	
with the following lines:
	
	cert = "Documents/certs/ems.pem"
	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",path_cert=cert,override="ems.cisco.com") as gc:
	
For example, to use the capabilities function of gNMI, our new code would be:
```python

from pygnmi.client import gNMIclient
import json

if __name__ == '__main__':
	
	cert = "/Users/rahusha7/Documents/rahusha7-github/pygnmi/functionalities/certs/ems.pem"
	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",path_cert=cert,override="ems.cisco.com") as gc:
		capability_result = gc.capabilities()
	print(json.dumps(capability_result, indent=4))
```

Similarly, we can use 'Get', 'Set' and 'Subscribe' functions with TLS.

# 'gNMIc' without TLS:

As oppossed to pygnmi, this is a CLI tool that enables us leverage gNMI functionalities.

## 1. Capabilities function
	
	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure capabilities

<details>
  <summary><b>Output</b></summary>
  
 
  ```

gNMI version: 0.7.0
supported models:
  - Cisco-IOS-XR-man-netconf-cfg, Cisco Systems, Inc., 2019-12-12
  - Cisco-IOS-XR-dot1x-oper, Cisco Systems, Inc., 2021-03-31
  - Cisco-IOS-XR-dot1x-oper-sub1, Cisco Systems, Inc., 2021-03-31
  - Cisco-IOS-XR-ethernet-link-oam-cfg, Cisco Systems, Inc., 2020-04-15
  - Cisco-IOS-XR-um-dci-fabric-interconnect-cfg, Cisco Systems, Inc., 2020-12-14
  - Cisco-IOS-XR-shellutil-delete-act, Cisco Systems, Inc., 2019-10-01
  - Cisco-IOS-XR-um-config-display-cfg, Cisco Systems, Inc., 2021-04-14
  - Cisco-IOS-XR-um-l2-ethernet-cfg, Cisco Systems, Inc., 2021-03-30
  - openconfig-igmp-types, OpenConfig working group, 0.1.1
  - openconfig-ospfv2, OpenConfig working group, 0.2.3
  - openconfig-ospfv2-global, OpenConfig working group, 0.2.3
  - openconfig-ospfv2-lsdb, OpenConfig working group, 0.2.3
  - openconfig-ospfv2-area-interface, OpenConfig working group, 0.2.3
  - openconfig-ospfv2-common, OpenConfig working group, 0.2.3
  - openconfig-ospfv2-area, OpenConfig working group, 0.2.3
supported encodings:
  - JSON_IETF
  - ASCII
  - PROTO
  ```
</details>

## 2. Get function

To retrieve the content of a container, here 'interfaces', we can use the following command:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure get --path 'openconfig-interfaces:/interfaces' -e json_ietf

> **_NOTE:_** Due to several naming conventions, first container in the path doesn't have a forward slash ('/') before it. So Make sure there is a forward slash ('/') right after the colon (':') follwed by YANG module name.
 		
<details>
  <summary><b>Output</b></summary>
  
 
  ```json
[
  {
    "source": "10.30.111.171:57777",
    "timestamp": 1661276820039040328,
    "time": "2022-08-23T12:47:00.039040328-05:00",
    "updates": [
      {
        "Path": "openconfig-interfaces:interfaces",
        "values": {
          "interfaces": {
            "interface": [
              {
                "config": {
                  "name": "BVI1",
                  "type": "iana-if-type:propVirtual"
                },
                "name": "BVI1",
                "state": {
                  "admin-status": "UP",
                  "counters": {
                    "carrier-transitions": "1"
                  },
                  "enabled": true,
                  "ifindex": 8,
                  "last-change": "1661274102416905208",
                  "logical": true,
                  "loopback-mode": false,
                  "mtu": 1514,
                  "name": "BVI1",
                  "oper-status": "UP",
                  "type": "iana-if-type:propVirtual"
                },
                "subinterfaces": {
                  "subinterface": [
                    {
                      "index": 0,
                      "openconfig-if-ip:ipv4": {
                        "addresses": {
                          "address": [
                            {
                              "config": {
                                "ip": "10.10.8.1",
                                "prefix-length": 24
                              },
                              "ip": "10.10.8.1",
                              "state": {
                                "ip": "10.10.8.1",
                                "origin": "STATIC",
                                "prefix-length": 24
                              }
                            }
                          ]
                        },
                        "neighbors": {
                          "neighbor": [
                            {
                              "ip": "10.10.8.1",
                              "state": {
                                "ip": "10.10.8.1",
                                "link-layer-address": "00:8a:96:aa:70:da",
                                "origin": "OTHER"
                              }
                            }
                          ]
                        },
                        "state": {
                          "counters": {
                            "in-octets": "0",
                            "in-pkts": "0",
                            "out-octets": "0",
                            "out-pkts": "0"
                          }
                        }
                      }
                    }
                  ]
                }
              },
              {  
              "config": {
                  "name": "Loopback0",
                  "type": "iana-if-type:softwareLoopback"
                },
                "state": {
                  "admin-status": "UP",
                  "counters": {
                    "carrier-transitions": "1"
                  },
                  "enabled": true,
                  "ifindex": 10,
                  "last-change": "1661273971876349223",
                  "logical": true,
                  "loopback-mode": false,
                  "mtu": 1500,
                  "name": "Loopback0",
                  "oper-status": "UP",
                  "type": "iana-if-type:softwareLoopback"
                },
                "subinterfaces": {
                  "subinterface": [
                    {
                      "index": 0,
                      "openconfig-if-ip:ipv4": {
                        "addresses": {
                          "address": [
                            {
                              "config": {
                                "ip": "1.1.1.1",
                                "prefix-length": 32
                              },
                              "ip": "1.1.1.1",
                              "state": {
                                "ip": "1.1.1.1",
                                "origin": "STATIC",
                                "prefix-length": 32
                              }
                            }
                          ]
                        }
                      }
                    }
                  ]
                }
              }
            ]  
          }
        }
      }
    ]
  }
] 
  ```
</details>

To retrive a specific leaf of a container, we need use key-value pair with the following command:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure get --path 'openconfig-interfaces:/interfaces/interface[name=Loopback0]' -e json_ietf
	
<details>
  <summary><b>Output</b></summary>
  
 
  ```json
  [
  {
    "source": "10.30.111.171:57777",
    "timestamp": 1661277653053111016,
    "time": "2022-08-23T13:00:53.053111016-05:00",
    "updates": [
      {
        "Path": "openconfig-interfaces:interfaces/interface[name=Loopback0]",
        "values": {
          "interfaces/interface": {
            "config": {
              "name": "Loopback0",
              "type": "iana-if-type:softwareLoopback"
            },
            "state": {
              "admin-status": "UP",
              "counters": {
                "carrier-transitions": "1"
              },
              "enabled": true,
              "ifindex": 10,
              "last-change": "1661273971876349223",
              "logical": true,
              "loopback-mode": false,
              "mtu": 1500,
              "name": "Loopback0",
              "oper-status": "UP",
              "type": "iana-if-type:softwareLoopback"
            },
            "subinterfaces": {
              "subinterface": [
                {
                  "index": 0,
                  "openconfig-if-ip:ipv4": {
                    "addresses": {
                      "address": [
                        {
                          "config": {
                            "ip": "1.1.1.1",
                            "prefix-length": 32
                          },
                          "ip": "1.1.1.1",
                          "state": {
                            "ip": "1.1.1.1",
                            "origin": "STATIC",
                            "prefix-length": 32
                          }
                        }
                      ]
                    }
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
]

  ```
</details>

To retrieve more granular information about a leaf, we use the following command:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure get --path 'openconfig-interfaces:/interfaces/interface[name=Loopback3]/state' -e json_ietf
	
<details>
  <summary><b>Output</b></summary>
  
 
  ```json
	[
  {
    "source": "10.30.111.171:57777",
    "timestamp": 1661353613153429526,
    "time": "2022-08-24T10:06:53.153429526-05:00",
    "updates": [
      {
        "Path": "openconfig-interfaces:interfaces/interface[name=Loopback3]/state",
        "values": {
          "interfaces/interface/state": {
            "admin-status": "UP",
            "counters": {
              "carrier-transitions": "1"
            },
            "enabled": true,
            "ifindex": 125,
            "last-change": "1661291780049212608",
            "logical": true,
            "loopback-mode": false,
            "mtu": 1500,
            "name": "Loopback3",
            "oper-status": "UP",
            "type": "iana-if-type:softwareLoopback"
          }
        }
      }
    ]
  }
]
  ```
</details>




## 3. Set function

This function helps us make configurational changes on the router. Here, we will create a new leaf with desired configuration and then delete it using following two commands.

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure set --update-path 'openconfig-interfaces:/interfaces/interface[name=Loopback3]' --update-file input.json
	
<details>
  <summary><b>Output</b></summary>
  
 
  ```json
	{
  "source": "10.30.111.171:57777",
  "timestamp": 1661358941551196624,
  "time": "2022-08-24T11:35:41.551196624-05:00",
  "results": [
    {
      "operation": "UPDATE",
      "path": "openconfig-interfaces:interfaces/interface[name=Loopback3]"
    }
  ]
}
  ```
</details>

<details>
  <summary>In this command, we use input.json file to pass required parameters to create an interface. Here is a sample input.json file.</summary>
  
 
  ```json
	{
    "config": {
        "name": "Loopback4",
        "type": "iana-if-type:softwareLoopback",
        "enabled": false,
        "description": "testing pygnmi to create an interface"
    }
}    
  ```
</details>

To delete a leaf,  we can use the following command:
	
	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure set --delete 'openconfig-interfaces:/interfaces/interface[name=Loopback3]'

<details>
  <summary><b>Output</b></summary>
  
 
  ```json
	{
  "source": "10.30.111.171:57777",
  "timestamp": 1661359383278642479,
  "time": "2022-08-24T11:43:03.278642479-05:00",
  "results": [
    {
      "operation": "DELETE",
      "path": "openconfig-interfaces:interfaces/interface[name=Loopback3]"
    }
  ]
}
  ```
</details>


## 4. Subscribe function

We can have a streaming telemetry at a given cadence using this function using the following CLI:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure subscribe --path 'openconfig-interfaces:/interfaces/interface[name=Loopback0]' --mode stream --sample-interval 20s --stream-mode sample
	
<details>
  <summary><b>Output</b></summary>
  
 
  ```json
	{
  "source": "10.30.111.171:57777",
  "subscription-name": "default-1661359862",
  "timestamp": 1661359913237000000,
  "time": "2022-08-24T11:51:53.237-05:00",
  "prefix": "openconfig-interfaces:",
  "updates": [
    {
      "Path": "interfaces/interface[name=Loopback0]",
      "values": {
        "interfaces/interface": {
          "subinterfaces": {
            "subinterface": {
              "index": 0,
              "openconfig-if-ip:ipv4": {
                "addresses": {
                  "address": {
                    "ip": "1.1.1.1",
                    "state": {
                      "ip": "1.1.1.1",
                      "origin": "STATIC",
                      "prefix-length": 32
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  ]
}
{
  "source": "10.30.111.171:57777",
  "subscription-name": "default-1661359862",
  "timestamp": 1661359933239000000,
  "time": "2022-08-24T11:52:13.239-05:00",
  "prefix": "openconfig-interfaces:",
  "updates": [
    {
      "Path": "interfaces/interface[name=Loopback0]",
      "values": {
        "interfaces/interface": {
          "subinterfaces": {
            "subinterface": {
              "index": 0,
              "openconfig-if-ip:ipv4": {
                "addresses": {
                  "address": {
                    "ip": "1.1.1.1",
                    "state": {
                      "ip": "1.1.1.1",
                      "origin": "STATIC",
                      "prefix-length": 32
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  ]
}
  ```
</details>

You can confirm the cadence in the time stamp of each stream, here it's 20s.

# 'gNMIc' with TLS:

To enable TLS with gNMIc, we need to copy the 'ems.pem' certificate present in the '/misc/config/grpc' directory on the router and pass it's location as one of the parameters with CLI.

For example, to use TLS with 'Capabilities' functions, our new command would be:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --tls-cert 'Documents/certs/ems.pem' --skip-verify capabilities -e 'json_ietf'

Here, we are passing two additional parameters - '--tls-cert' and '--skip-verify'.

We can use these paramters with 'Get','Set' and 'Subscribe' functions and leverage TLS with them.
