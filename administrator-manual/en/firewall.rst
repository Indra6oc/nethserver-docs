.. _firewall-section:

=====================
Firewall and gateway
=====================

|product| can act as :index:`firewall` and :index:`gateway` inside the network where is installed.
All traffic between computers on the local network and the Internet passes through the server that decides how to 
route packets and what rules to apply.
 
Main features:

* Advanced network configuration (bridge, bonds, alias, etc)
* Multi WAN support (up to 15)
* Firewall rules management
* Traffic shaping (QoS)
* Port forwarding
* Routing rules to divert traffic on a specific WAN
* Intrusion Prevention System (IPS)


Firewall and gateway modes are enabled only if:

* the `nethserver-firewall-base` package is installed
* at least there is one network interface configured with red role

.. _policy-section:

Policy
======

Each interface is identified with a color indicating its role within the system.
See :ref:`network-section`.

When a packet network passed through a firewall zone, the system evaluates a list of rules to decide whether 
traffic should be blocked or allowed. 
:dfn:`Policies` are the default which rules are applied if the network traffic does not match any existing criteria.

The firewall implements two default policies editable from the page :menuselection:`Firewall rules` -> :guilabel:`Configure`:

* :dfn:`Allowed`: all traffic from green to red is allowed
* :dfn:`Blocked`: all traffic from green to red network is blocked. Specific traffic must be allowed with custom rules.

Firewall :index:`policies` allow inter-zone traffic accordingly to this schema: ::

 GREEN -> BLUE -> ORANGE -> RED

Traffic is allowed from left to right, blocked from right to left.

You can create rules between zones to change default policies from :guilabel:`Firewall rules` page.

.. note::  Traffic from local network to the server on SSH port (default 22) and Server Manager port (default 980) is **always** permitted.

.. _firewall-rules-section:

Rules
=====

:index:`Rules` apply to all traffic passing through the firewall.
When a network packet moves from one zone to another, the system looks among configured rules. 
If the packet match rule, the rule is applied.

.. note:: Rule's order is very important. The system always applies the first rule that matches.

A rule consists of three main parts:

* Action: action to take when the rule applies
* Source: 
* Destination: 
* Service: 


Available actions are:

* :dfn:`ACCEPT`: accept the network traffic
* :dfn:`REJECT`: block the traffic and notify the sender host 
* :dfn:`DROP`: block the traffic, packets are dropped and not notification is sent to the sender host

.. note:: The firewall will not generate rules for blue and orange zones, if at least a red interface is configured.

REJECT vs DROP
--------------

As a general rule, you should use :index:`REJECT` when you want to inform the source host that the port to which it 
is trying to access is closed. 
Usually the rules on the LAN side can use REJECT. 

For connections from the Internet, it is recommended to use :index:`DROP`, in order to minimize the information disclosure to any 
attackers.

Log
---

When a rule matches the ongoing traffic, it's possible to register the event on a log file by checking the option from the web interface.
:index:`Firewall log` is saved in :file:`/var/log/firewall.log` file.

Examples
--------

Below there are some examples of rules. 

Block all DNS traffic from the LAN to the Internet: 

* Action: REJECT 
* Source: green 
* Destination: red 
* Service: DNS (UDP port 53) 

Allow guest's network to access all the services listening on Server1: 

* Action: ACCEPT 
* Source: blue 
* Destination: Server1 
* Service: -

Multi WAN
=========

The term :dfn:`WAN` (Wide Area Network) refers to a public network outside the server, usually connected to the Internet. 
A :dfn:`provider` is the company who actually manage the :index:`WAN` link.

The system supports up to 15 WAN connections. 
If the server has two or more configured red card, it is required to proceed with :index:`provider` configuration from :guilabel:`Multi WAN` page. 

Each provider represents a  WAN connection and is associated with a network adapter. 
Each provider defines a  :dfn:`weight`: higher the :index:`weight`, higher the priority of the network card associated with the provider. 

The system can use WAN connections in two modes (button  :guilabel:`Configure` on page :menuselection:`Multi WAN`): 

* :dfn:`Balance`: all providers are used simultaneously according to their weight 
* :dfn:`Active backup`: providers are used one at a fly from the one with the highest weight. If the provider you are using loses its connection, all traffic will be diverted to the next provider.


Example
-------

Given two configured providers:

* Provider1: network interface eth1, weight 100
* Provider2: network interface eth0, weight 50

If balanced mode is selected, the server will route a double number of connections on Provider1 over Provider2.

If active backup mode is selected, the server will route all connection on Provider1; only if Provider1 become
unavailable connections will be redirected to Provider2.


Port forward
============

