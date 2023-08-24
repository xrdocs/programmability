---
published: true
date: '2023-08-24 12:25 -0400'
title: CLI to sensor-path
position: hidden
author: Rahul Sharma
---
# Background

The YANG model represents a Data Definition Language containing nodes, objects, and leaves capable of configuration, management, and streaming from a router.

However, the methodology for utilizing these YANG models to address use-cases, such as replicating the output of the 'show ip interface brief' CLI command, might not be common knowledge.

This article elucidates the process of identifying an appropriate YANG model along with its corresponding node or leaf, enabling the retrieval of the desired output. The focus lies in identifying an alternative YANG model for the 'show ip interface brief' CLI command.

## YANG Model in Brief

The YANG model takes the form of a hierarchical tree-like data structure, comprising of a root, intermediate nodes, and leaf nodes. Visualize these nodes as data entities instrumental in router management and configuration.

## Understanding Sensor-Path

A sensor-path, concerning a specific node, delineates the path within a YANG tree. This path originates at the root, traverses intermediate nodes, and concludes at the target node.

This sensor-path holds significance during the management of a particular node.

The sensor-path adheres to the subsequent format:

```
<YANG Model name>:<Root of YANG Model/Container>/<intermediate-node1>/<intermediate-node2>..../<leaf-node/leaf-list>
```

## Locating the Sensor-Path for a given CLI

The process of identifying a sensor-path can be broken down into the following stages.

Step 1: Utilize 'yang-describe'

The 'yang-describe' is a command-line tool compatible with XR 6.x to XR 7.10. Its syntax is as follows:

```
yang-describe operational <CLI>
```

In this scenario, the subsequent command can be used:
```
yang-describe operational show ip interface brief
```

It's important to note that this utility may not cater to all CLI commands, rendering output null for specific cases.

If a sensor-path is generated, proceed to step 3. If not, proceed to step 2.

Step2: Utilize 'Cisco Feature Navigator'

We have another tool known as 'Cisco Feature Navigator' that we can use to traverse YANG models on a specific platform for a given XR release without actually having a physical device. 

You can access this tool using this link [here](https://cfnng.cisco.com/).

Once you click on this link, click on 'Continue as Guest'. This will land you on the homepage.

Click on 'YANG Data Models', then in the 'Product' and 'Release' drop-down lists, select any product and XR release you are using and hit 'Submit'. Here, we will use NCS5500 with XR 7.9.2.

You will see three lists of 'Cisco XR-Native models (XR-N)', 'Cisco XR-Unified models (XR-UM)' and 'OpenConfig Data Models (OC)'

Want to read about these types of YANG Models, click here.

Since we are looking sensor-path for CLI 'show ip interface brief', the component here is 'interface'. So we'll type 'interface' in the 'Search for Data Model Name' search tab.

We'll get list of 'interface' based YANG-models, we can choose either one of them, Here, I am choosing unified model 'Cisco-IOS-XR-um-interface-cfg'.

In 'Search for Node' tab, we'll again type 'interface' and we'll get all the node having keyword 'interface'. Since I am trying to get list of all interface, I'll click on the leaf-list 'interface' as shown in the image below.

On the right hand side, I can see 'Xpath', which is as good as a sensor-path. 

And finally we find the sensor-path we needed.

We'll go to next step to verify if this is a correct sensor-path.


Step3 Utilize 'run mdt_exec'

'run mdt_exec' is an XR CLI tool that gives as an output, the data corresponding to a sensor-path.
 
In order to check if sensor-path is correct or not, we will use this tool and analyse the output. If the output is as desired, then the sensor-path is correct and vice-versa.

Use the following syntax to check if your sensor-path is apt or not:

```
run mdt_exec -s "<sensor-path>"
```

Here, this syantax would resolve to:

```
run mdt_exec -s "Cisco-IOS-XR-um-interface-cfg:interfaces/interface"
```

Note: Be mindful of leading '/' before the YANG model name. Make sure you remove it before executing this command. 















