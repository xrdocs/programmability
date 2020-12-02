---
published: True
position: hidden
date: '2020-12-02 18:01 -0400'
title: Nornir Overview
position: top
author: Neelima Parakala
---
**Are you looking for a flexible, scalable and efficient network automation framework, where all tasks are written in Python programming language? If yes, then you are at the right place! And it is NORNIR !!!**

**Ok! Now what is NORNIR?**  
Nornir is a multi-threaded network automation framework which abstracts inventory and task execution. It helps to automate your network tasks efficiently. You can execute the tasks like configuring the devices, validating the operational data and enabling the services on the provided hosts which are part of the inventory. As it is multithreaded, it allows you to manage configuration of multiple network devices concurrently. It is an open source project, completely written in python and easy to use. You should write a simple python code to make use of NORNIR features.

**Well, why NORNIR?**
- Firstly, it  is an open-source project, hence it is free to use and you can develop features on top of Nornir framework based on your requirement.
- It has an active community and well maintained documentation.
- As Nornir is completely written in python, it is easy to
	- install
	- use
	- integrate with any other python frameworks (Flask, Django, pytest)
	- troubleshoot and debug the issues using python debug tools
- It reuses existing python libraries like netmiko and napalm to connect and manage the devices.
- Use of multithreads, greatly optimizes execution time of the tasks.
- You can effectively manage the hosts and groups separately part of the inventory

**So, how does it work?**
If you would like to isolate the dependencies of Nornir from the system, you can create a python virtual environment.

Install virtualenv package using pip. pip is a package installer for python.
```
pip install virtualenv
```
Using virtualenv package create a python virtual environment
```
virtualenv nornir_venv
```
Now activate the virtual environment
```
source ~/nornir_venv/bin/activate
```
In “nornir_venv” virtual environment, install **nornir**.
```
pip install nornir
```
Install Nornir plugin **nornir-napalm**.  It provides napalm connections through which you connect to the device and tasks like napalm_cli, napalm_configure, napalm_get, napalm_ping and napalm_validate.
```
pip install nornir-napalm
```
Install Nornir plugin **nornir-utils**.  It provides inventory, functions, processors and tasks.
```
pip install nornir-utils
```
Once you have all the required packages installed, go ahead and write the code to retrieve, configure or validate device data.

Create a directory called inventory and in that create the inventory files hosts.yaml, groups.yaml and defaults.yaml

**hosts.yaml**
```
# hosts.yaml
---
rt1:
   hostname: 171.190.10.64
   groups:
        - iosxr
rt2:
   hostname: 10.30.11.170
   groups:
        - ios
```
**groups.yaml**
```
# groups.yaml
---
iosxr:
    platform: 'iosxr'
ios:
    platform: 'ios'
```

**defaults.yaml**
```
# defaults.yaml
username: admin
password: admin
```
Now come out of the inventory directory and create a config file which provides information about the inventory and runner to the main file.

**config.yaml**
```
# config.yaml
---
inventory:
       plugin: SimpleInventory
       options:
            host_file: 'inventory/hosts.yaml'
            group_file: 'inventory/groups.yaml'
            defaults_file: 'inventory/defaults.yaml'
runner:
       plugin: threaded
       options:
            num_workers: 2
```
Config file provides inventory and task parallelization information to the main file. Nornir will use a ``different thread for each host`` to concurrently execute the tasks of the hosts. You can provide the number of threads to be used by your code in ```num_workers``` option of runner plugin. If ``num_workers == 1``, tasks of the hosts are executed one after the other in a simple loop. This case helps to troubleshoot or debug the issues. Generally you can provide a number greater than 1 to ``num_workers`` else it defaults to 20. In my case I am assigning value 2 to ``num_workers``, as I am dealing with two hosts.

**nornir_main.py**
```
from nornir import InitNornir
from nornir_utils.plugins.functions import print_result
from nornir_napalm.plugins.tasks import napalm_get

nr = InitNornir(
    config_file="config.yaml", dry_run=True
)

results = nr.run(
    task=napalm_get, getters=["facts"]
)

print_result(results)
```

Execute **nornir_main.py** file and retrieve the results.
```
python nornir_main.py
```
**Output**
```
napalm_get**********************************************************************
* rt1 ** changed : False *******************************************************
vvvv napalm_get ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
{ 'facts': { 'fqdn': 'pavarotti',
	     'hostname': 'pavarotti',
             'interface_list': [ 'GigabitEthernet0/0/0/0',
                                 'GigabitEthernet0/0/0/1',
                                 'Loopback0',
                                 'MgmtEth0/RP0/CPU0/0',
				 'Null0'],
             'model': 'R-IOSXRV9000-CC',
             'os_version': '6.5.3',
             'serial_number': 'E3FDA081DAC',
             'uptime': 18033322,
             'vendor': 'Cisco'}}
^^^^ END napalm_get ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* rt2 ** changed : False *******************************************************
vvvv napalm_get ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
{ 'facts': { 'fqdn': 'placido.placido.local',
             'hostname': 'placido',
             'interface_list': [ 'GigabitEthernet1',
				 'GigabitEthernet2',
				 'GigabitEthernet3'],
	     'model': 'CSR1000V',
	     'os_version': 'Virtual XE Software '
			   '(X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 16.9.3, '
	                   'RELEASE SOFTWARE (fc2)',
	     'serial_number': '9NSHRXZD4TZ',
             'uptime': 43016280,
             'vendor': 'Cisco'}}
^^^^ END napalm_get ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
 
