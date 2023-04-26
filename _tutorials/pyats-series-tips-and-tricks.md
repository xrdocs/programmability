---
published: false
date: '2023-04-26 16:09 +0200'
title: pyats-series-tips-and-tricks
position: hidden
author: Antoine Orsoni
tags:
  - iosxr
  - cisco
---
![pyats_hello2.jpg]({{site.baseurl}}/images/pyats_hello2.jpg){: .align-center}

# Introduction

Ever dreamed of a test framework that could be used across multiple platforms, OS and vendors, which could do regression, sanity and feature testing; already used by thousands of engineers and developers worldwide? Guess what, **it exists, it’s free, and you can start using it right now!**

pyATS (**Py**thon **A**utomated **T**est **S**ystems, to be pronounced "py A. T. S.") was first created as an internal project, to ease the validation of two OS versions. It has been made public in 2017 through **Cisco Devnet**.

This blog post will be the second one of a series on pyATS. Today, we will explore pyATS libraries (also known as Genie), and we will collect our first **parsed output**. More use cases are going to be covered in the next posts. 

# Other pyATS episodes

You've missed the first episode? You would like to read more? Below the list of published episodes:

| Episode 	| URL                                                                                              	| What's covered                                        	|
|---------	|--------------------------------------------------------------------------------------------------	|-------------------------------------------------------	|
| **1 - Install and use pyATS**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-install-and-use-pyats/){: .btn}  	|  What's pyATS, Install pyATS, Collect a raw CLI output 	|
| **2 - Parsing like  a pro**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-parsing-like-a-pro/){: .btn} 	|  Explore pyATS libraries, Collect and parse a CLI output        	|
| **3 - Be a model**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-be-a-model/){: .btn} 	|  What a pyATS model and when to use it        	|
| **4 - Collecting many show commands**       	| [Link](https://xrdocs.io/programmability/tutorials/pyats-series-collecting-many-show-commands/){: .btn} 	|  How to collect many show commands on many devices?

# pyATS Tips and Tricks

This article is intended to share my Tips and Tricks I wish I knew when I started to use pyATS. It's based on my own experience and does not intend to be exhaustive. I will try to update this article frequently.

Do you have a Tips or Trick that could benefit everyone? Feel free to share it in the comments!
{: .notice--info}

## pyATS installation and maintenance

This section will give general Tips and Tricks about pyATS installation and maintenance.

### Installing pyATS

After checking the [Requirements](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/prereqs/prerequisites.html#requirements), you can install pyATS and pyATS librairies by running the below command.

```
pip install "pyats[full]"
```

### Veryfying the installation

You can verify pyATS has been successfuly installed by running the below command. It should return the current pyATS version, with a similar output. This command also shows if there are any available package updates.

```
pyats version check

You are currently running pyATS version: 23.3
Python: 3.10.4 [64bit]

  Package                      Version
  ---------------------------- -------
  genie                        23.3   
  genie.libs.clean             23.3   
  
  ## output chunked for brevity ## 
  
  yang.connector               23.3   
```

### Updating pyATS

You can check if there is a newer pyATS version and update it with the below command. It should return a similar output.

```
pyats version update

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

## pyATS Testbed

This section will give general Tips and Tricks about pyATS testbeds.

### Default credentials and CLI	

When credentials or CLI options (ex: protocol, port number) are shared by many devices (ex: in a lab), it could be useful to set them as default. You should add them in your `testbed.yaml` file. You can still use specific information (i.e. non-default) for other devices in your testbed. 

In the below example, `xr1` and `xr2` will use all default information. `xr3` will use default `os` information, but specific `connections` and `credentials`.

```

testbed:										# Default information section
    credentials:
        default:                                # Default credentials
            username: cisco
            password: cisco
 	connections:								# Default CLI options
    	cli:
        	protocol: ssh
            port: 22
    os: iosxr									# Default OS, platform and type
    platform: iosxrv
    type: router
       
devices:
    xr1:                                    
        connections:
            cli:
                ip: 192.168.1.11
    xr2:                                    
        connections:
            cli:
                ip: 192.168.1.12 
    xr3:
    	connections:
            cli:
                ip: 192.168.1.12
                port: 2222
        credentials:
        	default:                                
            	username: antoine
            	password: cisco123
```

By default, the name of your device in the testbed should **EXACTLY** match the hostname of your device. This information is case sensitive.
{: .notice--info}

### Validating a Testbed file

You can verify that there is no typo or error in your testbed file by using the below command. If your testbed has error, it should look like the below output.

```
Loading testbed file: testbed.yaml
--------------------------------------------------------------------------------

YAML Lint Messages
------------------
  36:24     error    no new line character at the end of file  (new-line-at-end-of-file)```