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

{% include toc icon="table" title="ON THIS PAGE" %}

# Background

In today's era, where applications are built on microservices architecture, we need an API framework that allows these microservices to communicate effectively and allows developers to focus on the core code of their microservices rather than on code that allows them to talk to each other.

There are existing API frameworks with their downsides:

1. REST 
	a. It uses JSON, which is a text based encoding, which is much heavier as opposed to binary encoding.
    
	b. It uses HTTP/1.1 which uses request/response model, meaning that if a server gets requests from numerous clients at once, each request is dealt with separately. Also, it doesn't support TLS.

2. RPC 

	a. It also uses JSON/XML encoding which is heavier as it's a text-based.

An ideal API framework should possess following features:

1. Fast and light weight - effective encoding technique for structured data for example binary.
2. Is loosely coupled - independent of the underlying architecture.
3. Uses efficient and secured protocol - like HTTP/2 that sends data using TLS a in binary format.
4. Supports bidirectial communication - unlike SOAP and REST.

In order to check all these 4 boxes, Google came up with an API framework known as gRPC where 'g' stands for 'Google'.

# What is gRPC?

A Google-led open-source protocol that utilises an RPC, just like regular RPC, but combines it with a "Protocol Buffer." A protocol buffer can define the data structure, and from that description, code can be written to create or parse a stream of bytes that corresponds to the structured data. The real data is reduced by the binary data format by around 10 times. This compressed data is transmitted over HTTP/2, which may be secured using TLS.

Also, gRPC employs HTTP/2, which supports multiplexing, a request/response mechanism, and can handle many requests concurrently.

So far, we have read why gRPC is the best option we have. But how do we make use of gRPC? One way is to create an RPC on our own and then convert it into corresponding binary data format and transfer it to/from the server. Does this method seem interesting?? because creating is an RPC is 'very easy'. Another method is to have an interface that creates RPCs for us and handle their conversion into corresponding data bytes at the client as well as at server.

And that's when gNMI comes into the role.

The "gNMI" or "gRPC Network Management Interface" is an interface that uses Protocol Buffer to handle data translation (JSON to binary and vice versa) and provides RPCs to control network devices. It gives us the following four functions, all of which are based on gRPC:

**1. Capabilities:** To obtain every YANG module that is installed on the device. In order to use forthcoming functions, we require these modules.

**2. Get:**   To obtain operational data related to the YANG module.

**3. Set:** To use a YANG module to update the device's settings. To utilize this capability, you must be familiar with the YANG module's data structure.

**4. Subscribe:** To stream operational data corresponding to a particular YANG module at a given cadence. Ideally this cadence should not be less than 30 seconds. It might vary depending on the situation.

After reading the above mentioned material, you now have a general understanding of gRPC's advantages over regular RPC, the needs for gNMI, and its features. But as said, applying what we read helps us learn more. It's time to put what we have learned and read thus far to use and get our hands dirty.

Since we are utilizing gRPCs, which employs HTTP/2, we may utilize TLS to protect our connection. Let's begin without TLS for now. Additionally, we have two open-source gNMI clients, 'pygnmi' and 'gnmic'. For now, we'll start with 'pygnmi'.

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
	
This is an omitted output.

<script src="https://gist.github.com/rahusha7/ea2ded49a8d5327f96254a2478d8cc03.js"></script>

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

	<script src="https://gist.github.com/rahusha7/a41fdd960b696d4d4c58899b655f9929.js"></script>
	
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

**1. Update:** To create new configurations(data-node) as per the given path-value combination. 

**2. Delete:** To delete the configuration(data-node) in the on the router(data-tree) provided the path is correct and data-node already exists in the tree. Otherwise, the entire transaction is rolled back to its initial state. 

**3. Replace:** To replace the current configuration with the new supplied path-value combination.

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
 <script src="https://gist.github.com/rahusha7/cfc50c1ee4b7fc7577ee91b4937bbcef.js"></script>
  
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
  <script src="https://gist.github.com/rahusha7/36b599e8d1aa4a2b7e054fdce55c54b9.js"></script>
</details>

# 'pygnmi' with TLS:

In order to have TLS encryption to secure the data, we need an ems.pem certificate. This certificate is available at '/misc/config/grpc' directory on the router and can be copied into the same folder as the scripts above. Once it is copied, you can replace the line :
	
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
 <script src="https://gist.github.com/rahusha7/fa6285714399a4c76d46e11d0ff035da.js"></script> 
 
</details>

## 2. Get function

To retrieve the content of a container, here 'interfaces', we can use the following command:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure get --path 'openconfig-interfaces:/interfaces' -e json_ietf

> **_NOTE:_** Due to several naming conventions, first container in the path doesn't have a forward slash ('/') before it. So Make sure there is a forward slash ('/') right after the colon (':') follwed by YANG module name.
 		
<details>
  <summary><b>Output</b></summary>
  
 <script src="https://gist.github.com/rahusha7/2b22f837758282a70afe35a31a5e85b0.js"></script>
	
</details>

To retrieve a specific leaf of a container, we need use key-value pair with the following command:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure get --path 'openconfig-interfaces:/interfaces/interface[name=Loopback0]' -e json_ietf
	
<details>
  <summary><b>Output</b></summary>
  
<script src="https://gist.github.com/rahusha7/46cf78b838415667604e705db285bacf.js"></script>
</details>

To retrieve more granular information about a leaf, we use the following command:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure get --path 'openconfig-interfaces:/interfaces/interface[name=Loopback3]/state' -e json_ietf
	
<details>
  <summary><b>Output</b></summary>
<script src="https://gist.github.com/rahusha7/ba88c1f352efa79b9db1f24902c2c1d2.js"></script>
</details>


## 3. Set function

This function helps us make configurational changes on the router. Here, we will create a new leaf with desired configuration and then delete it using following two commands.

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure set --update-path 'openconfig-interfaces:/interfaces/interface[name=Loopback3]' --update-file input.json
	
<details>
  <summary><b>Output</b></summary>
  
<script src="https://gist.github.com/rahusha7/c51dadefb561889dca5d8e123cb0c311.js"></script>
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

<script src="https://gist.github.com/rahusha7/988a8adfd5085f605f2ba3af3599d9b3.js"></script>

</details>


## 4. Subscribe function

We can have a streaming telemetry at a given cadence using this function using the following CLI:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --insecure subscribe --path 'openconfig-interfaces:/interfaces/interface[name=Loopback0]' --mode stream --sample-interval 20s --stream-mode sample
	
<details>
  <summary><b>Output</b></summary>
  <script src="https://gist.github.com/rahusha7/5769cbac089107eec39028f1c0215e9f.js"></script>
</details>

You can confirm the cadence in the time stamp of each stream, here it's 20s.

# 'gNMIc' with TLS:

To enable TLS with gNMIc, we need to copy the 'ems.pem' certificate present in the '/misc/config/grpc' directory on the router and pass it's location as one of the parameters with CLI.

For example, to use TLS with 'Capabilities' functions, our new command would be:

	gnmic -a 10.30.111.171:57777 -u cisco -p cisco123! --tls-cert 'Documents/certs/ems.pem' --skip-verify capabilities -e 'json_ietf'

Here, we are passing two additional parameters - '--tls-cert' and '--skip-verify'.

We can use these paramters with 'Get','Set' and 'Subscribe' functions and leverage TLS with them.


# Conclusion

With these clients you can start to interact with device and grow your automation use cases.
