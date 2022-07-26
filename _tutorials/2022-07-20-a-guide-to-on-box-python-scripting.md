---
published: true
date: '2022-07-20 15:57 -0700'
title: A Guide to On-Box Python Scripting
author: Eliot Pickhardt
excerpt: Programmability using python scripts on IOS-XR
tags:
  - iosxr
  - cisco
hidden: true
position: hidden
---
{% include toc title="On-Box Scripting" %}

## Programmability of IOS XR  

Being a network engineer no longer requires poring over CLI commands for hours on end, ensuring every detail conforms to the requirements, just for a link to break, causing even more work. Using the programmability capabilities of IOS XR, you eliminate much of the tedious router-by-router configuration by manipulating the existing data models automatically. There are many ways to tap into the automated potential of IOS XR, including on-box Python scripting.

## Python Automation Scripts
Automation scripts are one way to leverage IOS XR to work for you. These are mainly Python scripts that run on-box. These scripts can work in four different ways to aid the configuration and maintenance of your network. 

[**Exec Scripts**](#exec-scripts): These scripts are run manually, but they can dramatically decrease the work required for configuration or other operational tasks.

[**Config scripts**](#config-scripts): These scripts run automatically every time a configuration change is committed. They are useful to ensure that a commit doesn’t go against any network rules, and can take action or throw errors if rules are broken.

[**Process Scripts**](#process-scripts): These scripts run continually once manually activated. They perform typical checks and will exit upon a preconfigured condition. They daemonize normal system monitoring.

[**EEM Scripts**](#eem-scripts): These are event-driven scripts, and can be configured to run under a variety of conditions, such as a preconfigured timing interval or in response to a system condition. They operate similarly to Exec scripts in that they can aid in configuration or system maintenance. 

> Note: As of IOS XR release 7.5.1, EEM scripts are not supported

![Scripts](https://xrdocs.github.io/xrdocs-images/assets/images/pickhardt-scripts.png)

### Using Syslog in Python Scripts
All types of on-box python scripts have access to the logging capabilities of IOS XR. As script-writers, we can utilize <a href="https://www.cisco.com/c/en/us/td/docs/routers/access/wireless/software/guide/SysMsgLogging.html#wp1054858" target="_blank">all levels of syslog</a> messaging, which represents the centralized logging standard for IOS-XR. The following example illustrates how you can leverage system logging within your scripts:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
from cisco.script_mgmt import xrlog

syslog = xrlog.getSysLogger('script-name')

<mark>syslog.{severity-level}("message")</mark>
</code>
</pre>
</div>

![Scripts](https://xrdocs.github.io/xrdocs-images/assets/images/pickhardt-script-exec.png)

## Exec Scripts

Exec scripts represent the most basic on-box scripting available within IOS XR. These scripts provide a way for the network managers to manually deploy programs that simplify their work as a whole, including automating the configuration process. In order to do this, we must leverage the `iosxr.xrcli.xrcli_helper` library. 

A simple, but useful, function from this library is `xrcli_exec(command)`. This function takes a command as input and returns the CLI output that you would see if you had just issued this command. 

### Manipulating String Arguments to Enable Complex Configurations
Being able to commit configurations from inside a python script enables us to streamline many of the repetitive CLI configurations we complete. Thus, exec scripts will commonly use the `xr_apply_config_string(configuration)` function from this library. This function takes a CLI configuration command as input, and then commits the provided configuration. However, the argument of this function is a *single-line* configuration, meaning we must make use of a few tricks to issue complex configurations.  

Using string formatters allows us to adjust the configuration to a command-line argument or an environment variable:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
xrcli_helper.xr_apply_config_string("interface <mark>%s</mark>" %interface_name)
</code>
</pre>
</div>
  
where `%s` indicates a string insertion.  

A second tool we can use is escape characters. Specfically, the `\n\r` combination will provide similar function to pressing "Enter" while using the CLI. This allows us to dramatically increase the complexity of commands used within this method.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
xrcli_helper.xr_apply_config_string("interface TenGigE0/0/0/1 <mark>\n\r</mark> ipv4 address 10.0.0.2 <mark>\n\r</mark> no shutdown")
</code>
</pre>
</div>
  
Using a combination of these two techniques enables us to maximize the potential of this function and Exec scripts overall.  Let's walk through an example:

### Exec Script Example

The script we'll be dissecting is [`test_cli_show_version.py`](https://github.com/CiscoDevNet/xr-python-scripts/blob/main/exec/test_cli_show_version.py). This is a relatively simple exec script, issuing a single command and printing its output. 

We begin by importing regex operations and aforementioned Cisco libraries:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
import re
from iosxr.xrcli.xrcli_helper import *
from cisco.script_mgmt import xrlog
</code>
</pre>
</div>

Then, we instantiate the SysLogger and CLI Helper:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
syslog = xrlog.getSysLogger('test_cli_show_version')
helper = XrcliHelper(debug = True)
</code>
</pre>
</div>

Within our function, we simply issue a command. If the operation was successful, search through the output and print it to syslog, otherwise, we throw an error to syslog:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
 cmd = "show version"
 result = helper.xrcli_exec(cmd)
 print(result)

 if result['status'] == 'success':
    syslog.info('SCRIPT : Show version successful')
    m = re.search(r'[^Version ]*$',result['output'])
    syslog.info("Script found " + m.group(0))

 else:
    syslog.error('SCRIPT : Show version failed')   
</code>
</pre>
</div>

Before we run this script, we must go through the steps of activating the script with IOS-XR. Thankfully, there is documentation available [here](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-5/programmability/configuration/guide/b-programmability-cg-asr9000-75x/exec-scripts.html) that explains it. 

However, there is one step that the guide doesn't mention, which is setting AAA permissions. For exec scripts, we must give the script the proper clearances before it will be effective. For my exec scripts, I use the following permissions:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
aaa authorization exec default group tacacs+ local
aaa authorization eventmanager default local
aaa authentication login default group tacacs+ local
</code>
</pre>
</div>

These might be slightly different depending on the particular use case for the script.

You should now be ready to create your very own exec script!

I also wrote a more complex exec script available [here](***INSERTLINK***), which initiates OSPF neighborship between routers on the same subnet.

![Scripts](https://xrdocs.github.io/xrdocs-images/assets/images/pickhardt-script-config.png)

## Config Scripts
As mentioned, config scripts are the best way to ensure that a commit doesn’t violate any existing rules for the network. Each config script should be relatively specific in its use (ie, regarding one protocol). Breaking down the general form of these scripts will help us understand exactly how they work.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
import cisco.config_validation as xr
</code>
</pre>
</div>

### Registering the Callback Function on a Path

The `cisco.config-valiation` Python library contains a few functions that are critical to the operation of config scripts. The first of these methods has the following signature:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
xr.register_validate_callback(path, cb_fn)
</code>
</pre>
</div>

This function tells IOS-XR when to execute the script, and what callback fuction to run. A well-written config script will not be called after every configuration attempt, because configurations will not pertain to the script in any way. 

The first argument, `path`, determines whether the configuration data relates to the validation occurring in the callback function, and is of the type `YPath`. Specifically, `path` is a schema path, which is a `YPath` that doesn’t point to a specific instance of a node. This means that no keys in the path are specified. A schema path conforms to the following format:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
"module-name:container/container2/another-module:container/leaf"
</code>
</pre>
</div>

The most effective way to derive the correct YPath is to search for the intended data within pertinent YANG models. If there is more than one node that should trigger the config script, a wildcard can be used as follows:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
"module-name:container/container2/another-module:container/*"
</code>
</pre>
</div>

This path will tell the script to execute when any leaf node under the last container is edited, created, or deleted.
There are a number of limitations on the available models that can be used for config scripts. Only XR-native YANG models are currently supported. Also, in order for the callback function to properly access the configuration data, the path needs to be from a “cfg” model, not an “oper” model. 

The second argument to this method, `callback_function`, is what will be run whenever a relevant commit is pushed to the configuration. The callback function is the “meat” of the config script. 

### The Callback Function

The signature of the function needs to be:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
def cb_fn_name(root):
    #function goes here
</code>
</pre>
</div>

where `root` is the root node.

This function retrieves specific nodes from the YANG model in order to check configuration status, including containers, leaves, and leaf-lists. In order to do this, we can use either the `<node>.get_node(path)` or `<node>.get_list(path)`. The first call of these functions should be called on the root node, which is passed to the callback function as an argument. Intuitively, `get_node` can be used to get leaf or container nodes, while `get_list` should be used to obtain leaf-list types. For each of these methods, the path is a specific instance of a YPath, called a data path. Unlike the schema path mentioned in the previous section, the keys of the path must be included to find a specific node or list. A data path matches the following skeleton: 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
"module-name:container/container2<mark>[key='value']</mark>/another-module:container<mark>[id=number]</mark>/leaf"
</code>
</pre>
</div>

The only difference between the data and schema paths is that the required key-value pairs are specified in the data path, while a schema path excludes them.

Key values should be surrounded by single quotes (key='value') except when the data type is an integer (id=number).

If the node retrieved is a leaf node, you can use the `.value` attribute to retrieve the associated data. Meanwhile, leaf-lists can be iterated through to get the leaf nodes within them. 

After the relevant data has been retrieved from the models, the necessary validation can occur. If the current configuration data doesn't comply with the script's standards, there are a few venues available to the script writer to correct the discrepancies. The first route to take is to generate warnings or errors in the syslog.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
syslog.error("Something is wrong!")
</code>
</pre>
</div>

This will allow the commit to pass, so this technique should be implemented when there is a minor problem with the configuration data. 

Next, you can block the commit by generating errors from the `cisco.config_validation` library. This will also send a custom error message to the user. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
xr.add_error(curr_node, "Something is wrong! Can't commit changes")
</code>
</pre>
</div>

This strategy should be employed when configuration doesn't match the desired values, but there is there is no obvious way to correct the problem.

If there is a straightforward solution, you can change configuration data automatically using the `curr_node.set_node(path, data)` or `curr_node.set(data)` methods. The former allows you to set the data of the current node or any child node relative to the current node, while the latter works for the current node only. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
curr_node.set_node("/leaf", data_to_set)

curr_node.set(data_to_set)
</code>
</pre>
</div>

Using these methods will allow the commit to pass with the changes made by the script. It is recommended to provide some information to syslog in order to inform the user what changes occurred.

### Config Script Example

Let's take a look at an application of these techniques with a simple script. This specific program will check to see if a specified ACL is present on a given interface when any ACL-related configuration is pushed. 

Here are the required import statements and instantiations:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
import cisco.config_validation as xr

from cisco.script_mgmt import xrlog
syslog = xrlog.getSysLogger('check_acl')

interface_name = "TenGigE0/0/0/10"
acl_name = "access-list-1"
</code>
</pre>
</div>

With config scripts, it's most logical to start with the callback validation funciton, which for this example, uses two different models: an [interface-configuration](https://github.com/YangModels/yang/blob/af90c053a1ca9b01a3f229e313e4af7e5b849c87/vendor/cisco/xr/751/Cisco-IOS-XR-ifmgr-cfg.yang) model along with a [pfilter](https://github.com/YangModels/yang/blob/af90c053a1ca9b01a3f229e313e4af7e5b849c87/vendor/cisco/xr/751/Cisco-IOS-XR-ip-pfilter-cfg.yang#L11) model. Notice that the pfilter model imports the interface configuration model, extending its reach. This function ensures that if a configuration is pushed that regards the ACL or any of the child nodes, this script will be called. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
xr.register_validate_callback(["/<mark>ifmgr-cfg</mark>:interface-configurations/ifmgr-cfg:interface-configuration/<mark>ip-pfilter-cfg</mark>:ipv4-packet-filter/*"], check_acl)
</code>
</pre>
</div>

Naturally, `check_acl` is the callback function that will perform the desired examination of the configuration data. 

The first part of this function is to retrieve the interface that is specified by the script:
<div class="highlighter-rouge">
<pre class="highlight">
<code>
int_config = root.get_node("/ifmgr-cfg:interface-configurations/interface-configuration[active='act',interface-name='%s']" %interface_name)
if int_config:
	syslog.info("Interface found")
</code>
</pre>
</div>
Essentially, we're using the YPath to see if an interface exists with the specified name. If so, we send an informational message to syslog. An important detail to notice is how the keys, 	`active` and `interface-name` are specified within this YPath.

We then attempt to find the list of ACLs registered under this interface. As you can see, we must use the `get_list` function in place of `get_node`.
<div class="highlighter-rouge">
<pre class="highlight">
<code>
acl = int_config.get_list("/ip-pfilter-cfg:ipv4-packet-filter/inbound/acl-name-array")
    if acl:
    	syslog.info("ACL list found")
</code>
</pre>
</div>

Finally, we iterate through the list of nodes to see if there is an ACL within the list with the provided name. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
	if acl_name in [x.value for x in acl]:
		syslog.info("ACL found")
    else:
		syslog.error("ACL not found")
</code>
</pre>
</div>

Normal list iteration is effective for leaf-lists, and we can access the `value` attribute of the leaf nodes.

Similarly to exec scripts, we must go through the steps of activating the script with IOS-XR. You can follow [this](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-5/programmability/configuration/guide/b-programmability-cg-asr9000-75x/config-scripts.html)  guide for the process of how to do so.

You now have the information you need to begin implementing config scripts on your router. 

This script helped to illustrate some of the methods that config scripts utilize, but is limited in practicality. I created another sample config script that demonstrates more of the capabilites of config scripts. The link to that script can be found [here](***INSERTLINK***).
  

![Scripts](https://xrdocs.github.io/xrdocs-images/assets/images/pickhardt-script-process.png)
  
## Process Scripts
  
Process scripts are the best way to automatically monitor operational data within IOS-XR. Since process scripts run continuously by nature, we must register them with AppMgr for them to run. Information about how to correctly set up process scripts can be found [here](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-5/programmability/configuration/guide/b-programmability-cg-asr9000-75x/process-scripts.html). 

Process scripts begin with a typical Python `if __name__ == "__main__":` statement. In this statement, there must be an infinite loop, which can call any necessary helper functions. Many process scripts also utilize the `time` library, which allows the script to wait for a number of seconds (or minutes) at the end of the script before running again. This is helpful in saving power, since script execution is suspended during this time. 


### NETCONF RPCs

When creating a process script, a common practice is to use a NETCONF RPC to retrieve and edit operational data. We can import `NetconfClient` from the `iosxr.netconf.netconf_lib` in order to access an RPC. All [NETCONF operations](https://en.wikipedia.org/wiki/NETCONF#Operations:~:text=SNMP%20modeling%20language.-,Operations,-%5Bedit%5D) are available with this rpc with the following syntax:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
nc = NetconfClient(debug=True)
nc.connect()
<mark>nc.{operation}(file=None, request=None)</mark>
nc.close() #always close the netconf session when you are done
</code>
</pre>
</div>

In order to find the correct piece of data, we should use a string representing an XML filter. The structure of this string is:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
"""
&lt;container1 xmls="http:cisco.com/ns/yang/Cisco-IOS-XR-{namespace-of-YANG-model}"&gt;
    &lt;list&gt;
        &lt;key-to-list&gt;<mark>KeyData</mark>&lt;/key-to-list&gt;
            &lt;leaf-name/&gt;
    &lt;/list&gt;
&lt;/container1&gt;
"""
</code>
</pre>
</div>

where the leaf contains the data we wish to monitor/manipulate. This filter string should be passed to the `request` argument of the NETCONF function. This template illustrates how to format the XML filter, including key-value pairs.

We can use the reply from the Netconf Client operation (`nc.reply`) and details about the filter path to retrieve the desired data from here. The reply will be in XML, so we will have to employ some parsing strategies to extract the specific data. Demonstration of how to do this can be found in the example and videos at the end of this section. 


### Process Script Example

Let's walk through a fairly simple process script together. This script finds the number of alarms present on the router every minute and sends the information to syslog. Here are the familiar libraries and initializations:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
import time
import os
import xmltodict
import re

from cisco.script_mgmt import xrlog
from iosxr.netconf.netconf_lib import NetconfClient

log = xrlog.getScriptLogger('Alarm')
syslog = xrlog.getSysLogger('Alarm')
</code>
</pre>
</div>
Next, let's look at the `main` function, which executes the script every sixty seconds within an infinite loop: 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
if __name__ == '__main__':
	while(1):
		check_curr_alarm_num()
		time.sleep(60)
</code>
</pre>
</div>

We now write our `check_curr_alarm_num` function. The `Netconf` client is started, and the filter string is used to retrieve the data associated with the number of alarms. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
def check_curr_alarm_num():
	"""
	Checks current number of alarms
	"""
	curr_count = 0
	filter_string = """
	&lt;alarms xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-alarmgr-server-oper"&gt;
		&lt;detail&gt;
			&lt;detail-system&gt;
				&lt;stats>
					&lt;reported/&gt;
				&lt;/stats&gt;
			&lt;/detail-system&gt;
		&lt;/detail&gt;
	&lt;/alarms&gt;"""

	nc = NetconfClient(debug=True)
	nc.connect()
	do_get(nc, filter=filter_string)
</code>
</pre>
</div>

Here is the `do_get` function, which helps to demonstrate how the NETCONF client can be used. We see the `nc.rpc.get` call, but other NETCONF operations could be used with this client.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
def do_get(nc, filter=None, path=None):
	try:
		if path is not None:
			nc.rpc.get(file=path)
		elif filter is not None:
			nc.rpc.get(request=filter)
		else:
			return False
	except Exception as e:
			return False
	return True
</code>
</pre>
</div>

Returning to our initial function, we convert the reply from the client to a dictionary using the `xmltodict` library's capabilities. The dictionary is used to retrieve the integer value of the number of alarms, the number is printed to syslog, and the client is closed. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
	ret_dict = _xml_to_dict(nc.reply, 'alarms')
	curr_count = int(ret_dict['alarms']['detail']['detail-system']['stats']['reported'])
	syslog.info(curr_count)
	nc.close()
</code>
</pre>
</div>

Here is the `_xml_to_dict` helper method. Without diving too deeply into `XML`, the function searches through the reply for a specific pattern and translates that into a dictionary, which allows us much simpler access to the data.

<div class="highlighter-rouge">
<pre class="highlight">
<code>

def _xml_to_dict(xml_output, xml_tag=None):
	"""
	convert netconf rpc request to dict
	:param xml_output:
	:return:
	"""
	if xml_tag:
		pattern = "<data>\s+(<%s.*</%s>).*</data>" % (xml_tag, xml_tag)
	else:
		pattern = "(<data>.*</data>)"
	xml_output = xml_output.replace('\n', ' ')
	xml_data_match = re.search(pattern, xml_output)
	ret_dict = xmltodict.parse(xml_data_match.group(1))
	return ret_dict

</code>
</pre>
</div>

Once again, we must properly activate the script before it will run. Process scripts have a slightly different implementation procedure because they are running constantly. Thus, we use AppMgr to set them up. Find out more about this routine [here](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-5/programmability/configuration/guide/b-programmability-cg-asr9000-75x/process-scripts.html)

Now that we have set up the script with AppMgr, you have all the tools you need to make process scripts that align with your particular solutions. 

The purpose of this example was to demonstrate some of the key aspects of process scripts, but the applications of the example are limited. I buffed out the script to provide a possible use case [here](***INSERTLINK***)

## EEM Scripts
As EEM scripts are not currently supported, there is no script-writing guide available. 

**Full script examples can be found at the xr_python_scripts [GitHub repository](https://github.com/CiscoDevNet/xr-python-scripts)**
