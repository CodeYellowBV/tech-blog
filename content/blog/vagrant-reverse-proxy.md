+++
title = "Accessing your Vagrant boxes through a reverse proxy"
author = "Peter Bex"
slug = "vagrant-reverse-proxy"
date = "2016-03-16"
description = ""
aliases = ["/vagrant-reverse-proxy.html"]
keywords = ["Tools", "DevOps", "Vagrant", "Ruby"]
tags = ["Tools", "DevOps", "Vagrant", "Ruby"]
+++

The past year we've been making increased use of
[Vagrant](http://www.vagrantup.com) to streamline our development
systems.  Vagrant makes it easy to manage and provision virtual
machines for each project we work on.  When a new developer joins a
project team, there is no more need to spend half a day fiddling to
make the dev stack "just so" that it works for the project.  In this
post, we'd like to show why we made a Vagrant plugin for proxying
your local HTTP server to your vagrant boxes.

![Vagrant is very helpful](/images/vagrant-logo.jpg)

## Accessing a dev machine

While Vagrant makes it easy to access a virtual machine from the host
box (especially with the
[host manager plugin](https://github.com/smdahlen/vagrant-hostmanager/)),
it is a bit more difficult to make it accessible from elsewhere in the
network.

One way to do it is to
[forward a port](https://www.vagrantup.com/docs/networking/forwarded_ports.html)
from the VM's local network interface to a port on your real machine's
physical network interface.  Another way is to use a
[public network](https://www.vagrantup.com/docs/networking/public_network.html).
These approaches are both viable and allow you to quickly provide
access to a visiting client or a colleague, but they have some
limitations.

## Our network is getting crowded

At Code Yellow, we have several projects where a web application is
supported by tablets or other devices.  And to make things even more
complex, sometimes the web application is available on multiple
virtual hosts and uses the host name to control its behaviour.

Unless you're doing load balancing or other tricks, only one IP
address can have a certain host name within the network.  Of course,
several people will be working on the same project and running the
same Vagrant box simultaneously.  This means you'll quickly run into
trouble with assigning IP addresses and host names.  We also want to
avoid assigning a static IP address for every combination of Vagrant
box and employee, as that quickly becomes intractable.

All this means that using the public network method is not very
practical for us.  We could rely on port forwarding, but that still
doesn't solve the virtual hosts problem.  It's easy to (temporarily)
edit `/etc/hosts` on a developer's box to go look at another
developer's work, but explaining how to do that to a visiting client
is a bit trickier, and on a tablet you don't even have access to the
hosts file as far as I'm aware.

## Using a proxy to solve our problems

So we've come up with a reasonably elegant solution to this: instead
of fiddling with the network, we decided that if you want to share
your application to the outside network, you can run NGINX (or, in
theory, any other web server) and *proxy* incoming requests to boxes
depending on the URL.

For example, if you have a dev box for [REX](https://talkrex.com/)
with two test customers, they'll be available under
`customer1.rex.test` and `customer2.rex.test`.  Let's say your IP
address is `192.168.1.123`, then you can use
`http://192.168.1.123/customer1.rex.test` to access that customer's
REX instance.  The proxy will set the `Host` header based on the first
path component, and forward the remaining part of the URL to the web
server running inside the VM.

To help automate this proxy configuration, we've created a Vagrant
plugin called
[vagrant-reverse-proxy](https://github.com/CodeYellowBV/vagrant-reverse-proxy).
It can be installed in the usual way:

```sh
$ vagrant plugin install vagrant-reverse-proxy
```

After that, all you need is to add a few lines to your `Vagrantfile`:

```ruby
if Vagrant.has_plugin?("vagrant-reverse-proxy")
  config.reverse_proxy.enabled = true
  config.reverse_proxy.vhosts = ['client1.rex.test', 'client2.rex.test']
end
```

And you'll need to do a one-time manual configuration of NGINX on your
dev box.  If you're on a Debian system (or derivative) and don't have
any other web sites configured, just put this in
`/etc/nginx/sites-enabled/default`:

```
server {
	listen [::]:80 default ipv6only=off;
	server_name default;
	include "vagrant-proxy-config";
}
```

The Vagrant plugin will write to a file called
`/etc/nginx/vagrant-proxy-config` and issue a `service nginx reload`
command each time you bring up or halt a box.  Only running boxes are
forwarded.

This setup makes it very easy to configure a tablet to contact a
Vagrant VM, simply through your dev box's public IP address or its
host name if you have a local DNS server.
