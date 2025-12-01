---
published: false
date: '2025-12-01 11:39 +0100'
title: Radkit - Tips and tricks
author: Antoine Orsoni
tags:
  - cisco
  - Radkit
position: hidden
---
{% include toc icon="table" title="Table of Contents" %}

# Introduction

RADKit is a Software Development Kit (SDK): a set of ready-to-use tools and Python modules allowing efficient and scalable interactions with local or remote equipment. RADKit is available at no additional cost with your existing Support Contracts.

# Useful links

| What?         | Link                                             |
|---------------|--------------------------------------------------|
| Documentation | https://radkit.cisco.com/docs/index.html         |
| Download      | https://radkit.cisco.com/downloads/              |
| Installation  | https://radkit.cisco.com/docs/install/index.html |

# Tips and tricks

## Filtering the inventory

You can filter the inventory on any attribute (description, device_type...). Example for `device_type` attribute. On the below example, we will filter only the `IOS_XR` device type.

```python
service.inventory.filter("device_type", "IOS_XR")
```

