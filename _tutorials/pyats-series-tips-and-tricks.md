---
published: true
date: '2023-04-26 16:09 +0200'
title: pyATS Series - Tips and Tricks
position: hidden
author: Antoine Orsoni
tags:
  - iosxr
  - cisco
---
{% include toc icon="table" title="Table of Contents" %}
![pyats_hello2.jpg]({{site.baseurl}}/images/pyats_hello2.jpg){: .align-center}

# Introduction

Ever dreamed of a test framework that could be used across multiple platforms, OS and vendors, which could do regression, sanity and feature testing; already used by thousands of engineers and developers worldwide? Guess what, **it exists, it’s free, and you can start using it right now!**

pyATS (**Py**thon **A**utomated **T**est **S**ystems, to be pronounced "py A. T. S.") was first created as an internal project, to ease the validation of two OS versions. It has been made public in 2017 through **Cisco Devnet**.

Many thanks to Romain Cyrille, Cisco CX Engineer, for your help writting this article!
{: .notice--info}

# Other pyATS episodes

You've missed the first episode? You would like to read more? Below the list of published episodes:

| Episode 	| URL                                                                                              	| What's covered                                        	|
|---------	|--------------------------------------------------------------------------------------------------	|-------------------------------------------------------	|
| **1 - Install and use pyATS**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/){: .btn}  	|  What's pyATS, Install pyATS, Collect a raw CLI output 	|
| **2 - Parsing like  a pro**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-parsing-like-a-pro/){: .btn} 	|  Explore pyATS libraries, Collect and parse a CLI output        	|
| **3 - Be a model**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-be-a-model/){: .btn} 	|  What a pyATS model and when to use it        	|
| **4 - Collecting many show commands**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-collecting-many-show-commands/){: .btn} 	|  How to collect many show commands on many devices? |
| **5 - Tips and Tricks**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-tips-and-tricks/){: .btn} 	|  pyATS Tips and Tricks |

# Abstract

This article is intended to share my Tips and Tricks I wish I knew when I started to use pyATS. It's based on my own experience and does not intend to be exhaustive. I will try to update this article frequently.

Do you have a Tips or Trick that could benefit everyone? Feel free to share it in the comments!
{: .notice--info}

# pyATS installation and maintenance

This section will give general Tips and Tricks about pyATS installation and maintenance.

## Installing pyATS

