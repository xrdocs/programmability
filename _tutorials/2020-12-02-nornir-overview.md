---
published: True
position: hidden
date: '2020-12-02 18:01 -0400'
title: Nornir Overview
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
This is the main file where you initialize Nornir with InitNornir function and provide the configuration file. In the next step, call run method and provide the tasks to be executed, here I provided napalm_get imported from nornir_napalm plugin. It executes the provided napalm getters over all the hosts provided in the inventory and return the results. 

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
 
 **Execute tasks without config file**
 
 In this case you just need to create a hosts file and nornir main file.
 
 **hosts.yaml**
 
 ```
 # hosts.yaml
---
rt1:
    hostname: pavarotti
    platform: 'iosxr'
    username: admin
    password: admin

rt2:
    hostname: placido
    platform: 'ios'
    username: admin
    password: admin
```

**nornir_main.py**

```
from nornir import InitNornir
from nornir_utils.plugins.functions import print_result
from nornir_napalm.plugins.tasks import napalm_cli

nr = InitNornir()

results = nr.run(
    task=napalm_cli, commands=["show interfaces"]
)

print_result(results)
```

Execute **nornir_main.py** file and retrieve the results.
```
python nornir_main.py
```

**output**

```
napalm_cli**********************************************************************
* rt1 ** changed : False *******************************************************
vvvv napalm_cli ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
{ 'show interfaces': 'Loopback0 is up, line protocol is up \n'
                     '  Interface state transitions: 1\n'
                     '  Hardware is Loopback interface(s)\n'
                     '  Description: PRIMARY ROUTER LOOPBACK-TEST-Replace\n'
                     '  Internet address is 172.16.255.2/32\n'
                     '  MTU 1500 bytes, BW 0 Kbit\n'
                     '     reliability Unknown, txload Unknown, rxload '
                     'Unknown\n'
                     '  Encapsulation Loopback,  loopback not set,\n'
                     '  Last link flapped 30w6d\n'
                     '  Last input Unknown, output Unknown\n'
                     '  Last clearing of "show interface" counters Unknown\n'
                     '  Input/output data rate is disabled.\n'
                     '\n'
                     'Null0 is up, line protocol is up \n'
                     '  Interface state transitions: 1\n'
                     '  Hardware is Null interface\n'
                     '  Internet address is Unknown\n'
                     '  MTU 1500 bytes, BW 0 Kbit\n'
                     '     reliability 255/255, txload Unknown, rxload '
                     'Unknown\n'
                     '  Encapsulation Null,  loopback not set,\n'
                     '  Last link flapped 30w6d\n'
                     '  Last input never, output never\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  5 minute input rate 0 bits/sec, 0 packets/sec\n'
                     '  5 minute output rate 0 bits/sec, 0 packets/sec\n'
                     '     0 packets input, 0 bytes, 0 total input drops\n'
                     '     0 drops for unrecognized upper-level protocol\n'
                     '     Received 0 broadcast packets, 0 multicast packets\n'
                     '     0 packets output, 0 bytes, 0 total output drops\n'
                     '     Output 0 broadcast packets, 0 multicast packets\n'
                     '\n'
                     'MgmtEth0/RP0/CPU0/0 is up, line protocol is up \n'
                     '  Interface state transitions: 1\n'
                     '  Hardware is Management Ethernet, address is '
                     '5254.00a4.1409 (bia 5254.00a4.1409)\n'
                     '  Description: *** MANAGEMENT INTERFACE ***\n'
                     '  Internet address is 10.30.110.86/23\n'
                     '  MTU 1514 bytes, BW 1000000 Kbit (Max: 1000000 Kbit)\n'
                     '     reliability 255/255, txload 0/255, rxload 0/255\n'
                     '  Encapsulation ARPA,\n'
                     '  Full-duplex, 1000Mb/s, EX, link type is '
                     'autonegotiation\n'
                     '  output flow control is off, input flow control is off\n'
                     '  loopback not set,\n'
                     '  Last link flapped 30w6d\n'
                     '  ARP type ARPA, ARP timeout 04:00:00\n'
                     '  Last input 00:00:00, output 00:00:00\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  5 minute input rate 0 bits/sec, 1 packets/sec\n'
                     '  5 minute output rate 1000 bits/sec, 0 packets/sec\n'
                     '     385136892 packets input, 32366312259 bytes, 1600594 '
                     'total input drops\n'
                     '     0 drops for unrecognized upper-level protocol\n'
                     '     Received 78089 broadcast packets, 9717802 multicast '
                     'packets\n'
                     '              0 runts, 0 giants, 0 throttles, 0 parity\n'
                     '     97 input errors, 0 CRC, 0 frame, 0 overrun, 0 '
                     'ignored, 0 abort\n'
                     '     373742434 packets output, 32010334625 bytes, 30518 '
                     'total output drops\n'
                     '     Output 0 broadcast packets, 0 multicast packets\n'
                     '     0 output errors, 0 underruns, 0 applique, 0 resets\n'
                     '     0 output buffer failures, 0 output buffers swapped '
                     'out\n'
                     '     1 carrier transitions\n'
                     '\n'
                     'GigabitEthernet0/0/0/0 is up, line protocol is up \n'
                     '  Interface state transitions: 3\n'
                     '  Hardware is GigabitEthernet, address is 5254.00ae.d917 '
                     '(bia 5254.00ae.d917)\n'
                     '  Description: CONNECTS TO LER1 (g0/0/0/1)\n'
                     '  Internet address is 172.16.0.1/31\n'
                     '  MTU 1514 bytes, BW 1000000 Kbit (Max: 1000000 Kbit)\n'
                     '     reliability 255/255, txload 0/255, rxload 0/255\n'
                     '  Encapsulation ARPA,\n'
                     '  Duplex unknown, 1000Mb/s, link type is force-up\n'
                     '  output flow control is off, input flow control is off\n'
                     '  loopback not set,\n'
                     '  Last link flapped 30w6d\n'
                     '  ARP type ARPA, ARP timeout 04:00:00\n'
                     '  Last input 8w3d, output 00:00:00\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  5 minute input rate 0 bits/sec, 0 packets/sec\n'
                     '  5 minute output rate 1000 bits/sec, 0 packets/sec\n'
                     '     228456 packets input, 212477801 bytes, 3 total '
                     'input drops\n'
                     '     0 drops for unrecognized upper-level protocol\n'
                     '     Received 0 broadcast packets, 0 multicast packets\n'
                     '              0 runts, 0 giants, 0 throttles, 0 parity\n'
                     '     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 '
                     'ignored, 0 abort\n'
                     '     2947992 packets output, 3359901634 bytes, 0 total '
                     'output drops\n'
                     '     Output 4 broadcast packets, 0 multicast packets\n'
                     '     0 output errors, 0 underruns, 0 applique, 0 resets\n'
                     '     0 output buffer failures, 0 output buffers swapped '
                     'out\n'
                     '     0 carrier transitions\n'
                     '\n'
                     'GigabitEthernet0/0/0/1 is up, line protocol is up \n'
                     '  Interface state transitions: 11\n'
                     '  Hardware is GigabitEthernet, address is 5254.009c.5644 '
                     '(bia 5254.009c.5644)\n'
                     '  Description: CONNECTS TO LSR1 (g0/0/0/0)\n'
                     '  Internet address is 172.16.0.2/31\n'
                     '  MTU 1514 bytes, BW 1000000 Kbit (Max: 1000000 Kbit)\n'
                     '     reliability 255/255, txload 0/255, rxload 0/255\n'
                     '  Encapsulation ARPA,\n'
                     '  Duplex unknown, 1000Mb/s, link type is force-up\n'
                     '  output flow control is off, input flow control is off\n'
                     '  loopback not set,\n'
                     '  Last link flapped 1d00h\n'
                     '  ARP type ARPA, ARP timeout 04:00:00\n'
                     '  Last input 00:00:00, output 00:00:00\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  5 minute input rate 1000 bits/sec, 0 packets/sec\n'
                     '  5 minute output rate 1000 bits/sec, 0 packets/sec\n'
                     '     2986024 packets input, 3290896911 bytes, 43 total '
                     'input drops\n'
                     '     0 drops for unrecognized upper-level protocol\n'
                     '     Received 5 broadcast packets, 0 multicast packets\n'
                     '              0 runts, 0 giants, 0 throttles, 0 parity\n'
                     '     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 '
                     'ignored, 0 abort\n'
                     '     3603284 packets output, 3384198762 bytes, 0 total '
                     'output drops\n'
                     '     Output 10 broadcast packets, 0 multicast packets\n'
                     '     0 output errors, 0 underruns, 0 applique, 0 resets\n'
                     '     0 output buffer failures, 0 output buffers swapped '
                     'out\n'
                     '     0 carrier transitions'}
^^^^ END napalm_cli ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* rt2 ** changed : False *******************************************************
vvvv napalm_cli ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
{ 'show interfaces': 'GigabitEthernet1 is up, line protocol is up \n'
                     '  Hardware is CSR vNIC, address is 5254.00df.1325 (bia '
                     '5254.00df.1325)\n'
                     '  Description: *** MANAGEMENT INTERFACE ***\n'
                     '  Internet address is 10.30.110.87/23\n'
                     '  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, \n'
                     '     reliability 255/255, txload 1/255, rxload 1/255\n'
                     '  Encapsulation ARPA, loopback not set\n'
                     '  Keepalive set (10 sec)\n'
                     '  Full Duplex, 1000Mbps, link type is auto, media type '
                     'is Virtual\n'
                     '  output flow-control is unsupported, input flow-control '
                     'is unsupported\n'
                     '  ARP type: ARPA, ARP Timeout 04:00:00\n'
                     '  Last input 00:00:00, output 00:00:00, output hang '
                     'never\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  Input queue: 3/375/0/0 (size/max/drops/flushes); Total '
                     'output drops: 0\n'
                     '  Queueing strategy: fifo\n'
                     '  Output queue: 0/40 (size/max)\n'
                     '  5 minute input rate 0 bits/sec, 0 packets/sec\n'
                     '  5 minute output rate 0 bits/sec, 0 packets/sec\n'
                     '     15884405591 packets input, 1445989033603 bytes, 0 '
                     'no buffer\n'
                     '     Received 0 broadcasts (0 IP multicasts)\n'
                     '     0 runts, 0 giants, 0 throttles \n'
                     '     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 '
                     'ignored\n'
                     '     0 watchdog, 0 multicast, 0 pause input\n'
                     '     1448247 packets output, 292267844 bytes, 0 '
                     'underruns\n'
                     '     0 output errors, 0 collisions, 0 interface resets\n'
                     '     848333 unknown protocol drops\n'
                     '     0 babbles, 0 late collision, 0 deferred\n'
                     '     0 lost carrier, 0 no carrier, 0 pause output\n'
                     '     0 output buffer failures, 0 output buffers swapped '
                     'out\n'
                     'GigabitEthernet2 is administratively down, line protocol '
                     'is down \n'
                     '  Hardware is CSR vNIC, address is 5254.00e7.29f9 (bia '
                     '5254.00e7.29f9)\n'
                     '  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, \n'
                     '     reliability 255/255, txload 1/255, rxload 1/255\n'
                     '  Encapsulation ARPA, loopback not set\n'
                     '  Keepalive set (10 sec)\n'
                     '  Full Duplex, 1000Mbps, link type is auto, media type '
                     'is Virtual\n'
                     '  output flow-control is unsupported, input flow-control '
                     'is unsupported\n'
                     '  ARP type: ARPA, ARP Timeout 04:00:00\n'
                     '  Last input never, output 1y20w, output hang never\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  Input queue: 0/375/0/0 (size/max/drops/flushes); Total '
                     'output drops: 0\n'
                     '  Queueing strategy: fifo\n'
                     '  Output queue: 0/40 (size/max)\n'
                     '  5 minute input rate 0 bits/sec, 0 packets/sec\n'
                     '  5 minute output rate 0 bits/sec, 0 packets/sec\n'
                     '     2 packets input, 214 bytes, 0 no buffer\n'
                     '     Received 0 broadcasts (0 IP multicasts)\n'
                     '     0 runts, 0 giants, 0 throttles \n'
                     '     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 '
                     'ignored\n'
                     '     0 watchdog, 0 multicast, 0 pause input\n'
                     '     6 packets output, 912 bytes, 0 underruns\n'
                     '     0 output errors, 0 collisions, 0 interface resets\n'
                     '     0 unknown protocol drops\n'
                     '     0 babbles, 0 late collision, 0 deferred\n'
                     '     1 lost carrier, 0 no carrier, 0 pause output\n'
                     '     0 output buffer failures, 0 output buffers swapped '
                     'out\n'
                     'GigabitEthernet3 is administratively down, line protocol '
                     'is down \n'
                     '  Hardware is CSR vNIC, address is 5254.005e.0f63 (bia '
                     '5254.005e.0f63)\n'
                     '  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, \n'
                     '     reliability 255/255, txload 1/255, rxload 1/255\n'
                     '  Encapsulation ARPA, loopback not set\n'
                     '  Keepalive set (10 sec)\n'
                     '  Full Duplex, 1000Mbps, link type is auto, media type '
                     'is Virtual\n'
                     '  output flow-control is unsupported, input flow-control '
                     'is unsupported\n'
                     '  ARP type: ARPA, ARP Timeout 04:00:00\n'
                     '  Last input 1y20w, output 1y20w, output hang never\n'
                     '  Last clearing of "show interface" counters never\n'
                     '  Input queue: 0/375/0/0 (size/max/drops/flushes); Total '
                     'output drops: 0\n'
                     '  Queueing strategy: fifo\n'
                     '  Output queue: 0/40 (size/max)\n'
                     '  5 minute input rate 0 bits/sec, 0 packets/sec\n'
                     '  5 minute output rate 0 bits/sec, 0 packets/sec\n'
                     '     17 packets input, 20690 bytes, 0 no buffer\n'
                     '     Received 0 broadcasts (0 IP multicasts)\n'
                     '     0 runts, 0 giants, 0 throttles \n'
                     '     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 '
                     'ignored\n'
                     '     0 watchdog, 0 multicast, 0 pause input\n'
                     '     6 packets output, 912 bytes, 0 underruns\n'
                     '     0 output errors, 0 collisions, 0 interface resets\n'
                     '     2 unknown protocol drops\n'
                     '     0 babbles, 0 late collision, 0 deferred\n'
                     '     1 lost carrier, 0 no carrier, 0 pause output\n'
                     '     0 output buffer failures, 0 output buffers swapped '
                     'out'}
^^^^ END napalm_cli ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
