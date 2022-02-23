---
published: true
date: '2022-02-23 16:46 -0500'
title: Mixing EEM & Python on IOS-XR
position: hidden
author: Alain Nicolas
excerpt: Short description tbd
tags:
  - iosxr
  - Python
---
## Mixing EEM & Python on IOS-XR

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.

Embedded Event Manager scripts are used since ages to capture outputs, or make action right after an external trigger. Unfortunately, they are still limited to the good old TCL language. Nowadays, the majority of engineers prefer modern languages, like Python. So let's mix things up!

Examples were taken on IOS-XR 6.3.3.

{:refdef: style="text-align: center;"}
![EEMandPython](/images/eempython.jpg){:height="230px" width="460px"}
{: refdef}

### Python on XR?
eXR platforms are powered with Linux kernel, and includes Python. Let's start slowly, and call a show clock in python. To find the system version of your favorite IOS-XR regular command, you can use the command "describe" :

~~~
RP/0/RP0/CPU0:NCS#describe show clock
The command is defined in iosclock.parser


User needs ALL of the following taskids:

        basic-services (READ)

It will take the following actions:
Thu Jan 28 14:56:01.414 UTC
  Spawn the process:
        iosclock -e 0x0

~~~
And then, in running context :

~~~
RP/0/RP0/CPU0:NCS#run
[xr-vm_node0_RP0_CPU0:~]$iosclock -e 0x0
14:59:07.834 UTC Thu Jan 28 2021
~~~

Or in python :
~~~
[xr-vm_node0_RP0_CPU0:~]$python
Python 2.7.3 (default, Nov 16 2019, 21:56:43)
[GCC 4.9.1] on linux2
>>> from subprocess import call
>>> call(['iosclock', '-e', '0x0'])
15:06:47.103 UTC Thu Jan 28 2021
0
~~~
For some commands we may have a lot of options and arguments, so these ones should be passed as list.

Directly from a script :

~~~
[xr-vm_node0_RP0_CPU0:~]$cat test.py
from subprocess import call
output = call(['iosclock', '-e', '0x0'])
print output
[xr-vm_node0_RP0_CPU0:~]$exit
logout

RP/0/RP0/CPU0:par-th2-pb1-nc5#run python test.py
Thu Jan 28 15:09:36.308 UTC
15:09:36.588 UTC Thu Jan 28 2021
0
~~~

## The good old EEM

Small refresh on how to do an EEM script. Don't forget your aaa configuration! You will also need a user able to execute a script. In a nutshell :

~~~
event manager environment _cron_entry */1 * * * *
event manager directory user policy disk0:/script
event manager policy eem_throttle.tcl username cisco persist-time 3600 type user
!
aaa authorization eventmanager default local
~~~

What are those commands?
+ event manager **environment** is the variable that our script will call. It helps us manage the cron value we are going to use dynamically.
+ event manager **directory** is where all the file related to our EEM are going to be located.
+ event manager **policy** is the declaration of our script, where cisco is the user.

We are now ready to launch eem_throttle.tcl.
~~~bash
::cisco::eem::event_register_timer cron name crontimer2 cron_entry $_cron_entry maxrun 240
namespace import ::cisco::eem::*
namespace import ::cisco::lib::*

if [catch {cli_open} result] {
 error $result $errorInfo
  } else {
    array set cli1 $result
    }

if [catch {cli_exec $cli1(fd) "run python /disk0\:/throttle.py"} result] {
 error $result $errorInfo
}

if [catch {cli_close $cli1(fd) $cli1(tty_id)} result] {
 error $result $errorInfo
}
~~~

Pretty simple : this tcl script will open the CLI, execute throttle.py, and close the CLI.

## A touch of python

Here is a python script used in a real production environment. Basically, it checked if we have any BGP session throttling on RR.
The standard command output:
~~~
RP/0/RP0/CPU0:RR#sh bgp update out sub-group brief
  SG             UG      Status    Limit      OutQ       SG-R Nbrs Version    (PendVersion)
  0.4            0.8     Throttled 335544     288672     0    20   1993968    (1993881)
~~~

And here is a small python script to simplify network operator's life. It generates a log message each time our script catch a throttling BGP session.


~~~python
import subprocess
from subprocess import call
import re

cmd_list = ['bgp_show -updgen -updgencmd sgrp -V default -A 0x1 -W 0x1 -brief -instance default']

for item in cmd_list:
    args = item.split()
    p = subprocess.Popen(args, stdout=subprocess.PIPE)
    (output, Err) = p.communicate()
    p_status = p.wait()
    if re.search("Throttled", output):
      call(['logger','TESTING - This device has a BGP session Throttling']

~~~

### Conclusion

To sum up, we used python to get usual CLI output, EEM Script to schedule when we want to play our script, and once again, python to react to the output. It is now up to you to use python on your XR device to solve your real life use cases!