The firewall blocks request from public networks to private ones. 
For example, if web server is running inside the LAN, only computers on the local network can access the service on the green zone. 
Any request made by a user outside the local network is blocked. 

To allow any external user access to the web server you must create a :dfn:`port forward`.
A :index:`port forward` is a rule that allows limited access to resources from outside of the LAN. 

When you configure the server, you must choose the listening ports. The traffic from red interfaces will be redirected to selected ports.
In the case of a web server, listening ports are usually port 80 (HTTP) and 443 (HTTPS). 

When you create a port forward, you must specify at least the following parameters: 

* The source port, can be a number or a range in the format XX:YY (eg: 1000:1100 for begin port and end port 1100)
* The destination port, which can be different from the origin port
* The address of the internal host to which the traffic should be redirected

Example
-------

Given the following scenario:

* Internal server with IP 192.168.1.10, named Server1
* Web server listening on port 80 on Server1
* SSH server listening on port 22 on Server1

If you want to make the server web available directly from public networks, you must create a rule like this:

* origin port: 80
* destination port: 80
* host address: 192.168.1.10

All incoming traffic on firewall's red interfaces on port 80, will be redirected to port 80 on Server1.

In case you want to make accessible from outside the SSH server on port 2222, you will have to create a port forward like this:

* origin port: 2222
* destination port: 22
* host address: 192.168.1.10

All incoming traffic on firewall's red interfaces on port 2222, will be redirected to port 22 on Server1.
 

Limiting access
---------------

You can restrict access to port forward only from some IP address or networks using the field :guilabel:`Allow only from`.

This configuration is useful when services should be available on from trusted IP or networks.
Some possible values:

* ``10.2.10.4``: enable port forward for traffic coming from 10.2.10.4 IP
* ``10.2.10.4,10.2.10.5``: enable port forward for traffic coming from 10.2.10.4 and 10.2.10.5 IPs
* ``10.2.10.0/24``: enable port forward only for traffic coming from 10.2.10.0/24 network
* ``!10.2.10.4``: enable port forward  for all IPs except 10.2.10.4
* ``192.168.1.0/24!192.168.1.3,192.168.1.9``: enable port forward for 192.168.1.0/24 network, except for hosts 192.168.1.3 and 192.168.1.9

NAT 1:1
=======

One-to-one NAT is a way to make systems behind a firewall and configured with private IP addresses appear to have public IP addresses.

If you have a bunch of public IP addresses and if you want to associate one of these to a specific network host, :index:`NAT 1:1` is the way.

Example
-------

In our network we have an host called ``example_host`` with IP ``192.168.5.122``. We have also associated a public IP address ``89.95.145.226`` as an alias of ``eth0`` interface (``RED``).

We want to map our internal host (``example_host`` - ``192.168.5.122``) with public IP ``89.95.145.226``.

In the :guilabel:`NAT 1:1` panel, we choose for the IP ``89.95.145.226`` (readonly field) the specific host (``example_host``) from the combobox. We have configured correctly the one-to-one NAT for our host.


Traffic shaping
===============

:index:`Traffic shaping` allows to apply priority rules on network traffic through the firewall. 
In this way it is possible to optimize the transmission, check the latency and tune 
the available bandwidth. 

To enable traffic shaping is necessary to know the amount of available bandwidth in both directions 
and fill in the fields indicating the speed Internet link. Be aware 
that in case of congestion by the provider there is nothing to do in order to improve performance. 

Traffic shaping can be configured inside from the page :menuselection:`Traffic shaping` -> :guilabel:`Interface rules`.

The system provides three levels of priority, high, medium and low: as default all traffic has medium priority.
It is possible to assign high or low priority to certain services based on the port used (eg low traffic peer to peer). 

The system works even without specifying  services to high or low priority, 
because, by default, the interactive traffic is automatically run at high priority 
(which means, for example, it is not necessary to specify ports for VoIP traffic or SSH). 
Even the traffic type PING is guaranteed high priority. 


.. note:: Be sure to specify an accurate estimate of the band on network interfaces.


Firewall objects
================

:index:`Firewall objects` are representations of network components and are useful to simplify the creation 
of rules. 

There are 4 types of objects: 

* Host: representing local and remote computers. Example: web_server, pc_boss 
* Groups of hosts: representing homogeneous groups of computers. Hosts in a host group should always be reachable using the same interface.
  Example: servers, pc_segreteria 
* Zone: representing networks of hosts. Although similar in concept to a group of hosts, you can express networks using CIDR notation 
* Services: a service listening on a host with at least one port and protocol. Example: ssh, https 

When creating rules, you can use the records defined in :ref:`dns-section` and :ref:`dhcp-section` like host objects.
In addition, each network interface with an associated role is automatically listed among the available zones.

