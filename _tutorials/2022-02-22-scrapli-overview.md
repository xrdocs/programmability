---
published: True
position: hidden
date: '2022-03-11 10:00 -0400'
title: Scrapli Overview
excerpt: Scrapli is a python library which grabbed the attention of network engineers. Learn more about it and take a look how you could manage Cisco devices with it.
author: Neelima Parakala
---
{% include toc icon="table" title="Scrapli Overview" %}

Are you a pro in CLI commands but tired of manually logging in to each network device and managing the network device configuration and operational data using the command line interface ? Then its time for you to explore Scrapli and automate CLI scraping !!!!

## What is Scrapli?

Scrapli is a python library that helps you to connect multiple network devices via SSH or Telnet, synchronously or asynchronously for screen scraping. Wondering what is screen scraping? well, it is the process of collecting screen display data from the network device and dumping it on to another screen or file. 

## Why Scrapli?
- Firstly, it is an open-source project, hence it is free to use and you can add new device/transport support to it based on your requirement.
- It is fast, flexible and well tested
- It has an active community and well-maintained documentation.
- As scrapli is completely written in python, it is easy to install, write code, integrate with any other python frameworks, troubleshoot and debug the issues using python debug tools
- It provides asynchronous and synchronous support for both ssh and telnet transport protocols
- It provides multi-vendor support (Arista EOS, Cisco NX-OS, Cisco IOS-XE, Cisco IOS-XR, Juniper JunOS)
         
## How does it work?

### Setup virtual environment(Optional)

If you would like to isolate the dependencies of scrapli from the system, you can create a python virtual environment.

