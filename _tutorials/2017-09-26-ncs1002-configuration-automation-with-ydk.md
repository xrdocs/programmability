---
published: true
date: '2017-09-26 08:04 -0700'
title: NCS1002 Configuration Automation with YDK
author: Viktor Osipchuk
excerpt: NCS1002 Configuration Automation with YDK
tags:
  - iosxr
  - automation
  - YDK
  - programmability
  - NCS1002
  - NCS1k
---

This tutorial is the next one in our series of documents, devoted to automation of configuration of Cisco optical products. We will give here exact examples and explanations of how to use OpenConfig models and YDK infrastructure to configure a slice on [NCS1002 (terminal device)](http://www.cisco.com/c/en/us/products/collateral/optical-networking/network-convergence-system-1000-series/datasheet-c78-733699.html). You can refer to the [previous tutorial](https://xrdocs.github.io/programmability/tutorials/2017-08-22-ncs1002-configuration-automation-overview/) to refresh information about different modes of NCS1002 slices and corresponding OpenConfig models.

There are many ways to configure a device using OpenConfig models. This document describes how you can automate configuration with help of YDK. 
[YANG Development Kit (YDK)](http://ydk.io) is a powerful, open-source tool that generates APIs using YANG models. YDK supports different programming languages and has several built-in components that isolate you from the protocol, transport and encoding specifics. There is no need to code protocol specifics (e.g. [NETCONF](https://tools.ietf.org/html/rfc6241), [RESTCONF](https://tools.ietf.org/html/rfc8040)) or manipulate serialized data (e.g. [XML](https://www.w3.org/XML/) or [JSON](http://www.json.org/)) directly, as YDK has this integrated. With YDK you can focus your attention on the data structures and automation logic. YDK supports Python and C++ today with more languages coming soon (e.g., Go Lang, Ruby, etc.). 
YDK performs local validation of your code based on information embedded in the YANG model. The tool checks types, values, semantics, deviations, etc to make sure that some possible errors are found locally, before applying configuration on a device. 

## Slice configuration overview

In our introduction tutorial, we talked about five different slice modes (three modes for 100G client ports and two modes for 10G clients). 
Let’s have a look on a configuration procedure for the simplest case, a [2x100GE-2x100G](https://xrdocs.github.io/programmability/tutorials/2017-08-22-ncs1002-configuration-automation-overview/#1openconfig-configuration-for-2x100ge--2x100g-slice-mode) mode. Full YDK configuration examples for all modes can be found on [GitHub](https://github.com/CiscoDevNet/ydk-py-samples/tree/master/samples/basic/netconf/models/openconfig/openconfig-terminal-device).
As you remember, in “2x100GE2x100G” mode, the speed of each line port equals to the speed of any client port and this allows to have direct 1-to-1 mappings between client ports and line (trunk) ports. 
The code provided below is very basic by the intention to make it easier. Slice number, wavelengths and power levels are predefined in the code, but nothing prevents you from modifying this code according to your needs. 

There are three major OpenConfig models needed to configure a slice:
1. [openconfig-interfaces](https://github.com/openconfig/public/blob/master/release/models/interfaces/openconfig-interfaces.yang)
2. [openconfig-terminal-device](https://github.com/openconfig/public/blob/master/release/models/optical-transport/openconfig-terminal-device.yang)
3. [openconfig-platform](https://github.com/openconfig/public/blob/master/release/models/platform/openconfig-platform.yang)

They are complementary to each other (or, in other words, there are interdependences). That means that you need to commit configurations from all of them at the same time.
Let’s start with the first model listed, “openconfig-interfaces”. The task of this model is to make sure that line ports (trunks) are “UP”. This operation should be done on each port you have in the slice (if you like to follow [schemes](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/2_100_2_100.png), you can see that these ports correspond to “Transceivers on Line Ports”)
<div class="highlighter-rouge">
<pre class="highlight">
<code>   	
interface = oc_interface.Interface()
interface.name = 'Optics0/0/0/5' 
if_config = interface.Config()
if_config.name = 'Optics0/0/0/5'
if_config.type = iana_if_type.OpticalchannelIdentity()
if_config.enabled = True  ## ”True” means “no shut”
interface.config = if_config
oc_interface.interface.append(interface)
</code>
</pre>
</div>
(the code, that you will see on [GitHub](https://github.com/CiscoDevNet/ydk-py-samples/blob/master/samples/basic/netconf/models/openconfig/openconfig-terminal-device/nc-edit-config-oc-terminal-device-20-ydk.py)) will have several enhancements to remove repetitive configuration, but the overall logic stays the same as in this document) 

In the second model (“openconfig-terminal-device”) you need to define:
- Logical Ethernet Channels and map them to the client ports.
- Mappings between Ethernet Logical Channels and OTN Logical Channels.
- Mappings between OTN Logical Channels and Optical Channels.

Our final goal is to map the first client port to the first line port, and the last client port should be mapped to the second line port. You can find all the configurations for the first client/line ports pair below. Code will be similar for the second pair, and full configuration is available on [GitHub](https://github.com/CiscoDevNet/ydk-py-samples/blob/master/samples/basic/netconf/models/openconfig/openconfig-terminal-device/nc-edit-config-oc-terminal-device-20-ydk.py) 

a) At this step, you need to define Ethernet Logical Channels and make proper mapping between them and Transceivers. You also need to specify the speed, status, and Ethernet mode of the channel:
<div class="highlighter-rouge">
<pre class="highlight">
<code>  
channel = oc_logical_channel.logical_channels.Channel()
channel.index = 1 ## indexing can be any, except 0
channel_config = channel.Config()  
channel_config.rate_class = oc_tr_types.Trib_Rate_100GIdentity()
channel_config.admin_state = oc_tr_types.AdminStateTypeEnum.ENABLED
channel_config.trib_protocol = oc_tr_types.Prot_100G_MlgIdentity()
channel_config.logical_channel_type = oc_tr_types.Prot_EthernetIdentity()
channel.config = channel_config
channel_ingress = channel.Ingress()
channel_ingress_tr = channel_ingress.Config()
channel_ingress_tr.transceiver = '0/0-Optics0/0/0/0'
channel_ingress.config = channel_ingress_tr
channel.ingress = channel_ingress
</code>
</pre>
</div>
b) After you created Ethernet Logical Channels, you need to map them to OTN Logical Channels (will be created in the next step). At this step, you need to define the speed of the mapped Ethernet Logical Channel inside the OTN channel (things are easy for this mode, but with [5x100GE-2x250G]( https://xrdocs.github.io/programmability/tutorials/2017-08-22-ncs1002-configuration-automation-overview/#3openconfig-configuration-for-5x100ge--2x250g-slice-mode) mode you will need to split middle Ethernet Logical Channel into two different OTN Logical Channels in a 50/50 ratio)
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
channel_assignment = channel.logical_channel_assignments.Assignment()
channel_assignment.index = 1  
channel_assignment_config = channel_assignment.Config()
channel_assignment_config.allocation = Decimal64('100') 
channel_assignment_config.assignment_type = channel_assignment.config.AssignmentTypeEnum. LOGICAL_CHANNEL
channel_assignment_config.logical_channel = 51 
channel_assignment.config = channel_assignment_config
channel.logical_channel_assignments.assignment.append(channel_assignment)
oc_logical_channel.logical_channels.channel.append(channel)
</code>
</pre>
</div>
c) At this step, you need to define OTN logical channels, their speed and map them to Optical Channels (that are defined in the final step using “openconfig-platform” model): 
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
channel = oc_logical_channel.logical_channels.Channel()
channel.index = 51
channel_config = channel.Config()
channel_config.admin_state = oc_tr_types.AdminStateTypeEnum.ENABLED
channel_config.logical_channel_type = oc_tr_types.Prot_OtnIdentity()
channel.config = channel_config
channel_assignment = channel.logical_channel_assignments.Assignment()
channel_assignment.index = 1
channel_assignment_config = channel_assignment.Config()
channel_assignment_config.allocation = Decimal64('100')
channel_assignment_config.assignment_type = channel_assignment.config. AssignmentTypeEnum.OPTICAL_CHANNEL
channel_assignment_config.optical_channel = '0/0-OpticalChannel0/0/0/5’
channel_assignment.config = channel_assignment_config
channel.logical_channel_assignments.assignment.append(channel_assignment)
oc_logical_channel.logical_channels.channel.append(channel)
</code>
</pre>
</div>
Finally, in the “openconfig-platform” model you need to create Optical Channels and map them with real line ports on the slice. You also define power level, wavelength and FEC mode (mode1 = 7% or mode2 = 20%) for each trunk. Please consider, that in the “openconfig-platform” model power levels are defined in "0.01dM" and frequency is expressed in "MHz". 

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
component = oc_component.Component()
component.name = '0/0-OpticalChannel0/0/0/5'
optical_channel_config = component.optical_channel.Config()
optical_channel_config.line_port = '0/0-Optics0/0/0/5' 
optical_channel_config.operational_mode = 2 
optical_channel_config.target_output_power = 0 
optical_channel_config.frequency = 191300000
component.optical_channel.config = optical_channel_config
oc_component.component.append(component)
</code>
</pre>
</div>

At this step, we configured the main logic of our code, let’s have a look at other parts of the YDK code.

## Connecting to NCS1002

For a successful connection of your YDK code to a NCS1002, you need to have this configured on your device: 

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
xml agent tty
!
netconf agent tty
 session timeout 500
!
netconf-yang agent
 ssh
!
ssh server v2   ## requires key generation
</code>
</pre>
</div>

YDK code uses built-in NCCLIENT client module to handle the NETCONF connection establishment:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
from ydk.services import NetconfService
from ydk.providers import NetconfServiceProvider

provider = NetconfServiceProvider(address=device.hostname,
                                   port=device.port,
                                   username=device.username,
                                   password=device.password,
                                   protocol=device.scheme)

</code>
</pre>
</div> 

YDK will wrap-up the configuration based on OpenConfig models to XML format before sending to a NCS1002. Here is an example of such an output for 2x100GE2x100G mode that was explained in this document:

```
VOSIPCHU-M-C1GV:vosipchu$ python nc-create-oc-ncs1002-40-ydk.py ssh://root:lab@172.20.168.83 -v
2017-09-11 19:02:57,687 - ydk.providers.netconf_provider - INFO - NetconfServiceProvider connected to 172.20.168.83:None using ssh
2017-09-11 19:02:57,687 - ydk.services.netconf_service - INFO - Executing edit-config RPC
2017-09-11 19:02:57,704 - ydk.services.executor_service - INFO - Executor operation initiated
2017-09-11 19:02:57,712 - ydk.providers._provider_plugin - DEBUG - 
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:aded4537-5078-403c-a0ef-f205bdc10bf9">
  <edit-config>
    <target>
      <candidate></candidate>
    </target>
    <config>
      <interfaces xmlns="http://openconfig.net/yang/interfaces">
        <interface>
          <name>Optics0/0/0/5</name>
          <config>
            <enabled>true</enabled>
            <name>Optics0/0/0/5</name>
            <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:opticalChannel</type>
          </config>
        </interface>
        <interface>
          <name>Optics0/0/0/6</name>
          <config>
            <enabled>true</enabled>
            <name>Optics0/0/0/6</name>
            <type xmlns:idx="urn:ietf:params:xml:ns:yang:iana-if-type">idx:opticalChannel</type>
          </config>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>

2017-09-11 19:02:58,000 - ydk.providers._provider_plugin - DEBUG - 
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:aded4537-5078-403c-a0ef-f205bdc10bf9">
  <ok/>
</rpc-reply>

2017-09-11 19:02:58,001 - ydk.services.executor_service - INFO - Executor operation completed
2017-09-11 19:02:58,001 - ydk.services.netconf_service - INFO - Executing edit-config RPC
2017-09-11 19:02:58,016 - ydk.services.executor_service - INFO - Executor operation initiated
2017-09-11 19:02:58,020 - ydk.providers._provider_plugin - DEBUG - 
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:cbb88133-ef42-4eb1-b3ca-c8ef6472e52e">
  <edit-config>
    <target>
      <candidate></candidate>
    </target>
    <config>
      <terminal-device xmlns="http://openconfig.net/yang/terminal-device">
        <logical-channels>
          <channel>
            <index>100</index>
            <config>
              <admin-state>ENABLED</admin-state>
              <logical-channel-type xmlns:idx="http://openconfig.net/yang/transport-types">idx:PROT_ETHERNET</logical-channel-type>
              <rate-class xmlns:idx="http://openconfig.net/yang/transport-types">idx:TRIB_RATE_100G</rate-class>
              <trib-protocol xmlns:idx="http://openconfig.net/yang/transport-types">idx:PROT_100G_MLG</trib-protocol>
            </config>
            <ingress>
              <config>
                <transceiver>0/0-Optics0/0/0/0</transceiver>
              </config>
            </ingress>
            <logical-channel-assignments>
              <assignment>
                <index>1</index>
                <config>
                  <allocation>100</allocation>
                  <assignment-type>LOGICAL_CHANNEL</assignment-type>
                  <logical-channel>200</logical-channel>
                </config>
              </assignment>
            </logical-channel-assignments>
          </channel>
          <channel>
            <index>101</index>
            <config>
              <admin-state>ENABLED</admin-state>
              <logical-channel-type xmlns:idx="http://openconfig.net/yang/transport-types">idx:PROT_ETHERNET</logical-channel-type>
              <rate-class xmlns:idx="http://openconfig.net/yang/transport-types">idx:TRIB_RATE_100G</rate-class>
              <trib-protocol xmlns:idx="http://openconfig.net/yang/transport-types">idx:PROT_100G_MLG</trib-protocol>
            </config>
            <ingress>
              <config>
                <transceiver>0/0-Optics0/0/0/4</transceiver>
              </config>
            </ingress>
            <logical-channel-assignments>
              <assignment>
                <index>1</index>
                <config>
                  <allocation>100</allocation>
                  <assignment-type>LOGICAL_CHANNEL</assignment-type>
                  <logical-channel>201</logical-channel>
                </config>
              </assignment>
            </logical-channel-assignments>
          </channel>
          <channel>
            <index>200</index>
            <config>
              <admin-state>ENABLED</admin-state>
              <logical-channel-type xmlns:idx="http://openconfig.net/yang/transport-types">idx:PROT_OTN</logical-channel-type>
            </config>
            <logical-channel-assignments>
              <assignment>
                <index>1</index>
                <config>
                  <allocation>100</allocation>
                  <assignment-type>OPTICAL_CHANNEL</assignment-type>
                  <optical-channel>0/0-OpticalChannel0/0/0/5</optical-channel>
                </config>
              </assignment>
            </logical-channel-assignments>
          </channel>
          <channel>
            <index>201</index>
            <config>
              <admin-state>ENABLED</admin-state>
              <logical-channel-type xmlns:idx="http://openconfig.net/yang/transport-types">idx:PROT_OTN</logical-channel-type>
            </config>
            <logical-channel-assignments>
              <assignment>
                <index>1</index>
                <config>
                  <allocation>100</allocation>
                  <assignment-type>OPTICAL_CHANNEL</assignment-type>
                  <optical-channel>0/0-OpticalChannel0/0/0/6</optical-channel>
                </config>
              </assignment>
            </logical-channel-assignments>
          </channel>
        </logical-channels>
      </terminal-device>
    </config>
  </edit-config>
</rpc>

2017-09-11 19:02:58,309 - ydk.providers._provider_plugin - DEBUG - 
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:cbb88133-ef42-4eb1-b3ca-c8ef6472e52e">
  <ok/>
</rpc-reply>

2017-09-11 19:02:58,309 - ydk.services.executor_service - INFO - Executor operation completed
2017-09-11 19:02:58,309 - ydk.services.netconf_service - INFO - Executing edit-config RPC
2017-09-11 19:02:58,320 - ydk.services.executor_service - INFO - Executor operation initiated
2017-09-11 19:02:58,321 - ydk.providers._provider_plugin - DEBUG - 
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:0358444c-6097-4a80-8062-3232b3a87550">
  <edit-config>
    <target>
      <candidate></candidate>
    </target>
    <config>
      <components xmlns="http://openconfig.net/yang/platform">
        <component>
          <name>0/0-OpticalChannel0/0/0/5</name>
          <optical-channel xmlns="http://openconfig.net/yang/terminal-device">
            <config>
              <frequency>191300000</frequency>
              <line-port>0/0-Optics0/0/0/5</line-port>
              <operational-mode>2</operational-mode>
              <target-output-power>0</target-output-power>
            </config>
          </optical-channel>
        </component>
        <component>
          <name>0/0-OpticalChannel0/0/0/6</name>
          <optical-channel xmlns="http://openconfig.net/yang/terminal-device">
            <config>
              <frequency>196100000</frequency>
              <line-port>0/0-Optics0/0/0/6</line-port>
              <operational-mode>2</operational-mode>
              <target-output-power>0</target-output-power>
            </config>
          </optical-channel>
        </component>
      </components>
    </config>
  </edit-config>
</rpc>

2017-09-11 19:02:58,610 - ydk.providers._provider_plugin - DEBUG - 
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:0358444c-6097-4a80-8062-3232b3a87550">
  <ok/>
</rpc-reply>

2017-09-11 19:02:58,610 - ydk.services.executor_service - INFO - Executor operation completed
2017-09-11 19:02:58,610 - ydk.services.netconf_service - INFO - Executing commit RPC
2017-09-11 19:02:58,612 - ydk.services.executor_service - INFO - Executor operation initiated
2017-09-11 19:02:58,613 - ydk.providers._provider_plugin - DEBUG - 
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:dad37f83-ee8d-4049-be33-f7b4dbc595ac">
  <commit/>
</rpc>

2017-09-11 19:03:00,353 - ydk.providers._provider_plugin - DEBUG - 
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:dad37f83-ee8d-4049-be33-f7b4dbc595ac">
  <ok/>
</rpc-reply>

2017-09-11 19:03:00,353 - ydk.services.executor_service - INFO - Executor operation completed
2017-09-11 19:03:00,817 - ydk.providers.netconf_provider - INFO - NetconfServiceProvider disconnected from 172.20.168.83 using ssh
```
As you can see, YDK connects to the device, does its job, commits, and exits. You can also use non-verbose version (without “-v” in the command line) with less output to the console.

You can also check the updated configuration on a NCS1002 using “show” commands. New lines of config will be seen after your code successfully commits intended configuration. 
When you configure a slice using OpenConfig models, show output in CLI will be different, comparing to the output you have after configuration of a slice through CLI. That change was done to reflect OpenConfig logic. Proper protection mechanisms are implemented on NCS1002 to make sure that a slice configured through CLI cannot be configured again using OpenConfig models and vice versa (mutual exclusion). 

## CLI view from a NCS1002

Here is an output of Slice #0 configuration using OpenConfig models on a NCS1002:
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
!
controller Optics0/0/0/5
 transmit-power 0
 dwdm-carrier 100MHz-grid frequency 1913000
!
controller Optics0/0/0/6
 transmit-power 0
 dwdm-carrier 100MHz-grid frequency 1961000
!
terminal-device
 logical-channel 100
  rate-class 100G
  admin-state enable
  ingress-client-port Optics0/0/0/0
  trib-protocol 100G-MLG
  logical-channel-type Ethernet
  assignment-id 1
   allocation 100
   assignment-type logical
   assigned-logical-channel 200
  !
 !
 logical-channel 101
  rate-class 100G
  admin-state enable
  ingress-client-port Optics0/0/0/4
  trib-protocol 100G-MLG
  logical-channel-type Ethernet
  assignment-id 1
   allocation 100
   assignment-type logical
   assigned-logical-channel 201
  !
 !
 logical-channel 200
  admin-state enable
  logical-channel-type Otn
  assignment-id 1
   allocation 100
   assignment-type optical
   assigned-optical-channel 0_0-OpticalChannel0_0_0_5
  !
 !
 logical-channel 201
  admin-state enable
  logical-channel-type Otn
  assignment-id 1
   allocation 100
   assignment-type optical
   assigned-optical-channel 0_0-OpticalChannel0_0_0_6
  !
 !
 optical-channel 0_0-OpticalChannel0_0_0_5
  line-port Optics0/0/0/5
  operational-mode 2
 !
 optical-channel 0_0-OpticalChannel0_0_0_6
  line-port Optics0/0/0/6
  operational-mode 2
!
</code>
</pre>
</div>
Several helpful CLI “show” commands are listed here to check the configuration status using OC models:

a) A command to display accepted configuration layout with logical channels and their mappings:

<div class="highlighter-rouge">
<pre class="highlight">
<code> 

RP/0/RP0/CPU0:ios#<b>show terminal-device layout</b>
Thu Sep  12 09:36:28.139 UTC

Slice Id:                      0
Status:                        Config Accepted
Client Bitrate:                100G
Line Bitrate:                  100G

Client [Lane]     Logical Channel Logical Channel Optical Channel     Line
                    (Ethernet)      (Coherent)
Optics0/0/0/0 [0]     100             200 0_0-OpticalChannel0_0_0_5         Optics0/0/0/5
Optics0/0/0/4 [0]     101             201 0_0-OpticalChannel0_0_0_6         Optics0/0/0/6

</code>
</pre>
</div>

b) A command to display configuration details either for all logical channels or for a specific one:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ios#<b>show terminal-device logical-channel number 100</b>
Thu Sep  12 09:36:28.139 UTC
Logical Channel Index:      100
Name:                       HundredGigECtrlr0/0/0/0
Admin-State:                Enable
Loopback-Mode:              None
Type of Logical Channel:    Logical Level 1
Trib-Rate:                  100G tributary signal rate
Trib-Protocol:              100G MLG protocol
Protocol-Type:              Ethernet protocol framing
Ingress Client Port:        Optics0/0/0/0
Ingress Physical Channel:   0
Logical Assignment Index:   1
Logical Assignment Name:    NA
Logical Channel:            200
Optical Channel:            NA
Allocation:                 100G
Assignment Type:            Logical
</code>
</pre>
</div>
c) A command to display supported operational modes:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ios#<b>show terminal-device operational-modes</b>
Thu Sep  12 09:36:28.139 UTC
Operational Mode:       1
Description:            FEC Mode 7
Vendor:                 Cisco Systems, Inc.

Operational Mode:       2
Description:            FEC Mode 20
Vendor:                 Cisco Systems, Inc.
</code>
</pre>
</div>

Openconfig models for slice configuration came with IOS XR 6.2.1 release. 
Python 2.7 and YDK version 0.5.4 were used for scripts.


## Conclusion

YDK is a great tool that helps you to make configuration easier. In our example, we configured a full NCS1002 slice using OpenConfig models with a basic YDK script. You don’t need to deal with XML encoding, you don’t need to be an YANG expert to check semantics of your code, as YDK does it all for you. Try all example codes, modify them as you need and stay tuned for our next updates!