+++
title = "New development and test environment installed"
slug = "servers01"
date = "2013-03-27"
description = ""
aliases = ["/post1.html"]
keywords = ["servers", "development environment", "linux", "proxmox"]
categories = [""]
tags = ["servers", "development environment", "linux", "proxmox"]
+++


author: Rene Cremers


Today some new servers arrived at Code Yellow HQ! The boxes were obsolete at a data center of one of our customers
so we got 4 nice pieces of hardware to set up a new development environment. 

<img alt="servers 1" title="Servers" src="/images/servers1.jpg" style="width: 100%;" />

The new development environment consists of 4 dedicated boxes, all loaded into a [Proxmox](http://proxmox.com/products/proxmox-ve) cluster. 
For easy deployment we use a development image of Ubuntu server 12.04 with some tweaks.

For each project it is easy to setup a new VPS, load the code from our repositories hosted at [Bitbucket](http://bitbucket.org) and get on with developing and testing.

<img alt="servers 2" title="Servers mounted" src="/images/servers2.jpg" style="width: 100%;" />
    
