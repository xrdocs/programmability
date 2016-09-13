---
published: false
date: '2016-09-13 19:15 +0300'
title: Getting started with YDK on IOS-XR
author: Mike Korshunov
excerpt: Few examples on how to use YDK with IOS-XR
tags:
  - iosxr
  - YANG
---

{% include toc icon="table" title="Getting started with YDK on IOS-XR" %}



YANG is a data modeling language used to model configuration and state data manipulated by the NETCONF protocol.

YDK or YANG Development Kit is a Software Development Kit that provides API's that are modeled in YANG. The main goal of YDK is to reduce the learning curve by expressing the model semantics in API and abstracting protocol/encoding details. YDK provides a simple CRUD (Create/Read/Update/Delete) API that allows the developer to perform these operations on entities on a server that supports them



## Preinstallation

We will use old setup, which consist of devbox (Ubuntu instance) and rtr (IOS-XRv).
{: .notice--info}  
![setup](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/mkorshun/netmiko_setup.png){: .align-center}
{: .notice}


```
git clone https://github.com/ios-xr/vagrant-xrdocs.git
cd vagrant-xrdocs/ansible-tutorials/app_hosting
```

Before issuing the command _vagrant up_ you need to decide, which version of devbox you want to use (Check next section).


### Actions on devbox

You can install **ydk-py** (base requirement to run sample apps) 2 ways: use a prepared box from the atlas or install the package manually on Ubuntu.

To use ready image replace value in Vagrantfile for devbox.vm.box from "ubuntu/trusty64" to **"ciscoxr/ydk-py-ubuntu"**

Manual installation process described in [ydk-py readme file](https://github.com/CiscoDevNet/ydk-py.git) 



### Actions on rtr


Step 1. Verify that the Cisco IOS XR software installed on your device supports both NETCONF and YANG.

Step 2. Activate crypto keys by opening a shell and entering the following command: ```crypto key generate dsa```

Step 3. Configure NETCONF over SSH:

```
ssh server v2
ssh server netconf 
ssh server netconf port 830
netconf-yang agent ssh
```

Intro about YDK - http://ydk.cisco.com/py/docs/getting_started.html


## Running examples

Huge variety of sample apps (>200) already published on Github.Apps separated in two huge groups: Codec and CRUD. To start clone a repo:

```
git clone https://github.com/CiscoDevNet/ydk-py-samples.git
cd ydk-py-samples
```

First one and simplest app to display system uptime _hello-ydk.py_. Before running a sample IP address should be changed. Open file in your favorite editor.  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    # piece from hello-ydk.py
    # create NETCONF session
    provider = NetconfServiceProvider(address=<mark>"10.1.1.20"</mark>,
                                      port=830,
                                      username=<mark>"vagrant"</mark>,
                                      password=<mark>"vagrant"</mark>,
                                      protocol="ssh")

</code>
</pre>
</div>


Run the app: 

```
vagrant@ydk-py:ydk-py-samples$ python hello-ydk.py
System uptime is 15 days, 22:11:13
```
Here we are, the first simple app is working. 

If you get an error: "can't connect to socket", ensure that you don't miss section with rtr configuration.
{: .notice--warning}

The structure of project provided below. When you get to the files, you will notice name like  _nc-delete-config-policy-repository-20-ydk.py_ 

```
<nc> - stands for protocol/service (netconf in our case)
<delete-config-policy-repository> - Yang model
<20> - level of complexity (started from 10 and incrementing)
```

```
samples/
└── basic
    ├── codec
    │   └── ydk
    │       └── models
    │           ├── cdp
    │           ├── clns
    │           ├── crypto
    │           ├── ethernet
    │           ├── ifmgr
    │           ├── infra
    │           ├── ip
    │           ├── ipv4
    │           ├── man
    │           ├── openconfig
    │           └── policy
    └── crud
        └── ydk
            └── models
                ├── bgp
                ├── cdp
                ├── clns
                ├── crypto
                ├── ethernet
                ├── ifmgr
                ├── infra
                ├── ip
                ├── ipv4
                ├── lib
                ├── linux
                ├── man
                ├── openconfig
                ├── policy
                └── shellutil

```


Usually, we run script, providing URL with credentials.

```
cd ~/ydk-py-samples/samples/basic/crud/ydk/models/shellutil
python nc-read-oper-shellutil-filesystem-20-ydk.py ssh://vagrant:vagrant@10.1.1.20:830
Node: RP/0/RP0/CPU0
File Systems:

    Size(b)     Free(b)        Type  Flags  Prefixes
           0           0     network     rw  tftp:
   480907264   478601216       flash     rw  /misc/config
  1029754880  1027883008  flash-disk     rw  disk0:
           0           0     network     rw  ftp:
  1015308288  1007226880    harddisk     rw  harddisk:
```

Feel free to try any other models and you can build and verify whole CRUD workflow with it!
{: .notice--success}  


## Few more links for further reading

[NETCONF / Yang on Cisco Devnet](https://developer.cisco.com/site/nso/downloads/yang/) 

[YDK-py](https://github.com/CiscoDevNet/ydk-py)

[YDK documentation](http://ydk.cisco.com/py/docs/getting_started.html)

[Yang central](http://www.yang-central.org/twiki/bin/view/Main/WebHome) 

[Original RFC](https://tools.ietf.org/html/rfc6020)

[Generate API bindings to Yang data models](https://github.com/CiscoDevNet/ydk-gen)