Install virtualenv package using pip. pip is a package installer for Python. Here I am using Python 3.8.5 version.
```
pip install virtualenv
```
Using virtualenv package, create a python virtual environment.
Refer [creation of python virtual environments](https://docs.python.org/3/library/venv.html).
```
virtualenv scrapli_venv
```
Now activate the virtual environment.
```
source ~/scrapli_venv/bin/activate
```
### Install Scrapli and its plugins

In the “scrapli_venv” virtual environment, install **scrapli**. At the moment of producing this tutorial, latest version of scrapli is 2022.1.30.post1.
```
pip install scrapli
```
Once you have all the required packages installed, go ahead and write the code to retrieve, configure or validate device data.

### Write a few lines of code to automate your network

#### Example 1
```
"""examples.basic_usage.scrapli_driver"""
from scrapli import Driver

MY_DEVICE = {
    "host": "10.10.100.12",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
}

def main():
    """Example demonstrating using the `Driver` driver directly"""
    # the `Driver` driver is only for use if you *really* want to manually handle the channel
    # input/output if your device type is supported by a core platform you should probably use that,
    # otherwise check out the `GenericDriver` before diving into `Driver` as a last resort!
    conn = Driver(**MY_DEVICE)
    conn.open()

    print(conn.channel.get_prompt())
    print(conn.channel.send_input("show run | i hostname")[1])

    # paging is NOT disabled w/ Scrape driver!
    conn.channel.send_input("terminal length 0")
    print(conn.channel.send_input("show run")[1])
    conn.close()

    # Context manager is a great way to use scrapli:
    with Driver(**MY_DEVICE) as conn:
        result = conn.channel.send_input("show run | i hostname")
    print(result[1])


if __name__ == "__main__":
    main()

```
#### Example 2

```
"""examples.basic_usage.generic_driver"""
from scrapli.driver import GenericDriver

MY_DEVICE = {
    "host": "10.10.100.12",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
}

def main():
    """Simple example of connecting to an IOSXEDevice with the GenericDriver"""
    # the `GenericDriver` is a good place to start if your platform is not supported by a "core"
    #  platform drivers
    conn = GenericDriver(**MY_DEVICE)
    conn.open()

    print(conn.channel.get_prompt())
    print(conn.send_command("show run | i hostname").result)

    # IMPORTANT: paging is NOT disabled w/ GenericDriver driver!
    conn.send_command("terminal length 0")
    print(conn.send_command("show run").result)
    conn.close()

    # Context manager is a great way to use scrapli, it will auto open/close the connection for you:
    with GenericDriver(**MY_DEVICE) as conn:
        result = conn.send_command("show run | i hostname")
    print(result.result)


if __name__ == "__main__":
    main()
```
#### Example 3

```
"""examples.basic_usage.iosxe_driver"""
from scrapli.driver.core import IOSXRDriver

MY_DEVICE = {
    "host": "10.10.100.12",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
}

def main():
    """Simple example of connecting to an IOSXEDevice with the IOSXEDriver"""
    with IOSXRDriver(**MY_DEVICE) as conn:
        # Platform drivers will auto-magically handle disabling paging for you
        result = conn.send_command("show run")

    print(result.result)


if __name__ == "__main__":
    main()
```

### Example 4

```
"""examples.async_usage.async_iosxe_driver"""
import asyncio

from scrapli.driver.core import AsyncIOSXRDriver

MY_DEVICE = {
    "host": "10.10.100.12",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
    "transport": "asyncssh",
}


async def main():
    """Simple example of connecting to an IOSXRDevice with the AsyncIOSXRDriver"""
    async with AsyncIOSXRDriver(**MY_DEVICE) as conn:
        # Platform drivers will auto-magically handle disabling paging for you
        result = await conn.send_command("show run")

    print(result.result)


if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(main())
```

### Example 5

```
"""examples.async_usage.async_multiple_connections"""
import asyncio

from scrapli.driver.core import AsyncIOSXRDriver

IOSXR_DEVICE1 = {
    "host": "10.10.100.12",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
    "transport": "asyncssh",
    "driver": AsyncIOSXRDriver,
}

IOSXR_DEVICE2 = {
    "host": "10.10.100.13",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
    "transport": "asyncssh",
    "driver": AsyncIOSXRDriver,
}

DEVICES = [IOSXR_DEVICE1, IOSXR_DEVICE2]


async def gather_version(device):
    """Simple function to open a connection and get some data"""
    driver = device.pop("driver")
    conn = driver(**device)
    await conn.open()
    prompt_result = await conn.get_prompt()
    version_result = await conn.send_command("show version")
    await conn.close()
    return prompt_result, version_result


async def main():
    """Function to gather coroutines, await them and print results"""
    coroutines = [gather_version(device) for device in DEVICES]
    results = await asyncio.gather(*coroutines)
    for result in results:
        print(f"device prompt: {result[0]}")
        print(f"device show version: {result[1].result}")


if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(main())
```
### Example 6

```
"""examples.logging.basic_logging"""
import logging

from scrapli.driver.core import IOSXRDriver

# set the name for the logfile and the logging level... thats about it for bare minimum!
logging.basicConfig(filename="scrapli.log", level=logging.DEBUG)

MY_DEVICE = {
    "host": "10.10.100.12",
    "auth_username": "scrapli",
    "auth_password": "scrapli",
    "auth_strict_key": False,
}


def main():
    """Example demonstrating basic logging with scrapli"""
    conn = IOSXRDriver(**MY_DEVICE)
    conn.open()
    print(conn.get_prompt())
    print(conn.send_command("show run | i hostname").result)


if __name__ == "__main__":
    main()
```

## What are the flavours of Scrapli ?
- scrapli_netconf 
- nornir_scrapli
- scrapli_community
- scrapli_replay
- scrapli_cfg
- scrapli_stubs
- scrapli_asynchssh
- scrapli_ssh2
- scrapli_paramiko
- scrapli_telnetlib3
- scrapligo 
- srlinux-scrapli 
- scrapli[ttp]
- scrapli[genie]
- scrapli[textfsm]
- scrapli-scp

## Conclusion