After checking the [Requirements](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/prereqs/prerequisites.html#requirements), you can install pyATS and pyATS librairies by running the below command.

```
> pip install "pyats[full]"
```

## Veryfying the installation

You can verify pyATS has been successfuly installed by running the below command. It should return the current pyATS version, with a similar output. This command also shows if there are any available package updates.

```
> pyats version check

You are currently running pyATS version: 23.3
Python: 3.10.4 [64bit]

  Package                      Version
  ---------------------------- -------
  genie                        23.3   
  genie.libs.clean             23.3   
  
  ## output chunked for brevity ## 
  
  yang.connector               23.3   
```

## Updating pyATS

You can check if there is a newer pyATS version and update it with the below command. It should return a similar output.

```
> pyats version update

Checking your current environment...


The following packages will be removed:

  Package                      Version
  ---------------------------- -------
  genie                        23.3   
  genie.libs.clean             23.3   
    
  ## output chunked for brevity ## 
 
  unicon.plugins               23.3   
  yang.connector               23.3   


Fetching package list... (it may take some time)

... and updated with:

  Package                      Version      
  ---------------------------- -------------
  genie                        latest (23.4)
  genie.libs.clean             latest (23.4)

  ## output chunked for brevity ## 

  unicon.plugins               latest (23.4)
  yang.connector               latest (23.4)


Are you sure to continue [y/N]? 
```

# pyATS Testbed

This section will give general Tips and Tricks about pyATS testbeds.

## Testbed is not in the directory of execution

When the testbed cannot be found, you have a similar error as the output below. Double check the path to your testbed file. By default, pyATS is looking for a testbed file in the directory where you execute the command.

```
pyats.utils.yaml.exceptions.LoadError: Content of 'testbed.yaml' failed to load into a dict.
```

## Default credentials and CLI	

When credentials are shared by many devices (ex: in a lab), it could be useful to set them as default. You should add them in your `testbed.yaml` file. You can still use specific credentials (i.e. non-default) for other devices in your testbed. 

In the below example, `xr1` and `xr2` will use all default credentials. `xr3` will non-default credentials.

Only `credentials` can have default values. Objects like `os` or `connections` have to be set for each device in the testbed. Testbed and Devices configurable information can be found in [the documentation](https://pubhub.devnetcloud.com/media/pyats/docs/topology/concept.html#testbed-object).
{: .notice--info}

```
testbed:										# Default information section
    credentials:
        default:                                # Default credentials
            username: cisco
            password: cisco
       
devices:
    xr1:                                    
        connections:
            cli:
                ip: 192.168.1.11
                protocol: ssh
                port: 22
        os: iosxr									
    	platform: iosxrv
    	type: router
    xr2:                                    
        connections:
            cli:
                ip: 192.168.1.12 
                protocol: ssh
                port: 22
        os: iosxr									
    	platform: iosxrv
    	type: router
    xr3:
    	connections:
            cli:
                ip: 192.168.1.12
                protocol: ssh
                port: 22
        credentials:
        	default:                                
            	username: antoine
            	password: cisco123
        os: iosxr									
    	platform: iosxrv
    	type: router
```

By default, the name of your device in the testbed should **EXACTLY** match the hostname of your device. This information is case sensitive.
{: .notice--info}

## pyATS Supported Operating Systems and Platforms

pyATS relies on Unicon to support network devices. They are described with their operating system `os` (ex: `iosxr`) and optionally their `platform` (ex: `spitfire`, for Cisco 8000) and `model` (rarely used).

You can find supported platforms by Unicon in the [pyATS documentation](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/supported_platforms.html).

Ensure that devices you are using are accurately represented as this will serve as the source of truth for [Genie Abstract](https://pubhub.devnetcloud.com/media/genie-docs/docs/abstract/index.html) as well in a near future update.
{: .notice--info}

## Using a SSH Proxy

Your device might be accessible via a proxy (ex: Bastion). You can add a proxy to your testbed and indicates for which device you should first connect to the proxy.

In the below example, I will connect to `xrd1` via a proxy and to `xrd2` directly. `jumphost` will store information regarding how to connect to the proxy. You can name it how you want, as long at it matches with the device proxy value. Note that `jumphost` you could also make `jumphost` use the default credentials, as seen before.

The associated bash command would be: `ssh -J iosxr4ever@192.168.1.1 cisco@10.10.10.1`. The first `login/ip` are the ones of the proxy, the second couple are the ones from the device.

```
testbed:
  credentials:
    default:
      username: cisco
      password: cisco123

devices:
  jumphost:
    os: linux
    platform: linux
    type: linux
    connections:
      cli:
        ip: 192.168.1.1
        port: 22
        protocol: ssh
     credentials:
       default:
         username: iosxr4ever
         password: cisco!
  xrd1:
    os: iosxr
    platform: iosxrv
    type: router
    connections:
      cli:
        ip: 10.10.10.1
        proxy: jumphost
        port: 22
        protocol: ssh
    xrd2:
    os: iosxr
    platform: iosxrv
    type: router
    connections:
      cli:
        ip: 10.10.10.2
        port: 22
        protocol: ssh
```

More information about how to use proxy in the [pyATS documentation](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/proxy.html).
{: .notice--info}

## Validating a Testbed file

You can verify that there is no typo or error in your testbed file by using the below command. If your testbed has error, it should look like the below output.

```
Loading testbed file: testbed.yaml
--------------------------------------------------------------------------------

YAML Lint Messages
------------------
  36:24     error    no new line character at the end of file  (new-line-at-end-of-file)
```

## pyATS secret

If you don't want your credentials (ex: login, passwords) to appear as cleartext in your testbed, you can use the `pyats secret` tool. More information in the [pyATS documentation](https://pubhub.devnetcloud.com/media/pyats/docs/utilities/secret_strings.html).

## Avoid printting the default commands after connecting to a device

By default, after connecting to a device, pyATS will send a bunch of `exec` and `configuration` level commands. It will also send logging to standard output. You can disable them by editing their respective arguments: `init_exec_commands`, `init_config_commands` and `log_stdout` in the testbed `connection` `settings` parameter; like in the below example. 

Note that `init_exec_commands` and  `init_config_commands` should be list of commands to use when initializating the connection. `log_stdout` should be a boolean option to enable/disable logging to standard output.

```
  xrd1:
    os: iosxr
    platform: iosxrv
    type: router
    connections:
      cli:
        ip: 10.10.10.1
        proxy: jumphost
        port: 22
        protocol: ssh
      settings:
        init_exec_commands: []
        init_config_commands: []
        log_stdout: False
```

It's not advised to use `init_exec_commands: []` as your device might not have `terminal width` and `terminal length 0` commands. You could have issues when with long outputs.
{: .notice--warning}

More information about the device `connect()` method in the [pyATS documentation](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/connection.html).
{: .notice--info}

## Disconnecting quickly

When you disconnect from a device, using the `disconnect()` method, Unicon will wait about 10 seconds. The documentation says this is to prevent connection issues on rapid connect/disconnect sequences. It can be annoying when you script connect and disconnect from many devices.

To change the default timers, you can change the `GRACEFUL_DISCONNECT_WAIT_SEC` and `POST_DISCONNECT_WAIT_SEC` in the testbed `connection` `settings` parameter; like in the below example.

```
  xrd1:
    os: iosxr
    platform: iosxrv
    type: router
    connections:
      cli:
        ip: 10.10.10.1
        proxy: jumphost
        port: 22
        protocol: ssh
      settings:
        GRACEFUL_DISCONNECT_WAIT_SEC = 0
        POST_DISCONNECT_WAIT_SEC = 0
```

More information about the device `disconnect()` method in the [pyATS documentation](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/connection.html).
{: .notice--info}

# Using pyATS with Python

## Mismatch between the hostname and the keys in the testbed

The name of the device in your testbed, and the hostname **MUST** match. It's case sensitive. In case it doesn't match, you will have a similar error.

```
unicon.core.errors.TimeoutError: timeout occurred:
```

You can also specify in your Python script that you do not care if they don't match by setting the `learn_hostname` argument to `True` in your device `connect()` method.

```
device.connect(learn_hostname=True)
```

More information about the device `connect()` method in the [pyATS documentation](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/connection.html).
{: .notice--info}

## Avoid printting the default commands after connecting to a device

By default, after connecting to a device, pyATS will send a bunch of `exec` and `configuration` level commands. It will also send logging to standard output. You can disable them by editing their respective arguments: `init_exec_commands`, `init_config_commands` and `log_stdout` in the device `connect()` method; like in the below example.

```
device.connect(init_exec_commands=[],
               init_config_commands=[],
               log_stdout=False)
```

It's not advised to use `init_exec_commands: []` as your device might not have `terminal width` and `terminal length 0` commands. You could have issues when with long outputs.
{: .notice--warning}

More information about the device `connect()` method in the [pyATS documentation](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/connection.html).
{: .notice--info}

## Parsing an already saved output

If you already have an output (ex: in a text file) that you would like to parse with pyATS, you can use the below commands. You first need to load a device from a testbed, so the pyATS librairies know from which OS it should look for the appropriate parser.

```
from genie.testbed import load

device = load('./testbed.yaml')['device']

with open('./output.txt') as file:
  output = file.read()
  
device.parse('<show command>', output=output)
```

# Conclusion

In this fifth episode of the pyATS series, we saw a bunch of useful Tips and Tricks based on my experience with pyATS.

The code used for each blog post can be found [here](https://github.com/AntoineOrsoni/xrdocs-how-to-pyats). This link will include the code for all posts.
{: .notice--info}

# Resources

Below a few useful pyATS resources.

- [List of supported pyATS parsers](https://pubhub.devnetcloud.com/media/genie-feature-browser/docs/#/),
- [The official pyATS documentation](https://pubhub.devnetcloud.com/media/pyats/docs/getting_started/index.html),
- [List of Unicon supported platforms](https://pubhub.devnetcloud.com/media/unicon/docs/user_guide/supported_platforms.html),
- [Devnet code exchange](https://developer.cisco.com/codeexchange/),
- [Join the Webex space with the pyATS community](https://eurl.io/#r18UzrQVr).
