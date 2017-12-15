+++
title = "The ideal development image"
slug = "the-ideal-development-image"
date = "2013-04-11"
description = ""
aliases = ["/the-ideal-development-image.html"]
keywords = ["proxmox", "linux", "development image"]
categories = [""]
tags = ["proxmox", "linux", "development image"]
+++

author: Rene Cremers


To take full advantage of the Proxmox development servers we decided to make a fresh development image with all
settings preset, so when you start a project, you only have to load a fresh image into a Virtual Private Server
and go off developing great things. 

In this post we would like to share with you the default settings and configuration of the setup:

- Ubuntu server 12.04 (minimalistic install)
- MySQL
- [DVCS](http://hginit.com): Mercurial and Git
- PHP 5.4
- Apache 2
- Curl
- PhpMyAdmin
- Python 2.7 
- Htop
- Btsync
- Node Version Manager

Of course the standard settings for the firewall, logwatch user management and ssh are also part of the image. 

Some choices may seem a bit odd, but some things are nice to have without the extra apt-get hassle. However: in a 
production environment the list would be very different indeed.
