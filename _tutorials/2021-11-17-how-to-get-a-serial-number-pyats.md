---
published: true
date: '2021-11-17 16:21 +0100'
title: How to get a serial number - pyATS
author: Antoine Orsoni
excerpt: How to get a serial number on an IOS XR device using pyATS?
tags:
  - Programmability
  - Python
  - pyATS
  - Automation
position: hidden
---
{% include toc icon="table" title="Table of Contents" %}

Recently, I got a query from a Customer: how could I easily collect my device(s) serial number? At first, he question sounded silly: you could just do `show inventory all` on any IOS XR platform to get the platform serial number. What if you need to do it 100 times per day? What if you need to do it on 100 devices at once? The goal of this new series of article is to explain different ways to collect a serial number on a device. If you can do it with a serial number, you can do it with anything else!