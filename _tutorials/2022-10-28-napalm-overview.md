---
published: True
position: hidden
date: '2022-10-28 10:00 -0400'
title: Napalm Overview
excerpt: Napalm is a python library which grabbed the attention of network engineers. Learn more about it and take a look how you could manage cisco devices with it.
author: Neelima Parakala
---
{% include toc icon="table" title="Napalm Overview" %}



## What is Napalm ?


## Why Napalm ?

       
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
virtualenv napalm_venv
```
Now activate the virtual environment.
```
source ~/napalm_venv/bin/activate
```
### Install Napalm and its plugins

In the “napalm_venv” virtual environment, install **napalm**. At the moment of producing this tutorial, latest version of napalm is 3.3.1.
```
pip install napalm
```
Once you have all the required packages installed, go ahead and write the code to retrieve, configure or validate device data.

### Write a few lines of code to automate your network

#### Example 1

```
```

#### Example 2

```
```

#### Example 3

```
```

#### Example 4

```
```

#### Example 5

```
```


## What are the flavours of Napalm ?


# Conclusion
Napalm is a vendor neutral, cross-platform open-source project that provides a unified API to automate the mnagement of network devices. Being an open-sourced project written in python makes it easy for the user to debug and troubleshoot. Above all, it's time-efficient, free, and easy to use. Write simple lines of python code to execute your network tasks on a lot of your network devices quickly!! 
*Stay tuned to learn about another network automation tool in our next post.*
*Please do comment below, your questions, and what you would like to learn about network automation!!*

# Resources

- [Napalm Documentation](https://napalm.readthedocs.io/en/latest/)
- [Napalm GitHub repository](https://github.com/napalm-automation/napalm)
