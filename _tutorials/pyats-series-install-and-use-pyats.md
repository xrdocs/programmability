---
published: true
date: '2021-01-07 22:40 +0100'
title: pyATS series - Install and use pyATS
author: Antoine Orsoni
excerpt: Installation of pyATS and collecting your first CLI output
tags:
  - iosxr
  - cisco
  - pyATS
position: hidden
---

# pyATS overview

Ever dreamed of a test framework that could be used across multiple platforms, OS and vendors, which could do regression, sanity and feature testing; already used by thousands of engineers and developers worldwide? Guess what, it exists, it’s free, and you can start using it right now!

pyATS was first created as an internal project, to ease the validation of two OS versions. It has been made public in 2017 through Cisco Devnet.

This blog post will be the first one of a series on pyATS. Today, we will explain what’s pyATS, install pyATS and cover a basic use case (getting a CLI output from a XR device). More use cases are going to be covered in the next posts. 

The code used for each blog post can be found [here](https://github.com/AntoineOrsoni/xrdocs-how-to-pyats). This link will include the code for all posts. Today’s part will refer to the `0_get_cli_show` folder of the repo.

## Building blocks

pyATS is made of three main building blocks:
- **pyATS**, the core block of this ecosystem. It’s a Python framework which leverages multiple Python libraries such as [Unicon](https://pypi.org/project/unicon/), providing a simplified connection experience to network devices. It supports CLI, NETCONF and RESTCONF. It enables network engineers and developers to start with small and simple test cases 
- **pyATS libraries** (also known as Genie) which provides everything you need for network testing such as parsers, triggers and APIs. 
- **XPRESSO**, the pyATS Web UI Dashboard.

![pyATS ecosystem]({{site.baseurl}}/https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/_images/layers.png){: .align-center}

You can read more about pyATS ecosystem in the [official documentation](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/intro/introduction.html).

## Supported OS

This solution is built and thought from the ground up to be an agnostic ecosystem. As of January 2021, it comes out of the box with libraries for the below OS:

- IOS,
- IOS XE,
- IOS XR,
- NXOS,
- ASA,
- Linux,
- JUNOS,
- SROS,
- BIGIP,
- Viptela OS,
- DNA Center.

# Getting your hands dirty

Enough talking, how do YOU start using pyATS?

## pyATS requirements

Full requirements can be found in the [official documentation](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/prereqs/prerequisites.html).

### Hardware

pyATS is lightweight and scalable. As per the documentation, you need:
- 1GB of RAM,
- 1 vCPU.

### Operating System
 
It rusn in a Linux and Linux-like environments, such as Ubuntu, CentOS, Fedora and macOS. The pyATS ecosystem does **not** support Windows.

### Python version

As of January 2021, it requires Python version between 3.5 and 3.8. Version 3.9 is NOT yet supported.

![pyats_ready.png]({{site.baseurl}}/images/pyats_ready.png){: .align-center}

## pyATS installation

pyATS ecosystem can be installed in two ways: in a docker container or in a virtual environment. Today, we will focus on the second option. Remember, you need a Python version between 3.5 and 3.8 to use pyATS.

Full installation documentation for Docker and Virtual Environment can be found [here](https://pubhub.devnetcloud.com/media/pyats-getting-started/docs/install/installpyATS.html).

Let’s start! Open a bash terminal and run the below three commands. It will:
- Create a virtual environment.
- Activate the virtual environment.
- Install pyATS and its dependencies.

**From your bash terminal**
<div class="highlighter-rouge">
<pre class="highlight">
<code>
python -m venv venv
source venv/bin/activate
pip install pyats
</code>
</pre>
</div>

## Your first pyATS use case: getting a CLI output from a device

In this first use case, we are going to see step by step how we can get a **simple CLI output** (`show ip interface brief`) from a IOS XR device. This first use case do **not** demonstrates the full power of pyATS but should be a good example to cover the basics.

In order for everyone to be able to run the code, we will use the [IOS XR always-on sandbox on Cisco Devnet](https://devnetsandbox.cisco.com/RM/Diagram/Index/e83cfd31-ade3-4e15-91d6-3118b867a0dd?diagramType=Topology). Feel free to adapt the code to use your own device(s). Below the sandbox information.

| Key               	| Value                    	|
|-------------------	|--------------------------	|
| IOS XRv 9000 host 	| sbx-iosxr-mgmt.cisco.com 	|
|     SSH Port      	|     8181                 	|
|     Username      	|     admin                	|
|     Password      	|     C1sco12345           	|

## Building a testbed

The simplest way to connect to a device is through a pyATS testbed file, written in YAML. This information will be used by **Unicon** to connect to the device and send/get the requested commands.

**testbed.yaml**
<head>
  <title></title>
  <meta http-equiv="content-type" content="text/html; charset=utf-8">
  <style type="text/css">
/*
generated by Pygments <https://pygments.org/>
Copyright 2006-2020 by the Pygments team.
Licensed under the BSD license, see LICENSE for details.
*/
pre { line-height: 125%; margin: 0; }
td.linenos pre { color: #000000; background-color: #f0f0f0; padding: 0 5px 0 5px; }
span.linenos { color: #000000; background-color: #f0f0f0; padding: 0 5px 0 5px; }
td.linenos pre.special { color: #000000; background-color: #ffffc0; padding: 0 5px 0 5px; }
span.linenos.special { color: #000000; background-color: #ffffc0; padding: 0 5px 0 5px; }
body .hll { background-color: #ffffcc }
body { background: #f8f8f8; }
body .c { color: #008800; font-style: italic } /* Comment */
body .err { border: 1px solid #FF0000 } /* Error */
body .k { color: #AA22FF; font-weight: bold } /* Keyword */
body .o { color: #666666 } /* Operator */
body .ch { color: #008800; font-style: italic } /* Comment.Hashbang */
body .cm { color: #008800; font-style: italic } /* Comment.Multiline */
body .cp { color: #008800 } /* Comment.Preproc */
body .cpf { color: #008800; font-style: italic } /* Comment.PreprocFile */
body .c1 { color: #008800; font-style: italic } /* Comment.Single */
body .cs { color: #008800; font-weight: bold } /* Comment.Special */
body .gd { color: #A00000 } /* Generic.Deleted */
body .ge { font-style: italic } /* Generic.Emph */
body .gr { color: #FF0000 } /* Generic.Error */
body .gh { color: #000080; font-weight: bold } /* Generic.Heading */
body .gi { color: #00A000 } /* Generic.Inserted */
body .go { color: #888888 } /* Generic.Output */
body .gp { color: #000080; font-weight: bold } /* Generic.Prompt */
body .gs { font-weight: bold } /* Generic.Strong */
body .gu { color: #800080; font-weight: bold } /* Generic.Subheading */
body .gt { color: #0044DD } /* Generic.Traceback */
body .kc { color: #AA22FF; font-weight: bold } /* Keyword.Constant */
body .kd { color: #AA22FF; font-weight: bold } /* Keyword.Declaration */
body .kn { color: #AA22FF; font-weight: bold } /* Keyword.Namespace */
body .kp { color: #AA22FF } /* Keyword.Pseudo */
body .kr { color: #AA22FF; font-weight: bold } /* Keyword.Reserved */
body .kt { color: #00BB00; font-weight: bold } /* Keyword.Type */
body .m { color: #666666 } /* Literal.Number */
body .s { color: #BB4444 } /* Literal.String */
body .na { color: #BB4444 } /* Name.Attribute */
body .nb { color: #AA22FF } /* Name.Builtin */
body .nc { color: #0000FF } /* Name.Class */
body .no { color: #880000 } /* Name.Constant */
body .nd { color: #AA22FF } /* Name.Decorator */
body .ni { color: #999999; font-weight: bold } /* Name.Entity */
body .ne { color: #D2413A; font-weight: bold } /* Name.Exception */
body .nf { color: #00A000 } /* Name.Function */
body .nl { color: #A0A000 } /* Name.Label */
body .nn { color: #0000FF; font-weight: bold } /* Name.Namespace */
body .nt { color: #008000; font-weight: bold } /* Name.Tag */
body .nv { color: #B8860B } /* Name.Variable */
body .ow { color: #AA22FF; font-weight: bold } /* Operator.Word */
body .w { color: #bbbbbb } /* Text.Whitespace */
body .mb { color: #666666 } /* Literal.Number.Bin */
body .mf { color: #666666 } /* Literal.Number.Float */
body .mh { color: #666666 } /* Literal.Number.Hex */
body .mi { color: #666666 } /* Literal.Number.Integer */
body .mo { color: #666666 } /* Literal.Number.Oct */
body .sa { color: #BB4444 } /* Literal.String.Affix */
body .sb { color: #BB4444 } /* Literal.String.Backtick */
body .sc { color: #BB4444 } /* Literal.String.Char */
body .dl { color: #BB4444 } /* Literal.String.Delimiter */
body .sd { color: #BB4444; font-style: italic } /* Literal.String.Doc */
body .s2 { color: #BB4444 } /* Literal.String.Double */
body .se { color: #BB6622; font-weight: bold } /* Literal.String.Escape */
body .sh { color: #BB4444 } /* Literal.String.Heredoc */
body .si { color: #BB6688; font-weight: bold } /* Literal.String.Interpol */
body .sx { color: #008000 } /* Literal.String.Other */
body .sr { color: #BB6688 } /* Literal.String.Regex */
body .s1 { color: #BB4444 } /* Literal.String.Single */
body .ss { color: #B8860B } /* Literal.String.Symbol */
body .bp { color: #AA22FF } /* Name.Builtin.Pseudo */
body .fm { color: #00A000 } /* Name.Function.Magic */
body .vc { color: #B8860B } /* Name.Variable.Class */
body .vg { color: #B8860B } /* Name.Variable.Global */
body .vi { color: #B8860B } /* Name.Variable.Instance */
body .vm { color: #B8860B } /* Name.Variable.Magic */
body .il { color: #666666 } /* Literal.Number.Integer.Long */

  </style>
</head>
<body>
<h2></h2>

<div class="highlight"><pre><span></span><span class="kn">from</span> <span class="nn">pyats.topology</span> <span class="kn">import</span> <span class="n">loader</span>

<span class="c1"># Step 0: load the testbed</span>
<span class="n">testbed</span> <span class="o">=</span> <span class="n">loader</span><span class="o">.</span><span class="n">load</span><span class="p">(</span><span class="sa">f</span><span class="s1">&#39;./testbed.yaml&#39;</span><span class="p">)</span>

<span class="c1"># Step 1: testbed is a dictionnary. Extract the device iosxr1</span>
<span class="n">iosxr1</span> <span class="o">=</span> <span class="n">testbed</span><span class="o">.</span><span class="n">devices</span><span class="p">[</span><span class="s2">&quot;iosxr1&quot;</span><span class="p">]</span>

<span class="c1"># Step 2: Connect to the device</span>
<span class="n">iosxr1</span><span class="o">.</span><span class="n">connect</span><span class="p">(</span><span class="n">init_exec_commands</span><span class="o">=</span><span class="p">[],</span> <span class="n">init_config_commands</span><span class="o">=</span><span class="p">[],</span> <span class="n">log_stdout</span><span class="o">=</span><span class="kc">False</span><span class="p">)</span>

<span class="c1"># Step 3: saving the `show ip interface brief` output in a variable</span>
<span class="n">show_interface</span> <span class="o">=</span> <span class="n">iosxr1</span><span class="o">.</span><span class="n">execute</span><span class="p">(</span><span class="s1">&#39;show ip interface brief&#39;</span><span class="p">)</span>

<span class="c1"># Step 4: pritting the `show interface brief` output</span>
<span class="nb">print</span><span class="p">(</span><span class="n">show_interface</span><span class="p">)</span>

<span class="c1"># Step 5: disconnect from the device</span>
<span class="n">iosxr1</span><span class="o">.</span><span class="n">disconnect</span><span class="p">()</span>
</pre></div>
</body>