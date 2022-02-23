---
published: True
position: hidden
date: '2021-02-23 10:00 -0400'
title: Scrapli Overview
excerpt: Scrapli is a python library which grabbed the attention of network engineers. Learn more about it and take a look how you could manage Cisco devices with it.
author: Neelima Parakala
---
{% include toc icon="table" title="Scrapli Overview" %}

You no more want to use command line interface, to manage network device configuration? Then go ahead and scrape CLI using scrapli !!!!

## What is Scrapli?

Scrapli is a python library focused on connecting to devices, specifically network devices (routers/switches/firewalls/etc.) via SSH or Telnet. 
The name scrapli is just "scrape cli" (as in screen scrape) squished together! scrapli's goal is to be as fast and flexible as possible, 
while providing a thoroughly tested, well typed, well documented, simple API that supports both synchronous and asynchronous usage.

## Why Scrapli?

- Easy: It's easy to get going with scrapli -- check out the documentation and example links above, and you'll be connecting to devices in no time.
- Fast: Do you like to go fast? Of course you do! All of scrapli is built with speed in mind, but if you really feel the need for speed, check out the ssh2 transport plugin to take it to the next level!
- Great Developer Experience: scrapli has great editor support thanks to being fully typed; that plus thorough docs make developing with scrapli a breeze.
- Well Tested: Perhaps out of paranoia, but regardless of the reason, scrapli has lots of tests! Unit tests cover the basics, regularly ran functional tests connect to virtual routers to ensure that everything works IRL!
- Pluggable: scrapli provides a pluggable transport system -- don't like the currently available transports, simply extend the base classes and add your own! Need additional device support? Create a simple "platform" in scrapli_community to easily add new device support!
- But wait, there's more!: Have NETCONF devices in your environment, but love the speed and simplicity of scrapli? You're in luck! Check out scrapli_netconf!
- Concurrency on Easy Mode: Nornir's scrapli plugin gives you all the normal benefits of scrapli plus all the great features of Nornir.
    

## How does it work?
