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
# Background

Let’s first start with a little bit of background with why we need gRPC and why it is better than other available options.

# What is an RPC?

An RPC (Remote Procedure Call) is a mechanism in which a system 'A' (client) runs a program (or a process) on another system 'B' (server) and receives the output of this process as a response. It works on a client-server model and uses TCP protocols and JSON/XML for data serialization to transmit request/response data

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

**1. Capabilities function**

**2. Get function**

Path is provided as a list in the following format:
 
path=['yang-module:container/container[key=value]',
'yang-module:container/container[key=value]', ..]
 

Available path formats:
          - yang-module:container/container[key=value]
          - /yang-module:container/container[key=value]
          - /yang-module:/container/container[key=value]
          - /container/container[key=value]
        
The datatype argument may have the following values per gNMI specification:
          - all
          - config
          - state
          - operational

The encoding argument may have the following values per gNMI specification:
          - json
          - bytes
          - proto
          - ascii
          - json_ietf
         


**3. Set function**

There are 3 types of operations defined in the ‘SetRequest’ message:

**1.Update:** To create new configurations(data-node) as per the given path-value combination. 

**2.Delete:** To delete the configuration(data-node) in the on the router(data-tree) provided the path is correct and data-node already exists in the tree. Otherwise, the entire transaction is rolled back to its initial state. 

**3.Replace:** To replace the current configuration with the new supplied path-value combination.

For both Update and Replace operations, If the path specified does not exist, the target MUST create the data tree element and populate it with the data in the Update message, provided the path is valid according to the data tree schema. If invalid values are specified, the target MUST cease processing updates within the SetRequest method, return the data tree to the state prior to any changes, and return a SetResponse status indicating the error encountered.

Update and Delete operations are performed on the specific data-node and remaining configurations stays as it is. Whereas Replace operation replaces all the current configurations with new configurations. If a path-value combination for an already existing data-node is not given with this operation, then that data-node will get deleted if it doesn’t have a default values. If it has a default value, the values will be set to default.

**4. Capabilities function**
```python
from pygnmi.client import gNMIclient
import json

if __name__ == '__main__':

	with gNMIclient(target=("10.30.111.171",57777),username="cisco",password="cisco123!",insecure=True) as gc:
		
		capability_result = gc.capabilities()
	
	print(json.dumps(capability_result, indent=4))
```

