---
date: '2016-12-08'
title: "XR Data-model Overview"
author: Santiago Alvarez
tags:
  - programmability
published: true
# position: hidden
---

In a previous [blog post](https://xrdocs.github.io/programmability/blogs/2016-09-12-model-driven-programmability/), we described how the Cisco IOS XR programmability framework is based on data models.  But, what format do they have?  where are the models published?  how many are available?  how are they grouped?

If you are completely new to data models, all you need to understand at this point is that data models mostly specify the configuration and operational state that a device supports.  Models are defined as text files using the YANG modeling language ([RFC 7950](https://tools.ietf.org/html/rfc7950)) and they arrange data in a tree structure.  While data models are human readable, their strength comes from the fact that they have well-defined syntax and semantics that facilitate the development of software tools to process them.  They simplify network automation and orchestration.  In this tutorial, we will not dig into YANG specifics.  Instead, we will overview the organization and naming of XR native models.

XR devices ship with the YANG files that define the data models they support.  Using a management protocol (e.g. NETCONF, gRPC, RESTCONF), you can programmatically query a device for the list of models it supports and  retrieve the model files.  In addition, the XR models are made publicly available for download on GitHub.  In this tutorial, we will use that repository to explore the XR native models.

Let's create a directory to host the YANG files and clone the git repository:

```
~/$ mkdir -p ~/yang/modules/YangModels
~/$ cd ~/yang/modules/YangModels
YangModels/$ git clone git@github.com:YangModels/yang.git
Cloning into 'yang'...
remote: Counting objects: 5541, done.
remote: Compressing objects: 100% (52/52), done.
remote: Total 5541 (delta 7), reused 0 (delta 0), pack-reused 5489
Receiving objects: 100% (5541/5541), 11.82 MiB | 2.16 MiB/s, done.
Resolving deltas: 100% (3090/3090), done.
YangModels/$
```

If we take a closer look at the XR files, we see that they are grouped by software release:

```
YangModels/$ cd yang/vendor/cisco/xr/
xr/$ ls
530  531  532  533  600  601  611  612  check.sh  README.md
xr/$
```

The name of XR data models use the following notation:

```
Cisco-IOS-XR-<platform><technology><suffix>
```

XR models start with the prefix `Cisco-IOS-XR`, are followed by an optional platform substring (e.g. `ncs5500`, `asr9k`, etc), followed by a technology substring (e.g. `ipv4-bgp`) and finally ending with a suffix that indicates whether the model defines configuration data (`cfg`), operational data (`oper`) or an action (`act`).  Note that a YANG model can specify multiple types of data simultaneously.  However, Cisco IOS XR defines separate models for configuration, operational data and actions.

If we examine the directory for the XR 6.1.2 release, we see that there are 571 YANG files that define XR data models. The actual number of models is lower.  Each YANG file defines a module or submodule. Submodules are partial modules that contribute definitions to a module.  One or more modules and submodules define a model:

```
xr/$ cd 612
612/$ ls Cisco-IOS-XR-*.yang | wc -l
571
612/$
```

Based on the naming convention above, we identify 158 configuration models in this release:

```
612/$ ls Cisco-IOS-XR-*cfg.yang | wc -l
158
612/$
```

If we look for the BGP configuration model, we can see that it is platform independent (no platform substring), is defined in file `Cisco-IOS-XR-ipv4-bgp-cfg.yang` and has a corresponding name `Cisco-IOS-XR-ipv4-bgp-cfg`:

```
612/$ ls *bgp*cfg.yang
Cisco-IOS-XR-ipv4-bgp-cfg.yang
612/$
612/$ grep "module Cisco" Cisco-IOS-XR-ipv4-bgp-cfg.yang
module Cisco-IOS-XR-ipv4-bgp-cfg {
612/$
```

Following the naming convention described earlier, we find initially 151 files associated with operational models:

```
612/$ ls Cisco-IOS-XR-*oper.yang | wc -l
151
612/$
```

If we look for the BGP operational model, we can see that the BGP operational model is platform independent (no platform substring), is defined in file `Cisco-IOS-XR-ipv4-bgp-oper.yang` and has a corresponding name `Cisco-IOS-XR-ipv4-bgp-oper`:

```
612/$ ls *bgp*oper.yang
Cisco-IOS-XR-ipv4-bgp-oper.yang
612/$
612/$ grep "module Cisco" Cisco-IOS-XR-ipv4-bgp-oper.yang
module Cisco-IOS-XR-ipv4-bgp-oper {
612/$
```

At this point we have accounted for 309 of the initial list of 571 YANG files.  What about the other 262 files?  As mentioned above, data models are defined by one or more modules and submodules. Some operational models are defined using submodels.  Those files use the suffix `oper-sub` followed by a sequence number.  We find 236 files that define operational submodules:

```
612/$ ls Cisco-IOS-XR-*oper-sub[1-9].yang | wc -l
236
612/$
```

One of those submodules is actually associated with the BGP operational model (`Cisco-IOS-XR-ipv4-bgp-oper`) that we identified before:

```
612/$ ls Cisco-IOS-XR-*bgp*oper-sub[1-9].yang
Cisco-IOS-XR-ipv4-bgp-oper-sub1.yang
612/$
612/$ grep "submodule Cisco\|belongs-to" Cisco-IOS-XR-ipv4-bgp-oper-sub1.yang
submodule Cisco-IOS-XR-ipv4-bgp-oper-sub1 {
  belongs-to Cisco-IOS-XR-ipv4-bgp-oper {
612/$
```

Now we have accounted for 545 YANG files,  what about the remaining 26 files?  Out of those, 22 files define data types used in models.  The YANG language has some built-in data types (e.g. boolean, uint32, string, etc.) that can be used as primitives to define more elaborate types (e.g. interface name, BGP ASN, etc.)  Those files use the suffix `types`:

```
612/$ ls *types.yang | wc -l
22
612/$
```

One of these files (`Cisco-IOS-XR-ipv4-bgp-datatypes.yang`) actually corresponds to data types for BGP.  It defines some of the address families used by BGP models:

```
612/$ ls *bgp*types.yang
Cisco-IOS-XR-ipv4-bgp-datatypes.yang
612/$
```

What about the actions models we mentioned earlier?  They correspond to commands that can be executed on the device.  These actions are called remote procedure calls (RPCs) in YANG and are specified in files with suffix `act`.  The actions defined in this software release include commands to roll back configuation, test SNMP traps and generate custom Syslog messages:

```
612/$ ls Cisco-IOS-XR-*act.yang | wc -l
3
612/$
612/$ ls -1 Cisco-IOS-XR-*act.yang
Cisco-IOS-XR-cfgmgr-rollback-act.yang
Cisco-IOS-XR-snmp-test-trap-act.yang
Cisco-IOS-XR-syslog-act.yang
612/$
```

What about the other 40 YANG files that we have been conveniently ignoring in this discussion?  Those files do not define XR native data models.  We will discuss them in detail in future postings.

```
612/$ ls *.yang | grep -v Cisco-IOS-XR | wc -l
40
612/$
```

# In Summary
- XR native data models follow a specific naming convention (`Cisco-IOS-XR-<platform><technology><suffix>`)
- Each model provides a single type of definition (configuration data, operational state or action)
- The definition of a model may be involve multiple YANG files where each file specifies a YANG module or submodule
